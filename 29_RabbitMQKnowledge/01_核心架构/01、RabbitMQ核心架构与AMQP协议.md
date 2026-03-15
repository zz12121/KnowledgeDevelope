# RabbitMQ 核心架构与 AMQP 协议

## 一、RabbitMQ 概述

RabbitMQ 是实现了 **AMQP（高级消息队列协议）** 的开源消息代理软件，用 **Erlang/OTP** 语言编写。Erlang 天生为高并发、分布式、容错系统而生，使 RabbitMQ 具备出色的可靠性和可伸缩性。

**核心设计目标**：应用解耦、异步处理、流量削峰、可靠消息传递。

**默认端口**：消息服务 `5672`，Web 管理界面 `15672`。

**Java 集成方式**：
- 原生：`amqp-client` 库
- Spring Boot：`spring-boot-starter-amqp`

---

## 二、AMQP 协议

AMQP（Advanced Message Queuing Protocol）是一个**线级协议（Wire-level Protocol）**，定义了通过网络传输的字节流格式。任何实现了 AMQP 的客户端都能与任何实现了 AMQP 的 Broker 通信，实现跨语言、跨平台互操作。

**核心特性**：
- **二进制协议**，高效紧凑
- 支持消息确认、持久化、路由、安全等高级特性
- RabbitMQ 实现的是 AMQP 0-9-1 并做了扩展

**AMQP 核心模型**：生产者 → 交换器 → 队列 → 消费者

---

## 三、核心组件详解

### 生产者（Producer）
消息的发送方。在 Java 中通过 `ConnectionFactory` 创建连接，通过 `Channel` 发送消息。生产者**只向 Exchange 发送消息**，不直接操作 Queue。

### 交换器（Exchange）
消息路由的"大脑"。接收生产者发送的消息，根据路由规则（Binding + Routing Key）决定消息投递到哪些队列。**生产者永远不直接向队列发消息**，必须经过 Exchange。

支持四种类型：Direct / Fanout / Topic / Headers（详见《四种 Exchange 类型》）。

### 队列（Queue）
消息的存储容器，FIFO 结构，但支持优先级。队列属性：
- `durable`：是否持久化（重启后保留）
- `exclusive`：是否独占（仅当前连接使用，连接断开后删除）
- `autoDelete`：是否自动删除（最后一个消费者断开后删除）

### 绑定（Binding）
连接 Exchange 和 Queue 的规则，即"告诉 Exchange 哪些消息应该发到哪些 Queue"。绑定包含三要素：Exchange、Queue、Binding Key（路由规则）。

```java
// Direct Exchange 绑定
channel.queueBind("queue_name", "exchange_name", "routing_key");

// Fanout Exchange 绑定（忽略路由键）
channel.queueBind("queue_name", "exchange_name", "");

// Topic Exchange 绑定（通配符）
channel.queueBind("queue_name", "exchange_name", "*.error");
```

### 路由键（Routing Key）
生产者发送消息时携带的字符串属性。Exchange 用它结合 Binding Key 决定路由路径：
- Direct：精确匹配
- Topic：通配符匹配
- Fanout：忽略
- Headers：不使用 Routing Key，基于消息头匹配

### 虚拟主机（Virtual Host）
逻辑隔离机制，类似命名空间。一个 RabbitMQ 实例可划分多个 vhost，每个 vhost 拥有独立的 Exchange、Queue、Binding 和权限系统。

```bash
rabbitmqctl add_vhost /my_vhost
rabbitmqctl set_permissions -p /my_vhost username ".*" ".*" ".*"
```

常见用途：为开发、测试、生产环境分别创建不同 vhost，实现多租户隔离。

---

## 四、Connection 与 Channel

这是 RabbitMQ 中最重要的性能优化概念：

**Connection（连接）**
- 物理层面的 TCP 连接
- 建立和销毁成本**非常高**（约 15 个 TCP 报文交互）
- 应用程序应与 Broker 保持**少量长连接**

**Channel（信道）**
- 基于 TCP 连接的**逻辑虚拟连接**
- 开销**极低**，只是在 TCP 连接上多路复用的逻辑概念
- 一个 Connection 下可创建**多个** Channel
- 执行所有 AMQP 操作（声明队列、发送/消费消息等）

```java
// 建立昂贵的 TCP 连接（应保持长连接）
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");
Connection connection = factory.newConnection();  // 高开销

// 在同一 TCP 连接上创建多个轻量信道
Channel channel1 = connection.createChannel();  // 低开销
Channel channel2 = connection.createChannel();
// 不同线程使用不同 Channel，但可以共享同一个 Connection
```

**设计意义**：如果没有 Channel 机制，每个线程都需要独立的 TCP 连接，会迅速耗尽系统资源。Channel 实现了连接复用，大大降低了开销。
