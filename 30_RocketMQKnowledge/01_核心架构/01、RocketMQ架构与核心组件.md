# RocketMQ 架构与核心组件

## 一、四大核心组件

### NameServer — 轻量级路由中枢

**设计目标**：轻量级、无状态、高可用。所有路由信息（Topic、Broker、Queue 映射）全量存储在内存中，通过 HashMap 实现 O(1) 快速查询。

**工作机制：**
- **服务注册**：Broker 启动时与所有 NameServer 建立长连接，每隔 30 秒发送心跳包上报路由信息
- **故障剔除**：定时任务（默认 10 秒）扫描心跳时间，超过 120 秒未上报的 Broker 会从路由表中剔除
- **路由发现**：Producer/Consumer 启动时，轮询 NameServer 集群拉取完整路由信息

**高可用设计**：NameServer 节点间**互不通信**，每节点维护全量元数据，去中心化设计。只要有一个节点存活，整个系统就能正常服务。

### Broker — 消息存储与转发核心

Broker 是 RocketMQ 的"心脏"，承担消息的持久化、投递和高可用保障。

**角色架构**：
- Master（主节点，`brokerId=0`）：负责读写
- Slave（从节点，`brokerId≠0`）：负责读和备份

**存储层（核心文件三件套）：**

| 文件 | 作用 |
|------|------|
| **CommitLog** | 所有 Topic 消息按时间顺序统一追加写入，单文件默认 1GB，顺序 I/O |
| **ConsumeQueue** | 消费索引文件，每个 Topic 的每个 Queue 对应一个文件，存储消息在 CommitLog 中的偏移量 + 大小 + Tag 哈希码，定长 20 字节/条目 |
| **IndexFile** | 哈希索引文件，为消息 Key 建立索引，支持按 Key 或时间范围查询消息 |

**数据写入流程：**
```
Producer → CommitLog（顺序写）→ ReputMessageService（异步分发）→ ConsumeQueue + IndexFile
```

Consumer 消费时：先读 ConsumeQueue 获取偏移量 → 再根据偏移量从 CommitLog 读取消息体。

### Producer — 消息生产者

**负载均衡与故障规避**：
- 从 NameServer 获取 Topic 路由后，通过负载均衡策略（如轮询）选择 MessageQueue
- 内置 `MQFaultStrategy` 记录 Broker 响应延迟，自动规避响应慢或故障的 Broker

**三种发送方式：**
- **同步发送**：等待 SendResult，适合重要消息
- **异步发送**：带回调，高吞吐场景
- **单向发送（Oneway）**：不关心结果，日志场景

```java
// 同步发送（推荐关键消息使用）
producer.setRetryTimesWhenSendFailed(3);
SendResult result = producer.send(msg);
if (result.getSendStatus() != SendStatus.SEND_OK) {
    // 触发告警或补偿
}
```

### Consumer — 消息消费者

**两种消费模式：**
- **集群消费（默认）**：同一 Consumer Group 内多个实例分摊消费，每条消息只被消费一次
- **广播消费**：组内每个实例都消费全量消息，进度存在 Consumer 本地

**重试与死信：**
- 消费失败返回 `RECONSUME_LATER`，消息进入重试队列（`%RETRY%GroupName`）
- 默认重试 16 次，间隔逐渐递增（10s、30s、1m、2m...直到 2h）
- 16 次全部失败 → 进入死信队列（`%DLQ%GroupName`），需人工干预

---

## 二、Topic 与 Queue

**Topic（主题）**：消息的逻辑分类，是生产者/消费者关注的概念。

**MessageQueue（消息队列）**：Topic 的物理分片单位，是负载均衡和并行计算的基本元素。

**关键区别：**
- 消费者以**组**为单位订阅 Topic
- 集群消费模式下，一个 Queue 同一时刻只能被**同一 Group 内的一个消费者**消费
- Queue 内部消息严格 FIFO，是顺序消息的保证基础
- **增加 Queue 数量**是提升 Topic 吞吐能力的主要手段

---

## 三、Tag 与 Key

**Tag（标签）**：消息的二级分类标签，一条消息有且只有一个 Tag。
- 在 **Broker 端过滤**，无需读取 CommitLog 消息体，只在 ConsumeQueue 中基于 Tag 哈希值匹配，性能损耗极低
- 用法：`consumer.subscribe("OrderTopic", "PaySuccess || PayFail")`

**Key（业务键）**：消息的业务唯一标识符，用于运维调试和消息查询。
- Broker 为设置了 Key 的消息建立 IndexFile 哈希索引
- 用法：`message.setKeys("ORDER_20231027_001")`，可通过管理控制台或 `mqadmin queryMsgByKey` 快速定位

---

## 四、RocketMQ vs Kafka

| 维度 | RocketMQ | Kafka |
|------|----------|-------|
| 设计定位 | 金融级、低延迟、高可靠业务消息 | 高吞吐、高性能实时流数据平台 |
| 消息追踪 | 原生支持消息轨迹 | 需依赖外部组件 |
| 消息类型 | 延迟/事务/顺序/过滤等丰富类型 | 基础发布/订阅 |
| 生态 | Spring Cloud Alibaba 微服务套件 | 大数据生态（Flink/Spark） |
| 适用场景 | 电商/金融/交易/事务一致性 | 日志收集/流处理/数据管道 |

### 4.1 为什么 Kafka 无法满足需求

阿里在早期使用 ActiveMQ 5.x，随着业务增长遇到了性能瓶颈。转而尝试 Kafka 时发现以下问题：

| Kafka 问题 | 说明 |
|-----------|------|
| **分区限制** | 写入并行性受分区数量限制，分区越多，写入变随机，性能下降 |
| **IO 争用** | 多分区导致数据文件分散，Linux IO Group Commit 机制难以发挥作用 |
| **延迟较高** | 专为日志设计，对低延迟场景不友好 |
| **事务支持弱** | 与外部事务协调复杂 |

### 4.2 RocketMQ 如何支持海量队列

RocketMQ 采用独特的存储架构解决了这个问题：

**核心设计：**
- **CommitLog**：所有消息统一顺序写入一个文件，完全顺序 I/O
- **ConsumeQueue**：每个 Topic/Queue 对应的消费索引文件，轻量级元数据

**优势对比：**

| 特性 | Kafka | RocketMQ |
|------|-------|----------|
| 写入模型 | 多分区文件分散写入 | 单一 CommitLog 顺序写入 |
| 队列扩展 | 分区多则性能下降 | 队列多不影响写入性能 |
| 读取模型 | 从分区文件随机读 | 先读 ConsumeQueue 索引，再读 CommitLog |
| 最大队列数 | 建议不超过 100 | 支持数万级队列 |

> RocketMQ 通过"写顺序、读随机"的巧妙设计，在保证高吞吐的同时支持海量队列，成为阿里双十一等大规模业务的基石。
