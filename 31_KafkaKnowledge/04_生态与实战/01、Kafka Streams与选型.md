# Kafka Streams 与中间件选型

## 一、Kafka Streams 简介

Kafka Streams 是一个用于构建**实时流处理应用程序的 Java 客户端库**，而非独立的集群式框架。

**核心定位**：
- 作为普通 Java 依赖引入应用程序，无需维护额外的流处理集群
- 深度集成 Kafka，以 Kafka Topic 作为输入输出和内部状态存储的基础
- 天然支持 Kafka 的分区、副本、Consumer Group 模型
- 提供端到端 **Exactly-Once** 语义保证

---

## 二、核心概念

| 概念 | 含义 |
|------|------|
| **KStream** | 无限的持续更新数据流，每条记录是一个键值对事件 |
| **KTable** | 流的最新状态视图，每个 Key 只保留最新值（类似数据库表） |
| **拓扑（Topology）** | 描述数据从输入到输出所有处理步骤的有向无环图 |
| **状态存储（State Store）** | 保存聚合/连接的中间状态（默认基于 RocksDB 本地存储） |
| **Changelog Topic** | 状态存储的备份 Kafka Topic，用于容错恢复 |

### 2.1 三种处理器节点

- **源处理器（Source Processor）**：拓扑起点，从 Kafka Topic 读取数据
- **通用处理器（Stream Processor）**：执行转换、过滤、聚合等业务逻辑
- **Sink 处理器（Sink Processor）**：拓扑终点，将结果写回 Kafka Topic

---

## 三、Streams DSL 核心操作

### 3.1 基础操作

```java
// 创建拓扑
StreamsBuilder builder = new StreamsBuilder();

// 1. 源：从 Kafka Topic 读取
KStream<String, String> orderStream = builder.stream("OrderTopic");

// 2. 过滤：只处理金额大于100的订单
KStream<String, String> bigOrders = orderStream
    .filter((key, value) -> parseAmount(value) > 100);

// 3. 转换：将订单JSON转为通知消息
KStream<String, String> notifications = bigOrders
    .mapValues(value -> "订单通知: " + extractOrderId(value));

// 4. 拆分：一对多展开
KStream<String, String> items = orderStream
    .flatMapValues(value -> parseOrderItems(value));

// 5. Sink：写到另一个 Topic
notifications.to("NotificationTopic");
```

### 3.2 聚合操作（WordCount 示例）

```java
StreamsBuilder builder = new StreamsBuilder();

// 从 "streams-input" 读取文本行
KStream<String, String> textLines = builder.stream("streams-input");

// 单词计数
KTable<String, Long> wordCounts = textLines
    .flatMapValues(line -> Arrays.asList(line.toLowerCase().split("\\W+"))) // 拆分单词
    .groupBy((key, word) -> word)     // 按单词分组（自动创建状态存储）
    .count(Materialized.as("word-count-store")); // 聚合计数

// 将结果流写到输出 Topic
wordCounts.toStream()
    .to("streams-output", Produced.with(Serdes.String(), Serdes.Long()));

// 配置与启动
Properties config = new Properties();
config.put(StreamsConfig.APPLICATION_ID_CONFIG, "word-count-app");
config.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
config.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
config.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
config.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);

KafkaStreams streams = new KafkaStreams(builder.build(), config);
streams.start();

Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
```

### 3.3 窗口操作

| 窗口类型 | 特点 | 适用场景 |
|---------|------|---------|
| **滚动窗口（Tumbling）** | 固定大小、不重叠、连续 | 每分钟交易量统计 |
| **滑动窗口（Sliding）** | 固定大小、窗口间可重叠 | 最近5分钟均值（每1分钟滑动一次） |
| **会话窗口（Session）** | 动态大小，由不活跃间隙分隔 | 用户会话分析 |

```java
// 5分钟滚动窗口 —— 统计每5分钟的订单数
KTable<Windowed<String>, Long> windowedOrderCount = orderStream
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
    .count();

// 会话窗口 —— 用户行为会话分析（5分钟无操作则会话结束）
KTable<Windowed<String>, Long> sessionCount = clickStream
    .groupByKey()
    .windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(5)))
    .count();
```

### 3.4 Join 操作

```java
// KStream-KTable Join：实时订单流关联产品信息表（数据 enrichment）
KTable<String, Product> productTable = builder.table("products");
KStream<String, Order> orderStream = builder.stream("orders");

KStream<String, EnrichedOrder> enrichedOrders = orderStream.join(
    productTable,
    (order, product) -> new EnrichedOrder(order, product) // 连接器
);
enrichedOrders.to("enriched-orders");

// KStream-KStream Join（必须窗口化）：连接点击流和购买流
KStream<String, Click> clickStream = builder.stream("clicks");
KStream<String, Purchase> purchaseStream = builder.stream("purchases");

KStream<String, UserBehavior> behavior = clickStream.join(
    purchaseStream,
    (click, purchase) -> new UserBehavior(click, purchase),
    JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(10)) // 10分钟内的点击和购买认为相关
);
```

