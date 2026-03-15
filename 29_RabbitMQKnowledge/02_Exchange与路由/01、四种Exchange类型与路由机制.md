# 四种 Exchange 类型与路由机制

## 一、四种 Exchange 类型总览

RabbitMQ 遵循 AMQP 0-9-1 协议，提供四种核心 Exchange 类型，在 Java 客户端由 `BuiltinExchangeType` 枚举定义：

- **Direct**：精确匹配 Routing Key
- **Fanout**：广播，忽略 Routing Key（性能最高）
- **Topic**：通配符模糊匹配 Routing Key
- **Headers**：基于消息头属性匹配，不使用 Routing Key

此外还有一个**默认交换器（Default Exchange）**：名称为空字符串 `""`，是一个预置的 Direct 类型交换器，每个队列创建时自动以队列名为 Binding Key 绑定到它。

---

## 二、Direct Exchange（直连交换机）

**路由规则**：精确匹配。消息的 Routing Key 与 Binding Key **完全相同**时才路由。

**内部实现**：类似哈希表查找，时间复杂度接近 O(1)，非常高效。

```java
// 1. 声明 Direct Exchange（持久化）
channel.exchangeDeclare("my_direct_exchange", BuiltinExchangeType.DIRECT, true);

// 2. 声明队列并绑定，Binding Key = "error"
channel.queueDeclare("error_queue", true, false, false, null);
channel.queueBind("error_queue", "my_direct_exchange", "error");

// 3. 发送消息，Routing Key = "error" → 精确匹配，路由到 error_queue
channel.basicPublish("my_direct_exchange", "error", null, "An error occurred!".getBytes());
```

**典型场景**：日志系统，按级别（`info`/`warning`/`error`）路由到不同队列。

---

## 三、Fanout Exchange（扇出交换机）

**路由规则**：广播。**完全忽略** Routing Key，将消息无条件路由到所有绑定的队列。

**性能**：所有类型中**最高**，无需任何匹配计算。

```java
channel.exchangeDeclare("my_fanout_exchange", BuiltinExchangeType.FANOUT, true);

// 绑定三个队列，Routing Key 随意填写（会被忽略）
channel.queueBind("queue_a", "my_fanout_exchange", "");
channel.queueBind("queue_b", "my_fanout_exchange", "");
channel.queueBind("queue_c", "my_fanout_exchange", "");

// 发送一条消息 → queue_a、queue_b、queue_c 都会收到
channel.basicPublish("my_fanout_exchange", "any.key.ignored", null, "Broadcast!".getBytes());
```

**典型场景**：系统全局广播、用户上线通知、需要同时更新多个分布式缓存。

---

## 四、Topic Exchange（主题交换机）

**路由规则**：通配符模糊匹配。Routing Key 和 Binding Key 都是由 `.` 分隔的多段字符串。

**通配符规则**：
- `*`（星号）：匹配**恰好一个**单词。`*.orange.*` 匹配 `quick.orange.rabbit`，但不匹配 `quick.orange`
- `#`（井号）：匹配**零个或多个**单词。`lazy.#` 匹配 `lazy`、`lazy.orange`、`lazy.orange.rabbit`

**内部实现**：类似 Trie 树（前缀树），高效处理通配符的模式匹配。

```java
channel.exchangeDeclare("my_topic_exchange", BuiltinExchangeType.TOPIC, true);

// 队列1：关注所有美国体育消息
channel.queueBind("usa_sports", "my_topic_exchange", "usa.sports.*");

// 队列2：关注所有新闻（无论来自哪里）
channel.queueBind("all_news", "my_topic_exchange", "#.news");

// "usa.sports.news" 同时匹配两个 Binding Key → 两个队列都收到
channel.basicPublish("my_topic_exchange", "usa.sports.news", null, "USA wins!".getBytes());
```

**通配符使用注意**：
- `*` 的位置必须对应**恰好存在**的单词，`usa.*` 匹配 `usa.news` 但不匹配 `usa`
- 单独的 `#` 匹配所有消息，效果类似 Fanout Exchange
- 可以组合使用，如 `*.news.*` 匹配任何中间部分为 `news` 的三段式路由键

**典型场景**：股票行情推送（`stock.nyse.ibm`）、新闻分类（`sports.us.basketball`）等多层级动态订阅。

---

## 五、Headers Exchange（头交换机）

**路由规则**：忽略 Routing Key，根据消息的 **Headers 属性**（键值对集合）进行匹配。

绑定时通过 `x-match` 参数控制匹配方式：
- `x-match = all`：消息 Headers 必须**包含所有**绑定时指定的键值对（逻辑 AND）
- `x-match = any`：消息 Headers 只需**包含任意一个**绑定键值对（逻辑 OR）

```java
channel.exchangeDeclare("my_headers_exchange", BuiltinExchangeType.HEADERS, true);

// 绑定队列，设置匹配条件：format=pdf AND type=report
Map<String, Object> bindingArgs = new HashMap<>();
bindingArgs.put("format", "pdf");
bindingArgs.put("type", "report");
bindingArgs.put("x-match", "all");  // 必须全部满足
channel.queueBind("pdf_reports", "my_headers_exchange", "", bindingArgs);

// 发送带 Headers 的消息
Map<String, Object> headers = new HashMap<>();
headers.put("format", "pdf");
headers.put("type", "report");
AMQP.BasicProperties props = new AMQP.BasicProperties.Builder().headers(headers).build();
channel.basicPublish("my_headers_exchange", "", props, "Report content".getBytes());
```

**注意**：Headers Exchange 需要进行复杂的属性匹配，**性能相对最差**，实际业务中远不如前三种常用。

**适用场景**：路由规则基于消息自身多种属性（如消息格式、版本号、来源系统）的复杂场景。

---

## 六、默认交换器（Default Exchange）

名称为空字符串 `""`，预置的 Direct 类型交换器。每个队列创建时，自动以**队列名**为 Binding Key 绑定到默认交换器。

```java
// 不指定 Exchange，直接发消息到队列
// RabbitMQ 使用默认交换器，Routing Key = 队列名
channel.basicPublish("", "my_queue", null, "Hello World".getBytes());
// 等价于：已预置绑定 channel.queueBind("my_queue", "", "my_queue")
```

这也是很多入门教程发消息的方式——看起来直接发队列，实际经过了默认交换器。
