# 七、客户端与 API

> 双链：[[37_ZookeeperKnowledge/05_客户端与分布式应用]]

---

###### 1. Zookeeper 的 java 客户端都有哪些?

目前主流的 ZooKeeper Java 客户端有 **3 种**：

**① ZooKeeper 原生客户端**
- ZooKeeper 官方提供，包含在 `zookeeper` 依赖中
- 功能完整，但 API 较底层，需要处理 Watcher 重注册、重连逻辑等
- 适合对 ZooKeeper 原理理解深入、需要精细控制的场景

**② ZkClient**
- 早期流行的封装库（GitHub: sgroschupf/zkclient）
- 对原生客户端做了基础封装：自动重连、序列化、事件监听
- 目前已**不再维护**，不推荐新项目使用

**③ Apache Curator（推荐）**
- Netflix 开源，Apache 顶级项目
- 对原生客户端的全面封装，提供：
  - 自动重连和 Session 恢复
  - 重试策略（RetryPolicy）
  - 分布式原语（锁、选举、队列、Barrier、计数器）
  - NodeCache / PathChildrenCache / TreeCache（解决 Watcher 间隙问题）
  - 健康检查、连接状态监听

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>5.5.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.5.0</version>
</dependency>
```

生产环境一律用 Curator，不要再用 ZkClient 了。

---

###### 2. Curator 框架的优势是什么?

Curator 相比原生客户端有以下核心优势：

**① 自动重连与 Session 恢复**
- 网络抖动时自动重连，业务层只需监听 `ConnectionState` 变化
- Session 超时时自动重建，Watcher 自动重新注册（NodeCache/PathChildrenCache 封装）

**② 流畅的 Builder API**
```java
// 原生客户端：繁琐
byte[] data = zk.getData("/path", false, stat);
zk.setData("/path", newData, stat.getVersion());

// Curator：链式调用
byte[] data = client.getData().forPath("/path");
client.setData().withVersion(stat.getVersion()).forPath("/path", newData);
```

**③ 丰富的分布式原语（Recipes）**
- `InterProcessMutex`：可重入分布式互斥锁
- `InterProcessReadWriteLock`：读写锁
- `LeaderSelector`：Leader 选举
- `DistributedAtomicLong`：分布式原子计数器
- `DistributedQueue`：分布式队列
- `Barrier` / `DoubleBarrier`：分布式屏障

**④ 可配置重试策略**
```java
// 重试3次，每次间隔指数增长
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
CuratorFramework client = CuratorFrameworkFactory.builder()
    .connectString("zoo1:2181,zoo2:2181,zoo3:2181")
    .retryPolicy(retryPolicy)
    .sessionTimeoutMs(30000)
    .connectionTimeoutMs(5000)
    .build();
```

**⑤ 连接状态事件**
```java
client.getConnectionStateListenable().addListener((c, state) -> {
    if (state == ConnectionState.LOST) {
        // Session 超时，需要重新抢锁等操作
    }
});
```

---

###### 3. Zookeeper 原生客户端的缺点有哪些?

**① Watcher 是一次性的，需要手动重新注册**
- 每次触发后失效，稍不注意就漏掉通知
- 重新注册和事件触发之间有间隙，可能丢失中间变化

**② 没有自动重连逻辑**
- 连接断开后，原生客户端进入 CONNECTING 状态，但需要业务层自己处理后续的重建逻辑
- Session 超时后必须创建新的 ZooKeeper 实例，业务侵入性强

**③ 没有重试机制**
- 操作失败（如 ConnectionLossException）后需要业务层自己判断是否重试、如何重试

**④ 异常处理繁琐**
- 几乎所有方法都抛出受检异常（`KeeperException`、`InterruptedException`），需要大量 try-catch

**⑤ 缺乏高级分布式原语**
- 原生 API 只有基础的 CRUD + Watcher，分布式锁、选举等都需要自己实现，容易踩坑

**⑥ 序列化需要自己处理**
- 原生 API 只支持 `byte[]`，需要自己做对象的序列化/反序列化

---

###### 4. Zookeeper 客户端常用的操作有哪些?

使用 Curator 的常用操作示例：

```java
CuratorFramework client = CuratorFrameworkFactory.builder()
    .connectString("zoo1:2181")
    .retryPolicy(new ExponentialBackoffRetry(1000, 3))
    .build();
client.start();

// 1. 创建节点（持久）
client.create().creatingParentsIfNeeded()
    .withMode(CreateMode.PERSISTENT)
    .forPath("/app/config", "value".getBytes());

// 2. 创建临时节点
client.create().withMode(CreateMode.EPHEMERAL)
    .forPath("/app/lock");

// 3. 创建临时顺序节点
String path = client.create().withMode(CreateMode.EPHEMERAL_SEQUENTIAL)
    .forPath("/app/queue-");
// path = /app/queue-0000000001

// 4. 读取节点数据
byte[] data = client.getData().forPath("/app/config");

