# 四、Watcher 监听机制

> 双链：[[37_ZookeeperKnowledge/03_Watcher与通知机制]]

---

###### 1. 说说 Zookeeper Watcher 机制

Watcher 是 ZooKeeper 的**事件监听通知机制**，是实现分布式协调的核心能力之一。

**核心思想：发布-订阅模式**
- 客户端注册 Watcher 到某个 ZNode
- 当该 ZNode 发生指定事件时，ZooKeeper 服务端主动推送通知给客户端
- 客户端收到通知后异步执行回调逻辑

**Watcher 的三个特性：**
1. **一次性（One-time trigger）**：Watcher 触发后自动失效，如果要持续监听需要重新注册
2. **轻量级**：Watcher 通知不包含具体变化内容，只告诉你"发生了什么类型的事件"，具体数据要自己去 `getData()` 拉取
3. **异步通知**：服务端发送通知给客户端是异步的，但 ZooKeeper 保证同一个客户端的通知是**顺序投递**的

**Watcher 的整体流程：**
```
客户端 getData("/app", watcher)  ──注册Watcher──→  Server
                                                      ↓ 节点数据变化
客户端 Watcher.process(event)   ←─推送通知────  Server
     ↓
重新注册 Watcher（如需持续监听）
```

---

###### 2. 客户端是如何注册 Watcher 实现

客户端注册 Watcher 是"搭便车"的方式——在发起读请求时顺便注册。

**注册入口（ZooKeeper 原生 API）：**

```java
// 方式1：getData 时注册
Stat stat = new Stat();
byte[] data = zk.getData("/app/config", new Watcher() {
    @Override
    public void process(WatchedEvent event) {
        System.out.println("事件类型：" + event.getType());
        System.out.println("节点路径：" + event.getPath());
        // 重新注册（一次性Watcher必须在这里重新注册）
        zk.getData("/app/config", this, null);
    }
}, stat);

// 方式2：exists 时注册（即使节点不存在也能监听节点创建）
Stat exists = zk.exists("/app/lock", watcher);

// 方式3：getChildren 时注册（监听子节点变化）
List<String> children = zk.getChildren("/app", watcher);
```

**底层实现：**
1. 客户端在请求中设置 `watch = true` 标志
2. ZooKeeper 客户端库（ZKWatchManager）本地维护一个 Watcher 注册表：path → Set<Watcher>
3. 服务端也维护一个 WatchTable：path → Set<ServerCnxn>（连接句柄），建立 path 到客户端连接的映射

---

###### 3. 服务端是如何处理 Watcher 实现

**服务端的 Watcher 存储：**

服务端（DataTree）维护两张表：
- `watchTable`：`path → Set<Watcher>`，某路径上注册的所有 Watcher
- `watch2Paths`：`Watcher → Set<path>`，某客户端连接注册的所有路径（用于客户端断开时清理）

**服务端触发流程：**
1. 有写操作修改了某个 ZNode（setData / create / delete）
2. DataTree 在应用事务时调用 `WatchManager.triggerWatch(path, eventType)`
3. WatchManager 从 watchTable 中取出该 path 的所有 Watcher，并**从表中移除**（一次性语义）
4. 向每个 Watcher 对应的客户端连接发送 `WatchedEvent` 通知消息

**通知消息的内容（很精简）：**
```java
WatchedEvent {
    KeeperState keeperState;  // 连接状态
    EventType   eventType;    // 事件类型（NodeCreated/NodeDeleted/NodeDataChanged/NodeChildrenChanged）
    String      path;         // 发生事件的路径
    // 注意：没有具体数据！客户端要自己去拉
}
```

---

###### 4. 客户端是如何回调 Watcher

**客户端接收通知并回调的流程：**

1. 客户端有一个专门的 `EventThread`（事件线程），与 `SendThread`（通信线程）分离
2. `SendThread` 收到服务端发来的 Notification 消息，将其包装成 `WatcherSetEventPair` 放入 `waitingEvents` 队列
3. `EventThread` 持续从队列中取出事件，找到对应的 Watcher，调用 `watcher.process(event)`
4. 回调在 `EventThread` 中执行，因此 **Watcher 回调不能执行耗时操作**（会阻塞后续事件处理）