### 3.5 状态存储与交互式查询

```java
// 创建带名称的状态存储
KTable<String, Long> counts = stream
    .groupByKey()
    .count(Materialized.as("my-count-store"));

// 启动后通过交互式查询直接读取状态（无需再从 Topic 读回）
KafkaStreams streams = new KafkaStreams(builder.build(), config);
streams.start();

ReadOnlyKeyValueStore<String, Long> store =
    streams.store(StoreQueryParameters.fromNameAndType(
        "my-count-store",
        QueryableStoreTypes.keyValueStore()
    ));

Long count = store.get("some-word"); // 直接查询实时状态
```

---

## 四、Kafka Streams vs Spark Streaming

| 维度 | Kafka Streams | Spark Streaming |
|------|--------------|-----------------|
| **架构** | 客户端库，嵌入应用 | 独立集群框架（需 Spark 集群） |
| **部署** | 极简，仅需 Kafka | 重量级，需维护 Spark 集群 |
| **处理模型** | 逐条记录，毫秒级延迟 | 微批处理，秒级延迟 |
| **数据源** | 仅限 Kafka | Kafka/HDFS/S3/多种来源 |
| **状态管理** | 本地 RocksDB + Changelog Topic | HDFS 或集群内存 |
| **生态** | Kafka 生态原生 | 与 Spark SQL/MLlib/GraphX 集成 |

**选型建议**：
- **选 Kafka Streams**：纯 Kafka 生态、追求极低延迟、架构简单轻量、无需批处理/ML 任务
- **选 Spark Streaming**：多数据源、批流一体（Lambda 架构）、需要 SQL/ML 深度集成

---

## 五、三大消息队列选型对比

### 5.1 Kafka vs RabbitMQ

| 维度 | Kafka | RabbitMQ |
|------|-------|---------|
| 核心模型 | 分布式提交日志 | 企业级消息代理（AMQP） |
| 吞吐量 | 极高（百万级 TPS） | 高（万级 TPS） |
| 消息留存 | 持久化，按策略清理，可重放 | 消费后默认删除 |
| 路由机制 | 基于 Key 哈希到分区，相对简单 | 极强大（Direct/Fanout/Topic/Headers Exchange） |
| 延时队列 | 不支持（需自实现） | 原生支持（TTL+DLX 或延迟插件） |
| 消息顺序 | 分区内严格有序 | 单队列单消费者有序 |
| 适用场景 | 日志收集、数据管道、流计算、事件溯源 | 业务解耦、任务队列、复杂路由、延时任务 |

### 5.2 Kafka vs RocketMQ

| 维度 | Kafka | RocketMQ |
|------|-------|---------|
| 设计背景 | 大数据日志、流处理平台 | 金融级业务消息 |
| 事务消息 | 支持（但与外部事务协调复杂） | 原生支持（半消息机制，两阶段提交） |
| 定时消息 | 不支持 | 原生支持（4.x 18级别，5.x 自定义） |
| 消息顺序 | 分区内有序（主节点宕机可能短暂乱序） | 严格顺序消息（宁可不可用，不乱序） |
| 多队列性能 | 分区过多时性能明显下降 | CommitLog 混合存储，万级队列性能稳定 |
| 消息追踪 | 需外部组件 | 原生消息轨迹支持 |
| 生态集成 | 大数据（Flink/Spark/Hadoop） | Java/微服务（Spring Cloud Alibaba） |
| 适用场景 | 日志采集、实时数仓、流处理 | 电商交易、金融支付、分布式事务 |

### 5.3 Kafka vs ActiveMQ

| 维度 | Kafka | ActiveMQ |
|------|-------|----------|
| 定位 | 分布式日志系统、流处理平台 | 企业级消息中间件（JMS 实现） |
| 吞吐量 | 极高（百万级 TPS） | 中等（万级 TPS） |
| 消息持久化 | 顺序写磁盘，Page Cache 加速 | JDBC/KahaDB/LevelDB |
| 消息协议 | 自定义二进制协议 | 支持 AMQP、OpenWire、STOMP、MQTT |
| 消息模型 | 发布/订阅（基于分区） | Queue + Topic 两种模式 |
| 集群支持 | 原生分布式，副本机制 | 主从复制、Network of Brokers |
| 运维复杂度 | 依赖 ZooKeeper/KRaft | 相对简单 |
| 适用场景 | 日志收集、数据管道、流计算 | 传统企业级应用、兼容 JMS |

### 5.4 Kafka vs ZeroMQ

| 维度 | Kafka | ZeroMQ |
|------|-------|--------|
| 定位 | 分布式消息队列（有 Broker） | 底层网络通讯库（无 Broker） |
| 架构 | 有中心节点（Broker） | 去中心化（P2P） |
| 消息模式 | 发布/订阅、队列 | Request-Reply、Publisher-Subscriber、Pipeline |
| 吞吐量 | 高 | 极高（轻量级） |
| 持久化 | 支持（基于磁盘） | 不支持 |
| 消息确认 | 支持（ACK 机制） | 支持（但更底层） |
| 适用场景 | 消息队列、实时流处理 | 进程间通信、微服务间通讯、实时推送 |

