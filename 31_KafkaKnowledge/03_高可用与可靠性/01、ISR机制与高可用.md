# ISR 机制、高可用与数据可靠性

## 一、副本机制基础

### 1.1 Leader 与 Follower

Kafka 为每个 Partition 维护多个副本（Replica），分布在不同 Broker 上：

- **Leader 副本**：每个 Partition 有且仅有一个 Leader，负责处理所有生产者写入和消费者读取请求
- **Follower 副本**：从 Leader 主动拉取（Pull）数据，保持与 Leader 同步；不对外提供读写服务，仅用于数据冗余和故障切换

> Kafka 只让 Leader 处理读写，Follower 不分担读请求，保证了读的线性一致性（读到的都是已提交数据），代价是读写都打到 Leader。

### 1.2 AR、ISR、OSR

| 概念 | 全称 | 含义 |
|------|------|------|
| **AR** | Assigned Replicas | 该 Partition 的所有副本集合（含 Leader + 所有 Follower） |
| **ISR** | In-Sync Replicas | 与 Leader 保持同步的副本集合（AR 的子集），包含 Leader 本身 |
| **OSR** | Out-of-Sync Replicas | 落后于 Leader 的副本集合（AR - ISR） |

**关系**：AR = ISR + OSR

---

## 二、ISR 机制详解

### 2.1 ISR 的维护

Leader 负责动态维护 ISR 列表。一个 Follower 能留在 ISR 中的条件：

- 在 `replica.lag.time.max.ms`（默认 **10 秒**）内向 Leader 发送过 FetchRequest
- 副本的 LEO（Log End Offset）与 Leader 的差距不超过阈值

**缩减 ISR（maybeShrinkIsr）**：当 Follower 长时间未响应或落后太多，Leader 将其移出 ISR，移入 OSR。

**扩展 ISR（maybeExpandIsr）**：Follower 重新追上 Leader 的进度后，Leader 将其重新纳入 ISR。

### 2.2 HW 与 LEO

这两个概念是理解 Kafka 消息可见性和一致性的关键：

| 概念 | 全称 | 含义 |
|------|------|------|
| **LEO** | Log End Offset | 当前副本日志中**下一条待写入消息的位移**（日志末尾 +1） |
| **HW** | High Watermark | **消费者可见的最大消息位移**，代表已在 ISR 中所有副本都完成写入的最高位置 |

**HW 计算公式**：
```
HW = min(Leader LEO, Follower_1 LEO, Follower_2 LEO, ...)
     其中 Follower 仅计算 ISR 中的副本
```

**示意图**：
```
Leader LEO: 100
Follower-1 LEO: 98   (ISR)
Follower-2 LEO: 95   (ISR)
Follower-3 LEO: 80   (OSR，已被移出 ISR)

HW = min(100, 98, 95) = 95
消费者只能消费到 offset 94 的消息
```

**HW 的作用**：
- 消费者只能读取 HW 之前（包含 HW）的消息，保证消费者不会读到尚未在 ISR 中全部副本同步完成的消息
- 当 Leader 宕机，新选出的 Follower 会将日志截断到 HW，保证数据一致性（避免读到 HW 之后未提交的消息）

### 2.3 消息"已提交"的含义

Kafka 中一条消息被认为"已提交"（Committed），需满足：**消息已被成功写入 ISR 中的所有副本**。只有已提交的消息才会更新 HW，才对消费者可见。

这与 `acks` 配置的关系：
- `acks=1`：Leader 写入即认为成功（不等 ISR 确认），消息可能在 HW 更新前丢失
- `acks=all`：等所有 ISR 副本写入，消息才被真正"提交"，HW 才能推进

---

## 三、Controller 与 Leader 选举

### 3.1 Controller 节点

Controller 是 Kafka 集群的"管理大脑"，是集群中被选举出的一个特殊 Broker，负责：
- 监控所有 Broker 的存活状态
- 在 Leader Partition 宕机时触发 Leader 选举
- 管理 Topic 创建/删除/分区变更的元数据

**Controller 选举（ZooKeeper 模式）**：
1. 所有 Broker 启动时竞争在 ZooKeeper `/controller` 创建临时节点
2. 创建成功者成为 Controller
3. 其他 Broker 监听该节点；Controller 宕机时临时节点消失，触发新一轮选举

**Controller 选举（KRaft 模式，3.0+）**：
- 内置 Raft 共识算法，由指定的 Controller 节点集群（通常 3 个）自主选举，无需 ZooKeeper

### 3.2 Partition Leader 选举

当 Broker 宕机，该 Broker 上的 Partition Leader 需要重新选举：

1. **故障检测**：Controller 检测到 Broker 失联（ZooKeeper 临时节点消失 或 KRaft 心跳超时）
2. **触发选举**：Controller 遍历该 Broker 上的所有 Partition
3. **选择新 Leader**：**从 ISR 列表中选取第一个可用副本**作为新 Leader
4. **更新元数据**：将新 Leader 信息写入 ZooKeeper/KRaft，并通知所有 Broker

**选举速度**：秒级完成，客户端在此期间会有短暂的写入失败（重试后恢复）。

### 3.3 Unclean Leader 选举