**注意事项：**
- Watcher 回调是**串行执行**的，同一个客户端的多个 Watcher 按事件发生顺序依次回调
- 回调中**禁止阻塞操作**，如果需要异步处理，在回调里提交到业务线程池
- Watcher 是**一次性**的，回调完成后自动失效，如需持续监听，在回调中重新注册

```java
// 典型的持续监听写法
public class ConfigWatcher implements Watcher {
    private final ZooKeeper zk;
    private final String path;

    @Override
    public void process(WatchedEvent event) {
        if (event.getType() == EventType.NodeDataChanged) {
            try {
                byte[] data = zk.getData(path, this, null);  // 重新注册自己
                onConfigChanged(new String(data));
            } catch (Exception e) {
                log.error("重新注册 Watcher 失败", e);
            }
        }
    }
}
```

---

###### 5. Zookeeper 对节点的 watch 监听通知是永久的吗?

**不是永久的，默认是一次性的。**

标准 Watcher 触发一次后自动失效，如果要持续监听，必须在每次回调中重新注册。

**一次性的原因：**
- 降低服务端 Watcher 表的维护压力（不需要追踪哪些客户端还存活）
- 避免事件积压：如果某个 ZNode 频繁变化，持久 Watcher 会产生大量通知，客户端可能来不及处理

**持久 Watcher（3.6.0+ 新特性）：**

ZooKeeper 3.6.0 引入了 `addWatch()` API，支持两种持久监听模式：
```java
// 持久 Watcher（触发后不自动失效）
zk.addWatch("/app/config", watcher, AddWatchMode.PERSISTENT);

// 持久递归 Watcher（监听路径及所有子路径）
zk.addWatch("/app", watcher, AddWatchMode.PERSISTENT_RECURSIVE);
```

持久递归 Watcher 只监听 `NodeCreated`、`NodeDeleted`、`NodeDataChanged` 三种事件，不监听 `NodeChildrenChanged`（因为递归模式下子节点的创建/删除已经能感知到了）。

---

###### 6. 说一下 Zookeeper 的通知机制?

ZooKeeper 的通知机制本质上是**服务端推送（Push）而非客户端轮询（Pull）**，这是它高效实现分布式协调的关键。

**通知机制的完整链路：**

```
1. 客户端发送读请求（getData/exists/getChildren）并设置 watch=true
2. 服务端处理读请求，将 Watcher 注册到 WatchManager（以 path 为 key）
3. 其他客户端修改了该 path
4. 服务端在应用事务时，查找 WatchManager，找到该 path 的所有 Watcher
5. 服务端向每个 Watcher 对应的客户端连接发送 Notification（事件类型+路径）
6. 客户端 EventThread 收到 Notification，调用 Watcher.process()
7. 客户端重新注册 Watcher（一次性机制）
```

**通知机制的特点：**
- **只通知变化，不传数据**：减少网络传输量
- **顺序保证**：同一客户端的通知按顺序投递
- **尽力投递**：网络故障可能导致通知丢失（客户端重连后会感知到，但中间的事件可能错过）
- **异步**：通知的发送不阻塞写操作的完成

**实际开发中的注意点：**
> Watcher 只保证"曾经发生变化"，不保证"收到所有变化"。在网络抖动或客户端重连的情况下，可能错过中间某些变化。对于关键数据，建议在收到 Watcher 通知后主动 `getData()` 拉取最新状态，而不是依赖通知内容推断状态。

---

###### 7. Watcher 的特性有哪些?

ZooKeeper Watcher 的核心特性：

1. **一次性（One-time）**：Watcher 触发后自动移除，避免重复通知。缺点是需要手动重新注册，存在"监听间隙"（重新注册前的变化会错过）

2. **轻量级（Lightweight）**：Watcher 通知只包含事件类型和路径，不包含节点数据。减少网络传输，但客户端需要再次请求获取最新数据

3. **异步通知（Asynchronous）**：服务端异步推送通知，不阻塞写操作

