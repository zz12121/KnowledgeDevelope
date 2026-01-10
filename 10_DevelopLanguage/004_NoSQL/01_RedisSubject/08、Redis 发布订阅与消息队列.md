###### 1. Redis 的发布订阅模式是什么?
Redis发布订阅（Pub/Sub）是一种**消息通信模式**，包含三个核心角色：发布者（Publisher）、频道（Channel）和订阅者（Subscriber）。发布者向频道发送消息，订阅者订阅一个或多个频道，接收这些频道的所有消息。
**核心数据结构与源码实现**：
- **频道订阅关系**：存储在`redisServer.pubsub_channels`字典中，键为频道名，值为订阅该频道的客户端链表。
- **模式订阅关系**：存储在`redisServer.pubsub_patterns`链表中，节点包含客户端和订阅的模式（如`news.*`）。
**工作流程**：
1. **订阅**：客户端执行`SUBSCRIBE channel`或`PSUBSCRIBE pattern`，服务器将客户端添加到相应数据结构中。
2. **发布**：客户端执行`PUBLISH channel message`，服务器：
    - 在`pubsub_channels`中查找频道对应的客户端链表，向所有客户端发送消息。
    - 遍历`pubsub_patterns`，向匹配频道的模式的客户端发送消息。
3. **退订**：使用`UNSUBSCRIBE`或`PUNSUBSCRIBE`从数据结构中移除客户端。
**示例模型**：
```
发布者 → 频道 → 订阅者1
         ↳ 订阅者2
         ↳ 订阅者3
```
###### 2. Redis 发布订阅的缺点是什么?
Redis Pub/Sub的主要缺点包括：
1. **消息无持久化**：消息发送后若没有订阅者在线，消息会丢失。数据仅存储在内存，服务器重启后消息不复存在。
2. **无消息确认机制**：发布者无法知晓订阅者是否成功接收和处理消息。若订阅者处理消息失败，消息无法重试。
3. **无法回溯历史消息**：订阅者只能接收订阅后发布的新消息，无法获取订阅前的历史消息。
4. **客户端缓冲区溢出**：若订阅者消费速度慢，消息可能在客户端缓冲区堆积。当缓冲区超过限制（默认客户端输出缓冲区限制为8MB），Redis服务器可能会强制断开客户端连接。
5. **广播模式**：消息会被发送给所有订阅者，无法进行负载均衡。每个订阅者都会收到全量消息，不能通过增加订阅者数量来分摊处理压力。
这些缺点使得Redis Pub/Sub不适合对**消息可靠性**要求高的场景（如金融交易、订单处理）。
###### 3. Redis 如何实现消息队列?
Redis可通过多种方式实现消息队列，各有适用场景：
**1. 基于List的简单队列**
使用`LPUSH`（生产者）和`BRPOP`（消费者，阻塞版）命令：
```java
// 生产者
jedis.lpush("my_queue", "message1");

// 消费者
while (true) {
    // 阻塞等待，超时时间设为0表示无限等待
    List<String> messages = jedis.brpop(0, "my_queue");
    String message = messages.get(1);
    processMessage(message);
}
```
**特点**：简单，支持阻塞获取，但需自行处理消息确认和重试。
**2. 基于Pub/Sub的发布订阅**
适用于广播场景，但存在上述缺点（如消息丢失）。
**3. 基于Sorted Set的延迟队列**
将消息作为成员，延迟时间戳作为分数（score）。消费者轮询获取当前时间之前的数据：
```java
// 生产者：延迟5秒执行
jedis.zadd("delay_queue", System.currentTimeMillis() + 5000, "message1");

// 消费者
Set<String> messages = jedis.zrangeByScore("delay_queue", 0, System.currentTimeMillis(), 0, 1);
if (!messages.isEmpty()) {
    String message = messages.iterator().next();
    // 处理消息，并从集合中移除
    if (jedis.zrem("delay_queue", message) > 0) {
        processMessage(message);
    }
}
```
**特点**：可实现延迟消息，但轮询效率较低。
**4. 基于Stream的可靠队列（Redis 5.0+）**
```java
// 生产者
jedis.xadd("my_stream", StreamEntryID.NEW_ENTRY, ImmutableMap.of("field", "value"));

// 消费者（消费者组）
String groupName = "consumer_group";
try {
    jedis.xgroupCreate("my_stream", groupName, StreamEntryID.LAST_ENTRY, true);
} catch (Exception e) { /* 组可能已存在 */ }

while (true) {
    List<Map.Entry<String, List<StreamEntry>>> responses = jedis.xreadGroup(
        groupName, "consumer1", 1, 200, false, ImmutableMap.of("my_stream", StreamEntryID.UNRECEIVED_ENTRY)
    );
    for (StreamEntry entry : responses.get(0).getValue()) {
        processMessage(entry.getFields());
        jedis.xack("my_stream", groupName, entry.getID()); // 消息确认
    }
}
```
**特点**：支持消息持久化、消费者组和消息确认，是更成熟的消息队列方案。
###### 4. List 实现消息队列有什么问题?
尽管List简单易用，但在生产环境中存在以下问题：
1. **消息丢失风险**：`BRPOP`取出消息后，若消费者在处理消息前崩溃，消息将永久丢失。List本身不提供消息确认（ACK）机制。
2. **消息重复消费**：通过`RPOPLPUSH`将消息转移到备份列表可实现简易ACK。但若消费者在处理消息后、确认前崩溃，消息仍可能被重复消费。需引入外部状态跟踪（如Redis Hash）记录消息状态，增加了复杂性。
3. **不支持多消费者组**：一条消息只能被一个消费者消费（`BRPOP`取出即删除），无法实现发布-订阅模式或让多个消费者组独立消费同一消息流。
4. **功能有限**：缺乏延迟队列、优先级队列、消息重试等高级消息队列特性，需在应用层自行实现。
5. **消息堆积处理**：当消息生产速度远大于消费速度时，List长度会持续增长，可能耗尽Redis内存。缺乏类似Stream的**最大长度限制和旧消息淘汰机制**。
因此，List仅适用于**消息量小、允许消息丢失、业务逻辑简单**的场景。
###### 5. Redis Stream 是什么?
Redis Stream是Redis 5.0引入的**持久化、支持消费者组的消息队列数据结构**，旨在解决Pub/Sub和List的局限性。
**核心概念与源码结构**：
- **消息**：每个消息有唯一ID（如`timestamp-sequence`）和字段值对。Stream底层使用**基数树（radix tree）**​ 存储消息，按ID排序，高效支持范围查询。
- **消费者组（Consumer Group）**：允许多个消费者协同消费同一Stream。组内消费者共享进度，每条消息仅被组内一个消费者处理。
- **位置检查点（Last Delivered ID）**：每个组记录最后分发消息的ID，确保崩溃恢复后能从正确位置继续。
- **待处理列表（Pending Entries List, PEL）**：已分发但未ACK的消息列表，用于重试。
**常用命令**：
- `XADD mystream * key1 value1`：添加消息。
- `XREADGROUP GROUP mygroup consumer1 COUNT 1 STREAMS mystream >`：消费者组读取消息。
- `XACK mystream mygroup message-id`：确认消息处理完成。
**优势**：
- **消息持久化**：消息存储在Redis中，可回溯。
- **可靠性**：通过ACK机制确保消息至少被消费一次。
- **负载均衡**：消费者组内多个消费者分摊消息处理。
- **监控性**：提供`XPENDING`等命令监控消息状态。
###### 6. Redis Stream 和 Kafka 的区别?