**定义**：当 ISR 中所有副本都不可用时，是否允许从 OSR（落后的 Follower）中选举 Leader。

```properties
# Broker/Topic 级别配置
unclean.leader.election.enable=false  # 默认 false（推荐）
```

| 配置 | 优点 | 缺点 |
|------|------|------|
| `false`（推荐） | 避免数据丢失，保证一致性 | ISR 全宕时该 Partition 不可用 |
| `true` | 保证可用性，Partition 不会停止服务 | 新 Leader 数据落后，存在数据丢失和不一致风险 |

**选型建议**：金融、交易等核心业务必须设 `false`；监控日志等允许少量丢失的场景可设 `true`。

---

## 四、副本同步流程

```
1. Producer → Leader 写入消息（追加到 CommitLog）
2. Leader LEO 更新
3. Follower 定期向 Leader 发送 FetchRequest（包含自己的 LEO）
4. Leader 返回新消息 + 自身 HW
5. Follower 写入消息，更新自身 LEO
6. Leader 收集所有 ISR 中 Follower 上报的 LEO，计算新 HW
7. 下次 Follower 拉取时，携带 Leader 的 HW，Follower 更新自己的 HW
8. Consumer 从 Leader 读取 HW 之前的消息
```

**关键细节**：Follower 的 HW 更新是延迟的，需要等到下次 FetchRequest 时才从 Leader 的响应中获取最新 HW，这会导致短暂的 HW 不一致窗口。

---

## 五、数据不丢失完整方案

数据不丢失是生产者、Broker、消费者三方协同的结果：

### 5.1 生产者端

```java
// 必要配置
props.put(ProducerConfig.ACKS_CONFIG, "all");              // 等所有 ISR 副本确认
props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE); // 无限重试
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true"); // 幂等，避免重复

// 使用带 Callback 的异步发送
producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        // 业务级补偿：记录失败消息到 DB，人工或定时任务重发
        saveFailedMessage(record);
    }
});
```

### 5.2 Broker 端

```properties
# Topic 级别或 Broker 默认配置
replication.factor=3            # 3 副本，容忍 1 个 Broker 宕机
min.insync.replicas=2           # ISR 至少 2 个才允许写入
unclean.leader.election.enable=false  # 禁止从 OSR 选主

# 关键参数组合：replication.factor=3 + min.insync.replicas=2 + acks=all
# 效果：容忍 1 个 Broker 宕机，写入仍成功；2 个同时宕机写入拒绝但不丢数据
```

### 5.3 消费者端

```java
// 禁用自动提交，业务处理成功后再手动提交
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        // 消息处理（业务逻辑需要幂等，应对 At Least Once）
        processWithIdempotency(record);
    }
    // 所有消息处理完后再提交
    consumer.commitSync();
}
```

### 5.4 三端总结

| 环节 | 核心配置 | 保证 |
|------|---------|------|
| 生产者 | `acks=all` + `retries=MAX` + 幂等 | 消息至少被 ISR 所有副本写入 |
| Broker | `replication.factor=3` + `min.insync.replicas=2` + `unclean.leader.election.enable=false` | 数据多副本冗余，Leader 切换无数据丢失 |
| 消费者 | `enable.auto.commit=false` + 业务成功后 `commitSync()` + 幂等处理 | 消息处理成功才前移位点，重启从上次断点继续 |

---

## 六、数据不重复方案

| 层级 | 方案 | 原理 |
|------|------|------|
| **生产者** | `enable.idempotence=true` | PID + Sequence Number，Broker 端去重 |
| **跨分区事务** | `transactional.id` + `isolation.level=read_committed` | 事务原子性，消费者只读已提交消息 |
| **业务侧幂等** | 唯一 ID + Redis SETNX 或 DB 唯一约束 | 应用层保证，覆盖所有重复场景 |

---

## 七、常见问题与排查

### 7.1 消息堆积

```bash
# 查看消费者组的 Lag（堆积量）
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group your-consumer-group --describe

# 输出示例：
# TOPIC     PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID
# OrderTopic    0          1000            2000          1000  consumer-1
```

**处理策略**：
1. 扩容消费者实例（至 Partition 数量）
2. 增加 Partition 数量（同时扩消费者）
3. 提高消费者批量处理能力（`max.poll.records` 调大）

### 7.2 消费者组 Rebalance 频繁

```bash
# 查看消费者组状态
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group your-consumer-group --describe
# 状态显示 PreparingRebalance 说明正在重平衡
```

**原因排查**：
- `max.poll.interval.ms` 太小 → 消费处理超时被踢出
- `session.timeout.ms` 太小 → 心跳超时被认为宕机
- 消费者频繁重启 → 使用 `group.instance.id` 静态成员规避

### 7.3 Leader 分布不均

```bash
# 触发分区优先副本选举（将 Leader 恢复到 Preferred Replica）
kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type PREFERRED --all-topic-partitions
```

---

## 八、相关知识

- [[01_核心架构/01、Kafka架构与存储原理]] — Topic/Partition/Offset、Consumer Group
- [[02_生产者与消费者/01、生产消费与高吞吐原理]] — acks、幂等性、事务、手动提交 Offset
- [[04_生态与实战/01、Kafka Streams与选型]] — Kafka vs RabbitMQ vs RocketMQ 选型