4. **顺序投递（Sequential Delivery）**：同一客户端的事件通知按顺序投递，保证 A 在 B 之前变化，客户端也先收到 A 的通知

5. **可靠性（Reliability）**：客户端重连后，ZooKeeper 会检查当前状态，确保客户端能感知到重连期间的变化（但不保证每次变化都有单独通知）

6. **客户端本地维护**：Watcher 存储在客户端的 ZKWatchManager 中，按路径分类管理，便于清理

---

###### 8. Watcher 的触发类型有哪些?

根据触发事件（EventType）和注册方式的组合：

| 触发事件 | 触发 getData Watcher | 触发 exists Watcher | 触发 getChildren Watcher |
|---------|---------------------|---------------------|------------------------|
| 节点被创建 | - | ✅ | - |
| 节点被删除 | ✅ | ✅ | ✅ |
| 节点数据变化 | ✅ | ✅ | - |
| 子节点变化（增删） | - | - | ✅ |

**EventType 枚举值：**
- `None`：连接状态变化（如断开、重连）
- `NodeCreated`：节点被创建
- `NodeDeleted`：节点被删除
- `NodeDataChanged`：节点数据被修改（版本号变化）
- `NodeChildrenChanged`：子节点列表发生变化（子节点增加或删除）

**KeeperState（连接状态，配合 None 事件）：**
- `SyncConnected`：正常连接
- `Disconnected`：断开连接
- `Expired`：会话超时
- `AuthFailed`：认证失败

---

###### 9. Watcher 的注册和触发流程是怎样的?

**完整流程（端到端）：**

**注册阶段：**
```
①  客户端调用 getData("/app", watcher, stat)
②  ZooKeeper 客户端库在请求中设置 watch=true 标志
③  请求发送到服务端（通过 TCP 连接）
④  服务端处理读请求，返回数据
⑤  服务端将 <path, 客户端连接> 注册到 WatchManager
⑥  客户端库将 <path, watcher> 注册到本地 ZKWatchManager
```

**触发阶段：**
```
①  某个客户端执行 setData("/app", newData)
②  Leader 生成 Proposal，完成 ZAB 两阶段提交
③  DataTree 应用事务，触发 WatchManager.triggerWatch("/app", NodeDataChanged)
④  WatchManager 取出 "/app" 对应的所有客户端连接，清空该条目
⑤  向每个连接发送 WatchedEvent{NodeDataChanged, "/app"}
⑥  客户端 SendThread 收到通知，交给 EventThread 队列
⑦  EventThread 从本地 ZKWatchManager 取出 "/app" 的 Watcher 并移除
⑧  调用 watcher.process(event)，执行业务逻辑
```

**关键点：**
- 服务端移除 Watcher（步骤④）和客户端移除 Watcher（步骤⑦）都发生在通知发送/接收时，确保不重复触发
- 如果需要持续监听，在步骤⑧的业务逻辑中调用 `getData("/app", this, stat)` 重新注册

---

> **追问：如果在 Watcher 回调中重新注册 Watcher，期间节点再次变化，会丢失通知吗？**
>
> **会的，这是 ZooKeeper 一次性 Watcher 的已知缺陷，叫做"监听间隙"（Watcher Gap）。**
>
> 场景：
> ```
> 1. 客户端注册 Watcher 到 /app（version=1）
> 2. /app 更新为 version=2 → 服务端发通知，客户端移除 Watcher
> 3. 客户端收到通知，回调开始，准备重新注册
> 4. 在重新注册成功之前，/app 更新为 version=3
> 5. 重新注册成功（但只能读到 version=3 的值，version=2→3 的变化被"错过"）
> ```
>
> **解决方案：**
> 1. 使用 ZooKeeper 3.6.0+ 的 `addWatch(PERSISTENT)` 持久 Watcher，彻底解决间隙问题
> 2. 业务层做版本号比较：收到通知后主动 `getData()` 获取当前版本，与本地缓存的版本对比，判断是否需要处理
> 3. 使用 Curator 的 NodeCache / PathChildrenCache，内部封装了重试注册逻辑，用起来更安全