|**特性**​|**Redis Stream**​|**Apache Kafka**​|
|---|---|---|
|**数据存储**​|基于内存，容量受RAM限制，可通过持久化存盘，但重启后加载到内存。|基于磁盘，数据量可远超内存容量，支持长期存储。|
|**性能**​|读写延迟极低（亚毫秒级），适合实时性要求高的场景。|吞吐量极高，适合大数据量流水线处理，但单条消息延迟较高。|
|**集群模式**​|Redis Cluster提供分片，但跨槽操作可能需重定向。|原生分布式设计，分区（Partition）和副本机制成熟。|
|**消息保留**​|支持基于最大长度或存活时间的旧消息淘汰。|支持更灵活的时间、大小保留策略。|
|**生态系统**​|轻量，客户端支持广，但高级功能（如连接器）较少。|生态丰富，提供Kafka Connect、Streams等组件。|
|**适用场景**​|实时消息系统、事件驱动架构，需低延迟、快速部署。|日志聚合、流处理、大数据管道，需高吞吐、长期存储。|
**选择建议**：
- 需要**低延迟、快速原型开发**或数据量可控时选Redis Stream。
- 需要**高吞吐、长期存储、复杂流处理**时选Kafka。
###### 7. 如何保证消息不丢失?
保证消息不丢失需在**生产者、Redis服务器、消费者**三个环节采取措施：
**1. 生产者端**
- **确认机制**：使用Redis Stream的`XADD`命令，等待服务器返回成功响应。对于List，可使用`LPUSH`并检查返回值（列表长度）。
- **重试机制**：若发送失败（网络问题或Redis宕机），生产者应重试发送，并确保消息幂等性（避免重复处理）。
**2. Redis服务器端**
- **持久化配置**：开启RDB快照和AOF日志，平衡性能与数据安全。AOF可配置为`everysec`，减少数据丢失风险。
- **高可用架构**：使用Redis主从复制（Replication）和哨兵（Sentinel）或集群（Cluster），避免单点故障。
- **内存管理**：监控内存使用，防止因内存不足导致数据被淘汰或写入失败。
**3. 消费者端**
- **消息确认**：使用Stream的消费者组，处理完消息后发送`XACK`。对于List，可通过`RPOPLPUSH`将消息移至处理中列表，处理完成后再删除。
- **崩溃恢复**：消费者记录处理进度（如最后成功处理的消息ID）。重启后从该位置继续消费，避免消息遗漏。
- **幂等处理**：因网络重传或消费者重试可能导致消息重复，消费逻辑应设计为幂等（多次处理同一消息结果一致）。
**综合方案示例（Stream）**：
```java
// 生产者
StreamEntryID id = jedis.xadd("mystream", StreamEntryID.NEW_ENTRY, messageFields);
if (id == null) {
    // 重试逻辑
}

// 消费者（消费者组）
List<Map.Entry<String, List<StreamEntry>>> responses = jedis.xreadGroup(
    "group1", "consumer1", 1, 200, false, ImmutableMap.of("mystream", StreamEntryID.UNRECEIVED_ENTRY)
);
for (StreamEntry entry : responses.get(0).getValue()) {
    try {
        processMessage(entry.getFields());
        jedis.xack("mystream", "group1", entry.getID()); // 确认
    } catch (Exception e) {
        // 记录日志，消息将保留在PEL中，供后续重试
    }
}
```
###### 8. 如何实现消息的延迟队列?
延迟队列指消息在指定延迟时间后才被消费者处理的队列。Redis可通过以下方式实现：
**1. 基于Sorted Set的实现**
将消息内容作为成员，延迟执行的时间戳（当前时间+延迟时间）作为分数（score）。
```java
// 生产者：延迟10秒执行
long delayTime = System.currentTimeMillis() + 10000;
jedis.zadd("delay_queue", delayTime, "message_content");

// 消费者线程
while (!Thread.interrupted()) {
    // 获取当前时间之前的所有消息（已到执行时间）
    Set<String> messages = jedis.zrangeByScore("delay_queue", 0, System.currentTimeMillis(), 0, 1);
    if (messages.isEmpty()) {
        Thread.sleep(500); // 避免频繁轮询
        continue;
    }
    String message = messages.iterator().next();
    // 移除消息。使用事务确保原子性，避免多客户端重复消费
    Transaction tx = jedis.multi();
    tx.zrem("delay_queue", message);
    List<Object> results = tx.exec();
    if (results != null && !results.isEmpty()) {
        // 成功移除，处理消息
        processMessage(message);
    }
}
```
**缺点**：轮询方式有延迟，且ZRANGE+ZREM非原子操作需用事务或Lua脚本保证一致性。
**2. 基于Stream的精确延迟（Redis 5.0+）**
结合Stream和Sorted Set，利用Stream的**按ID读取**特性：
- 生产者将消息ID设置为未来时间戳（如`1700000000000-0`），直接添加到Stream。
- 消费者使用`XREAD`阻塞读取，指定最小ID为当前时间戳。Redis会阻塞直到指定时间戳的消息到达。
**3. 基于Redis模块（如Redisson）**
第三方库提供了更高级的延迟队列API。例如Redisson的`RDelayedQueue`：
```java
RBlockingQueue<String> queue = redisson.getBlockingQueue("my_queue");
RDelayedQueue<String> delayedQueue = redisson.getDelayedQueue(queue);
// 延迟10秒发送
delayedQueue.offer("message", 10, TimeUnit.SECONDS);
```
**方案选择**：
- **轻量级、延迟精度要求不高**：用Sorted Set轮询。
- **精确延迟、高可靠性**：用Stream特性或第三方库。