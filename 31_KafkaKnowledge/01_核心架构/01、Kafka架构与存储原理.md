# Kafka 架构与存储原理

## 一、Kafka 定位与核心价值

Apache Kafka 是一个**分布式、高吞吐量、可持久化的流数据平台**，诞生于 LinkedIn，解决的是海量用户行为日志的实时采集和数据管道问题。与传统消息队列不同，Kafka 的核心抽象是**分区的、只能追加的提交日志（Commit Log）**，兼顾消息系统、存储系统、流处理平台三种角色。

**三大核心作用**：
1. **发布/订阅消息流**：生产者和消费者解耦，支持多消费者组同时消费同一 Topic
2. **持久化存储流数据**：消息落盘保存（默认 7 天），消费者可以重放历史数据
3. **实时流处理**：通过 Kafka Streams API，无需引入外部处理框架即可完成流计算

---

## 二、核心架构组件

| 组件 | 角色 | 关键特性 |
|------|------|---------|
| **Broker** | 集群服务节点，负责消息存储和服务 | 一个集群由多个 Broker 组成，分散负载 |
| **Producer** | 向 Topic 发布消息的客户端 | 批量发送、压缩、幂等性、事务 |
| **Consumer** | 从 Topic 订阅消息的客户端 | Pull 模式，自主控制消费速率 |
| **Topic** | 消息的逻辑分类 | 类似数据库表名，多分区横向扩展 |
| **Partition** | Topic 的物理分片，水平扩展基石 | 分区内消息有序，分区间无序 |
| **Consumer Group** | 消费者组，负载均衡核心机制 | 同一分区只能被组内一个消费者消费 |
| **ZooKeeper/KRaft** | 集群元数据管理和协调 | 3.0+ 版本 KRaft 模式取代 ZooKeeper |

### 2.1 架构图

```
Producer → Topic（逻辑）
              ↓
    ┌──────────────────┐
    │ Partition 0      │ → Broker A（Leader）  → Broker B（Follower）
    │ Partition 1      │ → Broker B（Leader）  → Broker C（Follower）
    │ Partition 2      │ → Broker C（Leader）  → Broker A（Follower）
    └──────────────────┘
              ↓
     Consumer Group A    Consumer Group B
    (每个分区由1个消费者消费) (每个分区由1个消费者消费)
```

### 2.2 Topic、Partition、Offset 的关系

- **Topic（主题）**：消息的逻辑分类，好比一本书
- **Partition（分区）**：Topic 的物理分片，好比书的章节；每个分区是一个有序、不可变的消息序列
- **Offset（偏移量）**：消息在分区内的唯一序号（单调递增的 long），好比页码

> **一句话理解**：Topic 是书，Partition 是章节，Offset 是页码，Consumer Group 的消费位点就是书签。

---

## 三、存储机制

### 3.1 基于日志的持久化模型

每个 Partition 在物理上是一个**只能追加写入的日志目录**，目录内包含多组 Segment 文件：

```
/data/kafka/topic-name-0/          ← Partition 0 的目录
    ├── 00000000000000000000.log   ← Segment 文件（实际消息数据）
    ├── 00000000000000000000.index ← 稀疏偏移量索引（Offset → 物理位置）
    ├── 00000000000000000000.timeindex ← 时间戳索引
    ├── 00000000000000001234.log   ← 下一个 Segment
    ├── 00000000000000001234.index
    └── ...
```

**顺序 I/O 写入**：消息总是追加到当前活跃 Segment 的末尾，不存在随机写入，磁盘顺序写性能接近内存随机读写。

### 3.2 日志段（Log Segment）

每个 Segment 由一对文件组成：
- `.log`：实际消息数据，顺序写入
- `.index`：**稀疏偏移量索引**，建立 Offset → 物理位置的映射，支持二分查找快速定位

**查找消息的过程**：
1. 根据目标 Offset，用二分查找定位到对应的 Segment
2. 在 Segment 的 `.index` 中二分查找找到最近的索引项
3. 在 `.log` 文件中顺序扫描到目标 Offset

**Segment 滚动条件**（满足任一触发）：
- 大小达到 `segment.bytes`（默认 1GB）
- 时间达到 `segment.ms`（默认 7 天）
- 索引文件满

### 3.3 消息留存策略

| 策略 | 配置 | 描述 |
|------|------|------|
| **基于时间（默认）** | `retention.ms=604800000`（7天） | 超过保留时间的 Segment 被删除 |
| **基于大小** | `retention.bytes=1073741824` | 超过大小限制时删除旧 Segment |
| **日志压缩（Compaction）** | `cleanup.policy=compact` | 每个 Key 只保留最新的值，适合状态存储场景 |

---

## 四、高吞吐量原理

