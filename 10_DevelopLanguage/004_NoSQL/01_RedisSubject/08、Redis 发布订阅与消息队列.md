###### 1. Redis 的发布订阅模式是什么?
Redis 发布订阅（Pub/Sub）是一种**消息通信模式**，包含三个角色：发布者（Publisher）、频道（Channel）和订阅者（Subscriber）。发布者向频道发消息，订阅了该频道的所有订阅者都能收到。

**核心数据结构**：
- `redisServer.pubsub_channels`：字典，key 为频道名，value 为订阅该频道的客户端链表
- `redisServer.pubsub_patterns`：链表，存储模式订阅关系（如 `news.*`）

**工作流程**：
1. 订阅者执行 `SUBSCRIBE channel`，服务器把该客户端加入频道的订阅列表
2. 发布者执行 `PUBLISH channel message`，服务器找到频道对应的所有订阅者，逐个发送消息；同时遍历 `pubsub_patterns`，向匹配模式的订阅者也发送
3. 订阅者用 `UNSUBSCRIBE` 退订

**模式订阅**：`PSUBSCRIBE news.*` 可以订阅所有以 `news.` 开头的频道，支持 glob 风格匹配。

> 📖 [[../../../23_RedisKnowledge/08_消息队列/01、消息队列与发布订阅#1-pubsub-原理|知识库：Pub/Sub 原理]]

###### 2. Redis 发布订阅的缺点是什么?
Pub/Sub 很简单，但缺点也很明显，适用场景有限。

**消息无持久化**：消息发出去那一刻没有订阅者在线，消息直接丢失。Redis 重启后消息也消失，不可恢复。

**无消息确认（ACK）机制**：发布者不知道订阅者有没有成功接收和处理，消息处理失败了没有办法重试，数据可靠性无法保证。

**无法回溯历史**：订阅者只能收到订阅之后发布的消息，没法消费历史消息，对于需要追历史的场景完全无法满足。

**客户端缓冲区溢出风险**：订阅者消费速度慢，消息在客户端输出缓冲区堆积，超过限制（默认 8MB）后 Redis 会直接断开这个订阅者的连接，消息丢失。

**广播模式，无法负载均衡**：每个订阅者都收到全量消息，不能通过增加消费者来分摊处理压力，这是 Pub/Sub 和消息队列最本质的区别。

总结：Pub/Sub 适合对消息可靠性要求不高的通知类场景（如实时通知、聊天室），不适合需要可靠交付的业务逻辑。

> 📖 [[../../../23_RedisKnowledge/08_消息队列/01、消息队列与发布订阅#2-pubsub-局限性|知识库：Pub/Sub 局限性]]

###### 3. Redis 如何实现消息队列?
Redis 提供了四种实现消息队列的方式，各有侧重：

**List 简单队列**：生产者 `LPUSH`，消费者 `BRPOP` 阻塞等待。简单够用，但没有 ACK 机制，消费失败消息会丢。

**Pub/Sub 广播**：适合广播通知，不适合可靠消息传递。

**Sorted Set 延迟队列**：消息内容作为 member，执行时间戳作为 score，消费者轮询 `ZRANGEBYSCORE`。适合延迟任务。

**Redis Stream（推荐）**：Redis 5.0 引入的专门为消息队列设计的数据结构，支持持久化、消费者组、ACK 机制，是功能最完整的方案：

```java
// 生产者
jedis.xadd("orders", StreamEntryID.NEW_ENTRY, Map.of("orderId", "123", "status", "NEW"));

// 消费者组（多个消费者分摊消费）
jedis.xgroupCreate("orders", "order-processors", StreamEntryID.LAST_ENTRY, true);

// 消费者
while (true) {
    var responses = jedis.xreadGroup(
        "order-processors", "worker-1",
        1, 200, false,
        Map.of("orders", StreamEntryID.UNRECEIVED_ENTRY)
    );
    for (var entry : responses.get(0).getValue()) {
        try {
            process(entry.getFields());
            jedis.xack("orders", "order-processors", entry.getID());  // 确认
        } catch (Exception e) {
            // 不 ACK，消息留在 PEL 等待重试
        }
    }
}
```

> 📖 [[../../../23_RedisKnowledge/08_消息队列/01、消息队列与发布订阅#3-四种实现方式对比|知识库：消息队列实现方式]]

###### 4. List 实现消息队列有什么问题?
List 实现消息队列有几个明显的缺陷，让它只适合非常简单的场景：

**消息可能丢失**：`BRPOP` 取出消息后，如果消费者在处理过程中崩溃，消息就永远丢了，List 本身没有 ACK 机制。可以用 `RPOPLPUSH` 把消息先转移到"处理中"列表来模拟 ACK，但实现复杂且不可靠。

**不支持多消费者组**：一条消息被 `BRPOP` 取出后就从 List 里删除了，只有一个消费者能消费。如果需要多个业务模块同时处理同一条消息（比如订单成功后同时触发物流和积分两个服务），List 无法实现，只能发两次或用 Pub/Sub（但 Pub/Sub 又不可靠）。

**消息堆积风险**：生产速度 > 消费速度时，List 会无限增长，可能撑爆 Redis 内存。没有背压机制，也没有最大长度的自动淘汰（当然可以手动配置，但丢消息了）。

**功能缺失**：没有延迟消息、优先级、消息重试、消息追踪等特性，需要在应用层自己实现，成本高。

**结论**：List 只适合消息量小、允许偶发丢失、业务简单的场景，比如内部日志上报、非关键的任务调度。生产上更推荐 Redis Stream 或专业的 MQ（Kafka/RocketMQ）。

> 📖 [[../../../23_RedisKnowledge/08_消息队列/01、消息队列与发布订阅#4-list-消息队列问题|知识库：List 消息队列的局限性]]

###### 5. Redis Stream 是什么?
Redis Stream 是 Redis 5.0 引入的**专为消息队列场景设计的数据结构**，解决了 Pub/Sub 和 List 的主要缺陷。

**核心概念**：
- **消息 ID**：格式为 `timestamp-sequence`（如 `1699999999000-0`），全局唯一，自动生成
- **消费者组（Consumer Group）**：允许多个消费者协同消费，组内每条消息只被一个消费者处理，实现负载均衡
- **Last Delivered ID**：每个消费者组维护的消费进度标记，崩溃恢复后从这里继续
- **PEL（Pending Entries List）**：已分发但未 ACK 的消息列表，超时后可以重新分配给其他消费者

**常用命令**：

```redis
XADD mystream * key1 val1          # 添加消息，* 表示自动生成 ID
XREAD COUNT 10 STREAMS mystream 0  # 读取消息（从 ID=0 开始）
XREADGROUP GROUP g1 c1 COUNT 1 STREAMS mystream >   # 消费者组消费（> 表示未读消息）
XACK mystream g1 1699999999000-0   # 确认消息处理完成
XPENDING mystream g1 - + 10        # 查看 PEL 中待处理消息
```

**Stream 的可靠性保证**：生产者写入后消息持久化到 Redis（加上持久化配置后也落盘）；消费者未 ACK 的消息保留在 PEL，可以用 `XAUTOCLAIM` 在超时后重新分配给其他消费者，实现 at-least-once 语义。

> 📖 [[../../../23_RedisKnowledge/08_消息队列/01、消息队列与发布订阅#5-redis-stream|知识库：Redis Stream 详解]]

###### 6. Redis Stream 和 Kafka 的区别?
两者都能做消息队列，但定位不同，适用场景有差异。

**数据存储**方面，Redis Stream 基于内存，容量受 RAM 限制；Kafka 基于磁盘，可以存储远超内存的数据量，更适合长期大量存储。

**性能**方面，Redis Stream 延迟极低（亚毫秒级），Kafka 单条消息延迟相对高，但批量吞吐量极大，适合海量数据流水线。

**集群与分区**方面，Kafka 原生分布式设计，分区（Partition）和副本机制非常成熟，天然支持水平扩展；Redis Cluster 可以分片，但 Stream 的消费者组不能跨节点协同，扩展性相对受限。

**生态**方面，Kafka 有 Kafka Connect（连接各种数据源）、Kafka Streams（流处理）等丰富组件；Redis Stream 相对轻量，生态较简单。

**适用场景**：Redis Stream 适合**低延迟、实时性要求高、数据量可控**的场景（如实时推送、事件通知）；Kafka 适合**高吞吐、长期存储、复杂流处理**的场景（如日志聚合、大数据管道）。

总结：如果已经有 Redis 且消息量不大，用 Stream 很方便；如果需要高可靠的大规模消息系统，上 Kafka。

> 📖 [[../../../23_RedisKnowledge/08_消息队列/01、消息队列与发布订阅#6-stream-vs-kafka|知识库：Redis Stream vs Kafka]]

###### 7. 如何保证消息不丢失?
从生产者、服务端、消费者三个环节分别做保障。

**生产者端**：使用 `XADD` 写入 Stream，检查返回的消息 ID 是否成功；网络异常时重试发送，消息需要有幂等标识（如订单 ID），避免重复消费造成问题。

**服务端（Redis）**：开启 AOF 持久化（`appendfsync everysec`），保证 Redis 重启后数据不丢失；配置主从复制 + 哨兵或集群，避免单点故障；监控内存使用，防止内存满了触发淘汰策略把消息删掉（`noeviction` 策略不会删 key，但会拒绝写入）。

**消费者端**：消费成功后才发送 `XACK` 确认，失败时不 ACK，消息留在 PEL，Redis 不会删除；重启后查询 PEL 继续处理未确认消息；消费逻辑设计为幂等，即使同一消息被处理多次也不产生副作用。

**综合示例（Stream）**：

```java
// 消费者
while (true) {
    var msgs = jedis.xreadGroup("group1", "consumer1", 1, 200, false,
        Map.of("stream:orders", StreamEntryID.UNRECEIVED_ENTRY));
    for (var entry : msgs.get(0).getValue()) {
        try {
            processOrder(entry.getFields());
            jedis.xack("stream:orders", "group1", entry.getID());  // 成功才 ACK
        } catch (Exception e) {
            log.error("处理失败，消息将保留在 PEL 等待重试", e);
            // 不 ACK，等待超时后重新分配
        }
    }
}
```

> 📖 [[../../../23_RedisKnowledge/08_消息队列/01、消息队列与发布订阅#7-消息可靠性保证|知识库：消息不丢失保障]]

###### 8. 如何实现消息的延迟队列?
延迟队列让消息在指定时间后才被消费，Redis 有两种主流实现方案。

**方案一：Sorted Set（最常用）**

消息内容作为 member，执行时间戳（当前时间 + 延迟毫秒数）作为 score，消费者轮询分数 ≤ 当前时间的消息。

```java
// 生产者：10 秒后执行
jedis.zadd("delay:queue", System.currentTimeMillis() + 10000, messageJson);

// 消费者（每 500ms 轮询一次）
while (true) {
    Set<String> messages = jedis.zrangeByScore(
        "delay:queue", 0, System.currentTimeMillis(), 0, 1
    );
    if (!messages.isEmpty()) {
        String msg = messages.iterator().next();
        // ZREM 成功说明抢到了（防并发重复消费）
        if (jedis.zrem("delay:queue", msg) > 0) {
            process(msg);
        }
    } else {
        Thread.sleep(500);
    }
}
```

**注意**：`ZRANGEBYSCORE` + `ZREM` 不是原子的，并发消费时用 Lua 脚本或 WATCH 防止重复消费。

**方案二：Redisson RDelayedQueue（生产推荐）**

Redisson 封装了底层细节，API 简洁，可靠性高：

```java
RBlockingQueue<String> blockingQueue = redisson.getBlockingQueue("myQueue");
RDelayedQueue<String> delayedQueue = redisson.getDelayedQueue(blockingQueue);

// 发送延迟消息
delayedQueue.offer("hello", 10, TimeUnit.SECONDS);

// 消费（阻塞等待，精确到时就取到）
String msg = blockingQueue.take();
```

> 📖 [[../../../23_RedisKnowledge/08_消息队列/01、消息队列与发布订阅#8-延迟队列|知识库：延迟队列实现]]