// 5. 更新节点数据（带版本 CAS）
Stat stat = new Stat();
client.getData().storingStatIn(stat).forPath("/app/config");
client.setData().withVersion(stat.getVersion())
    .forPath("/app/config", "newValue".getBytes());

// 6. 删除节点（包括子节点）
client.delete().deletingChildrenIfNeeded().forPath("/app");

// 7. 获取子节点列表
List<String> children = client.getChildren().forPath("/app");

// 8. 检查节点是否存在
Stat stat2 = client.checkExists().forPath("/app/config");
boolean exists = stat2 != null;

// 9. NodeCache（持续监听节点数据变化，解决Watcher间隙）
NodeCache cache = new NodeCache(client, "/app/config");
cache.getListenable().addListener(() -> {
    byte[] newData = cache.getCurrentData().getData();
    System.out.println("配置变更：" + new String(newData));
});
cache.start();
```

---

###### 5. 如何使用 Zookeeper 实现分布式锁?

基于 ZooKeeper 的分布式锁利用**临时顺序节点**实现，这是 ZooKeeper 最经典的应用之一。

**原理：**
1. 所有竞争锁的客户端在同一父路径下创建临时顺序节点，如 `/locks/order-lock-`
2. 每个客户端获取父路径下所有子节点，**按序号排序**
3. 如果自己创建的节点是**最小节点**，则获取锁
4. 否则监听**前一个节点**（不是最小节点），等待前一个节点删除后重新判断
5. 持有锁的客户端完成业务后删除自己的节点，或 Session 断开时临时节点自动删除

**为什么监听前一个节点而不是最小节点？**
避免"羊群效应"——如果所有等待者都监听同一个（最小）节点，该节点删除时会同时唤醒所有等待者，产生大量无效竞争。监听前一个节点，每次只唤醒一个等待者，形成有序队列。

**Curator 实现（推荐，不要自己写）：**
```java
InterProcessMutex lock = new InterProcessMutex(client, "/locks/order-lock");

try {
    // 尝试获取锁，等待最多30秒
    if (lock.acquire(30, TimeUnit.SECONDS)) {
        try {
            // 执行业务逻辑（临界区）
            doBusinessLogic();
        } finally {
            lock.release();  // 务必在 finally 中释放
        }
    } else {
        // 获取锁超时
        throw new RuntimeException("获取分布式锁超时");
    }
} catch (Exception e) {
    log.error("分布式锁异常", e);
}
```

**Curator 提供的锁类型：**
- `InterProcessMutex`：可重入互斥锁（最常用）
- `InterProcessReadWriteLock`：读写锁
- `InterProcessSemaphoreMutex`：不可重入互斥锁
- `InterProcessMultiLock`：多节点联合锁

---

###### 6. Zookeeper 如何实现分布式队列?

**方式一：基于顺序节点的 FIFO 队列**

利用临时顺序节点的自增特性：
```java
// 入队：创建临时顺序节点
client.create().withMode(CreateMode.PERSISTENT_SEQUENTIAL)
    .forPath("/queue/item-", data);
// 路径变为 /queue/item-0000000001, item-0000000002 ...

// 出队：获取最小节点
List<String> items = client.getChildren().forPath("/queue");
Collections.sort(items);
String smallest = items.get(0);
// 消费 smallest，然后删除
byte[] data = client.getData().forPath("/queue/" + smallest);
client.delete().forPath("/queue/" + smallest);
```

**方式二：使用 Curator DistributedQueue（已废弃）**

Curator 的 `DistributedQueue` 已在 3.x 版本废弃，官方建议使用 `DistributedDelayQueue` 或自行实现。

**实际建议：**
> ZooKeeper 不是专业消息队列，不适合高吞吐量场景。分布式队列这个需求，建议用 **RocketMQ** 或 **Kafka**，ZooKeeper 的队列实现只适合低频、协调型场景（如任务分发、有序处理）。

---

> **追问：ZooKeeper 分布式锁和 Redis 分布式锁怎么选？**
>
> | 对比维度 | ZooKeeper 分布式锁 | Redis 分布式锁（Redisson） |
> |---------|------------------|--------------------------|
> | 可靠性 | 高（CP模型，锁数据不会不一致） | 中（主从切换时可能锁丢失） |
> | 锁释放保证 | 客户端宕机自动释放（临时节点） | 需设置过期时间，时间设短了会误释放 |
> | 性能 | 低于 Redis（ZK 写需要 ZAB 协议） | 高（Redis 内存操作，性能极好） |
> | 死锁风险 | 无（Session 超时自动删除临时节点） | 有（若未正确设置TTL可能永久持有） |
> | 公平性 | 天然公平（顺序节点有序） | 默认不公平（Redisson支持公平锁） |
> | 适用场景 | 需要强一致性/对锁可靠性要求高 | 高并发/对性能敏感 |
>
> 实际选型：金融类、订单类高一致性场景选 ZooKeeper；秒杀、高频API限流选 Redis。
