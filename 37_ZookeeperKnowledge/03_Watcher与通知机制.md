# 03 Watcher 与通知机制

> 关联面试题：[[10_DevelopLanguage/012_Zookeeper/01_ZookeeperSubject/04、Watcher 监听机制]]

---

## 一、Watcher 核心特性

| 特性 | 说明 |
|------|------|
| **一次性** | 触发后自动失效，需手动重新注册 |
| **轻量级** | 通知只含事件类型+路径，不含具体数据 |
| **异步** | 服务端异步推送，不阻塞写操作 |
| **顺序投递** | 同一客户端的通知按事件顺序投递 |

**3.6.0+ 新增持久 Watcher（`addWatch` API）：**
- `PERSISTENT`：持久，触发后不失效
- `PERSISTENT_RECURSIVE`：持久且递归监听所有子路径

---

## 二、Watcher 注册入口

| API | 监听内容 |
|-----|---------|
| `getData(path, watcher, stat)` | 节点数据变化、节点删除 |
| `exists(path, watcher)` | 节点创建、数据变化、节点删除 |
| `getChildren(path, watcher)` | 子节点列表变化（增删） |

---

## 三、触发事件类型矩阵

| 触发事件 | getData Watcher | exists Watcher | getChildren Watcher |
|---------|:--------------:|:--------------:|:-------------------:|
| 节点创建 | — | ✅ | — |
| 节点删除 | ✅ | ✅ | ✅ |
| 节点数据变化 | ✅ | ✅ | — |
| 子节点变化（增删） | — | — | ✅ |

**EventType 枚举：**
- `None`：连接状态变化
- `NodeCreated` / `NodeDeleted`
- `NodeDataChanged`
- `NodeChildrenChanged`

---

## 四、Watcher 端到端流程

```
【注册阶段】
客户端：getData("/app", watcher)
  → 请求中设置 watch=true
  → 服务端：WatchManager 注册 path→ServerCnxn
  → 客户端本地：ZKWatchManager 注册 path→Watcher

【触发阶段】
其他客户端修改 /app
  → ZK 应用事务时触发 WatchManager.triggerWatch()
  → 取出该 path 的所有 Watcher，从表中移除（一次性）
  → 向对应客户端连接发送 WatchedEvent{type, path}

【回调阶段】
客户端 EventThread 收到通知
  → 从本地 ZKWatchManager 取出并移除 Watcher
  → 调用 watcher.process(event)
  → 业务逻辑 + 重新注册 Watcher
```

---

## 五、Watcher 回调注意事项

```java
// 正确的持续监听写法
public class ConfigWatcher implements Watcher {
    @Override
    public void process(WatchedEvent event) {
        if (event.getType() == EventType.NodeDataChanged) {
            try {
                // ① 立刻重新注册（减少监听间隙）
                byte[] data = zk.getData(event.getPath(), this, null);
                // ② 处理业务（不要在回调里做耗时操作）
                executor.submit(() -> processConfig(data));
            } catch (Exception e) {
                log.error("重新注册Watcher失败", e);
            }
        }
    }
}
```

**⚠️ 重要限制：**
- Watcher 回调在 `EventThread` 中串行执行
- 回调中禁止阻塞操作（Thread.sleep、重量级 IO），否则阻塞后续所有事件
- 耗时业务逻辑必须提交到业务线程池异步处理

---

## 六、Watcher 间隙问题

**问题：** 在"收到触发通知"到"重新注册成功"的窗口期内，如果节点再次变化，这次变化会被错过。

```
版本1变化 → 收到通知 → 回调中重新注册 → 版本2变化（已错过！）
```

**解决方案：**

**方案1：Curator NodeCache（单节点监听）**
```java
NodeCache cache = new NodeCache(client, "/app/config");
cache.getListenable().addListener(() -> {
    // 内部自动处理重注册和间隙问题
    ChildData data = cache.getCurrentData();
    onConfigChanged(new String(data.getData()));
});
cache.start(true);  // true：立即加载当前值
```

**方案2：Curator PathChildrenCache（子节点监听）**
```java
PathChildrenCache cache = new PathChildrenCache(client, "/services/order", true);
cache.getListenable().addListener((c, event) -> {
    switch (event.getType()) {
        case CHILD_ADDED:
        case CHILD_REMOVED:
        case CHILD_UPDATED:
            refreshServiceList(cache.getCurrentData());
    }
});
cache.start(StartMode.BUILD_INITIAL_CACHE);
```

**方案3：Curator TreeCache（递归监听整棵子树）**
```java
TreeCache cache = new TreeCache(client, "/app");
cache.getListenable().addListener((c, event) -> {
    System.out.println("变化：" + event.getType() + " " + event.getData().getPath());
});
cache.start();
```

**方案4：ZooKeeper 3.6+ 持久 Watcher**
```java
zk.addWatch("/app/config", watcher, AddWatchMode.PERSISTENT);
// 触发后不失效，无间隙问题
```

---

## 七、连接状态事件

除了节点事件，Watcher 还会收到连接状态变化通知（`EventType.None`）：

| KeeperState | 含义 | 建议操作 |
|-------------|------|---------|
| `SyncConnected` | 正常连接 | 正常工作 |
| `Disconnected` | 连接断开（可能重连） | 暂停业务，等待重连 |
| `Expired` | Session 超时 | 重建所有 ZK 资源 |
| `AuthFailed` | 认证失败 | 检查认证配置 |
| `ConnectedReadOnly` | 只读模式（少数派节点） | 只读，停止写操作 |