> **ZeroMQ 特点**：又称 "史上最快的消息队列"，本质是一个 Socket 封装库，将网络通讯、进程通讯和线程通讯抽象为统一 API。与 RabbitMQ 相比，ZMQ 更像是一个底层通讯库而非传统消息服务器。

### 5.5 四维选型决策矩阵

| 场景 | 推荐选择 | 理由 |
|------|---------|------|
| 日志收集、用户行为追踪 | **Kafka** | 极高吞吐，数据可重放 |
| 实时流计算（Flink/Spark） | **Kafka** | 大数据生态的事实标准 |
| 业务系统解耦（复杂路由） | **RabbitMQ** | 强大的 Exchange 路由机制 |
| 延时/定时任务 | **RabbitMQ** | 原生死信队列+TTL 支持 |
| 电商订单、金融交易 | **RocketMQ** | 事务消息+顺序消息+延时消息开箱即用 |
| 分布式事务一致性 | **RocketMQ** | 半消息机制，最成熟的方案 |
| 海量 Topic/队列数量 | **RocketMQ** | CommitLog 混合存储，支持数万级队列 |
| 云原生弹性扩展 | **Pulsar** | 存算分离，秒级弹性 |
| 需要 JMS 兼容性 | **ActiveMQ** | 完全支持 JMS1.1 规范 |
| 进程间高性能通讯 | **ZeroMQ** | 轻量级、极低延迟、无需消息队列 |

### 5.4 混合架构模式

大型系统中常见的组合方案：

```
核心业务（订单/支付/库存）
    ↓ RocketMQ（事务消息、高可靠）
    
用户行为日志（点击/浏览/搜索）
    ↓ Kafka（高吞吐，接入 Flink 做实时计算）
    
通知/邮件/短信（复杂路由需求）
    ↓ RabbitMQ（灵活的 Exchange 路由）
```

---

## 六、Kafka 性能优化实践

### 6.1 生产者侧优化

```properties
# 批量发送（延迟换吞吐）
batch.size=65536          # 64KB，增大批次
linger.ms=10              # 等10ms填充批次

# 压缩（CPU 换 I/O）
compression.type=lz4      # lz4 是吞吐量和延迟的最佳平衡

# 内存缓冲
buffer.memory=134217728   # 128MB，应对流量峰值

# 并行度
max.in.flight.requests.per.connection=5  # 开启幂等性时最多5个未确认请求
```

### 6.2 消费者侧优化

```properties
# 批量拉取
max.poll.records=500          # 每次 poll 最多拉 500 条
fetch.min.bytes=1024          # 等累积到 1KB 才返回（减少网络往返）
fetch.max.wait.ms=500         # 最多等 500ms
max.partition.fetch.bytes=1048576  # 每个分区最多 1MB

# 避免 Rebalance
session.timeout.ms=30000
heartbeat.interval.ms=10000
max.poll.interval.ms=300000
```

### 6.3 Broker 侧优化

```properties
# 副本同步
num.replica.fetchers=4        # 增加副本拉取线程数
replica.fetch.min.bytes=1     # 立即拉取（低延迟）

# 网络和 I/O 线程
num.network.threads=8
num.io.threads=16

# 日志段大小（大批量写入场景可适当增大）
log.segment.bytes=1073741824  # 1GB

# 消息保留（根据磁盘容量调整）
log.retention.hours=168       # 7天
log.retention.bytes=107374182400  # 每个 Partition 最多100GB
```

---

## 七、Kafka 安全体系

```properties
# broker.properties —— SSL/TLS + SASL 认证
listeners=SASL_SSL://0.0.0.0:9093
security.inter.broker.protocol=SASL_SSL

ssl.keystore.location=/path/to/kafka.server.keystore.jks
ssl.keystore.password=keystore_password
ssl.truststore.location=/path/to/kafka.server.truststore.jks
ssl.truststore.password=truststore_password

sasl.mechanism.inter.broker.protocol=PLAIN
sasl.enabled.mechanisms=PLAIN

# ACL 授权
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
```

```bash
# 为用户授权：producer 用户对 OrderTopic 有写权限
kafka-acls.sh --bootstrap-server localhost:9093 \
  --add --allow-principal User:producer-user \
  --operation Write --topic OrderTopic \
  --command-config admin.properties

# consumer 用户对 OrderTopic 有读权限
kafka-acls.sh --bootstrap-server localhost:9093 \
  --add --allow-principal User:consumer-user \
  --operation Read --topic OrderTopic \
  --operation Read --group order-consumer-group \
  --command-config admin.properties
```

---

## 八、相关知识

- [[01_核心架构/01、Kafka架构与存储原理]] — Kafka 整体架构、存储机制、Consumer Group
- [[02_生产者与消费者/01、生产消费与高吞吐原理]] — acks、幂等性、事务、Offset 管理
- [[03_高可用与可靠性/01、ISR机制与高可用]] — ISR、HW/LEO、Leader 选举