Kafka 能达到数十万甚至百万级 TPS，是多项技术协同的结果：

### 4.1 顺序 I/O

磁盘顺序写性能是随机写的 3-4 个数量级，Kafka 的 Append-Only 日志设计完全利用了这一特性。

### 4.2 批处理（Batching）

生产者不会每条消息都发一次网络请求，而是先在内存 `RecordAccumulator` 中积累：
- 批次大小达到 `batch.size`（默认 16KB）**或**
- 等待时间达到 `linger.ms`（默认 0ms）

整批一次性发送给 Broker，大幅降低网络往返次数。

### 4.3 零拷贝（Zero-Copy）

消费者读取消息时，Broker 使用 Linux `sendfile` 系统调用：

```
传统路径（4次拷贝）：
磁盘 → 内核缓冲区 → 用户缓冲区 → Socket 缓冲区 → 网卡

sendfile 路径（2次拷贝，0次用户态拷贝）：
磁盘/Page Cache → 网络协议栈 → 网卡
```

Java 层面通过 `FileChannel.transferTo()` 调用 `sendfile64` 系统调用实现。

### 4.4 Page Cache 利用

Kafka 不直接管理内存，而是将内存管理完全交给操作系统 Page Cache：
- **写入**：消息先写 Page Cache，OS 异步刷盘（对生产者返回成功）
- **读取**：先查 Page Cache，命中则无磁盘 I/O

由于 Kafka 消费通常和写入时间接近（时间局部性），Page Cache 命中率极高，相当于内存读写速度。

### 4.5 压缩

生产者可以对整批消息进行压缩（`compression.type=lz4/snappy/zstd`），压缩以批为单位，压缩率远高于单条消息，节省网络带宽和磁盘空间，用 CPU 换 I/O。

---

## 五、消费者组机制

消费者组（Consumer Group）是 Kafka 实现横向扩展消费的核心机制：

### 5.1 分区分配规则

```
Topic 有 4 个分区：P0 P1 P2 P3

情况1：消费者组有 2 个实例
  Consumer-1 → P0, P1
  Consumer-2 → P2, P3

情况2：消费者组有 4 个实例
  Consumer-1 → P0
  Consumer-2 → P1
  Consumer-3 → P2
  Consumer-4 → P3

情况3：消费者组有 5 个实例（多于分区数）
  Consumer-1 → P0  Consumer-2 → P1  Consumer-3 → P2  Consumer-4 → P3
  Consumer-5 → 空闲（浪费）
```

> **最佳实践**：消费者实例数 = 分区数，实现最大并行度且无浪费。

### 5.2 两种消息模型

- **点对点（队列模式）**：所有消费者在同一组，每条消息只被一个消费者处理
- **发布/订阅模式**：不同消费者组各自订阅同一 Topic，每条消息被每个组各消费一次

### 5.3 Rebalance

当消费者组内成员变化（新增、宕机）或 Topic 分区数变化时，触发 Rebalance 重新分配分区。Rebalance 期间所有消费者暂停消费（Stop-the-World），频繁 Rebalance 会影响系统稳定性。

**减少 Rebalance 的配置**：
```properties
# 心跳超时，增大可减少因网络抖动触发的误判 Rebalance
session.timeout.ms=30000
# 心跳间隔（应 < session.timeout.ms / 3）
heartbeat.interval.ms=10000
# 最大拉取间隔，超过则认为消费者挂起
max.poll.interval.ms=300000
```

---

## 六、ZooKeeper 与 KRaft

### 6.1 旧版架构（ZooKeeper 依赖）

ZooKeeper 在 Kafka 旧版本中负责：
- Broker 注册与发现
- Controller 选举（通过抢占 `/controller` 临时节点）
- Topic/Partition 元数据存储
- Consumer Group Offset 存储（已迁移到 Kafka 内部 Topic）

**痛点**：外部依赖增加了运维复杂度；ZooKeeper 成为性能瓶颈（Partition 数超过 20 万时出现问题）。

### 6.2 新版 KRaft 模式（3.0+）

KRaft（Kafka Raft）将元数据管理内置到 Kafka 本身，通过 Raft 共识协议自管：
- 无需单独维护 ZooKeeper 集群
- 元数据操作延迟更低
- Controller 节点可单独部署或与 Broker 混部

---

## 七、相关知识

- [[02_生产者与消费者/01、生产消费与高吞吐原理]] — 生产者工作流程、acks 参数、消费位点管理
- [[03_高可用与可靠性/01、ISR机制与高可用]] — ISR、HW/LEO、Leader 选举、数据不丢失配置
- [[04_生态与实战/01、Kafka Streams与选型]] — Kafka Streams、与 RabbitMQ/RocketMQ 选型对比
