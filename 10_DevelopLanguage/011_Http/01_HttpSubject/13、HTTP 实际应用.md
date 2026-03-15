# 十三、HTTP 实际应用

> 关联知识：[[27_SpringCloudKnowledge]] | [[35_NettyKnowledge/05_协议与实战应用]]

###### 1. 什么是 RESTful API？

REST（Representational State Transfer，表述性状态转移）是一种 API 设计风格，不是协议或规范，而是一套**约束原则**。

**核心思想**：把网络上的资源用 URL 表示，用 HTTP 方法表达对资源的操作。

**RESTful API 设计示例**：

| 操作 | 非 RESTful（RPC 风格） | RESTful |
|------|-----------------------|---------|
| 获取用户列表 | GET /getUsers | GET /users |
| 获取单个用户 | GET /getUserById?id=1 | GET /users/1 |
| 创建用户 | POST /createUser | POST /users |
| 更新用户 | POST /updateUser | PUT /users/1 |
| 删除用户 | GET /deleteUser?id=1 | DELETE /users/1 |

**六大约束**：
- **无状态**：每次请求必须携带完整上下文
- **统一接口**：用 HTTP 方法 + 资源路径表达语义
- **资源标识**：每个资源有唯一 URL
- **资源表述**：同一资源可以有多种表示（JSON/XML）
- **超媒体驱动（HATEOAS）**：响应中包含下一步操作的链接（实际很少实现）
- **按需代码**（可选）：服务端可以返回可执行代码

---

###### 2. RESTful API 的设计原则有哪些？

**URL 设计原则**：

- 名词复数表示资源集合：`/users`、`/orders`
- 资源嵌套表示层级关系：`/users/1/orders`（用户1的订单）
- URL 使用小写字母和连字符：`/user-profiles`（不要驼峰）
- 不要在 URL 里放动词：`/users/1/delete` ❌ → `DELETE /users/1` ✅
- 过滤/排序/分页用查询参数：`/users?status=active&page=1&size=20&sort=name`

**HTTP 方法语义**：
- GET：查询（安全且幂等）
- POST：创建（不幂等）
- PUT：全量更新（幂等）
- PATCH：局部更新（通常幂等）
- DELETE：删除（幂等）

**响应设计**：
- 使用正确的 HTTP 状态码（不要万能 200）
- 响应格式统一（通常 `{"code": 0, "data": {}, "message": "success"}`）
- 错误响应包含足够的诊断信息
- 支持版本控制：`/api/v1/users` 或 Header 版本 `Accept: application/vnd.api+json;version=1`

**幂等性设计**：
- 创建操作提供幂等 Key（`Idempotency-Key: uuid`），防止重复提交

---

###### 3. 什么是幂等性？哪些 HTTP 方法是幂等的？

**幂等性**：对同一请求执行多次，产生的效果和执行一次相同。

| HTTP 方法 | 幂等 | 安全（不改变状态） | 说明 |
|-----------|-----|-------------------|------|
| GET | ✅ | ✅ | 查询，多次结果相同 |
| HEAD | ✅ | ✅ | 同 GET，只要头部 |
| OPTIONS | ✅ | ✅ | 查询服务器能力 |
| PUT | ✅ | ❌ | 全量替换，多次结果相同 |
| DELETE | ✅ | ❌ | 删了就是没有，再删还是没有 |
| POST | ❌ | ❌ | 每次可能创建新资源 |
| PATCH | ❌（通常）| ❌ | 取决于实现（增量操作不幂等） |

**幂等性的实际意义**：幂等的操作可以安全重试，不幂等的操作重试可能导致数据重复。

**POST 幂等化方案**：
- 客户端生成唯一 `Idempotency-Key`（UUID），放在请求头里
- 服务端把已处理的 Key 存 Redis，相同 Key 的请求直接返回缓存的响应
- 常见于支付、订单创建等场景（防止网络超时重试导致重复下单）

---

###### 4. 什么是 WebSocket？WebSocket 和 HTTP 有什么区别？

**WebSocket** 是 HTML5 引入的协议，提供**全双工、持久化**的通信通道——服务器可以主动向客户端推送消息，不需要客户端轮询。

**建立连接（HTTP Upgrade）**：

WebSocket 利用 HTTP 做握手，然后升级到 WebSocket 协议：

```
客户端请求：
GET /ws HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

服务端响应（101 Switching Protocols）：
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

之后就是纯 WebSocket 帧格式，与 HTTP 无关了。

**HTTP vs WebSocket 对比**：

| 维度 | HTTP | WebSocket |
|------|------|-----------|
| 通信方式 | 请求-响应（半双工） | 全双工 |
| 连接状态 | 无状态（短连接或 Keep-Alive 串行） | 持久连接 |
| 服务器推送 | 不支持（轮询变通） | 原生支持 |
| 开销 | 每次请求都有 Header | 握手后帧开销极小 |
| 适用场景 | 页面加载、API 调用 | 实时通信 |

**WebSocket 适用场景**：在线聊天/IM、实时游戏、股票行情推送、实时协作（在线文档）、IoT 设备数据流。

关联知识：[[35_NettyKnowledge/05_协议与实战应用]]

---

###### 5. 什么是 Server-Sent Events（SSE）？

SSE（Server-Sent Events）是一种**服务器向客户端单向推送**的技术，比 WebSocket 更简单，基于普通 HTTP。

**特点**：
- 服务器可以持续向客户端推送数据，但客户端不能反向发送（单向）
- 基于 HTTP 长连接（`Content-Type: text/event-stream`）
- 浏览器原生支持，断线自动重连（WebSocket 需要手动实现）
- 只支持文本，不支持二进制（WebSocket 两者都支持）

**数据格式**：

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

event: message
data: {"msg": "你好"}

id: 1
data: 第一条消息

data: 第二条消息（分2行）
data: 还有这行
```

**前端使用**：

```javascript
const evtSource = new EventSource('/events');
evtSource.onmessage = (event) => {
    console.log('收到:', event.data);
};
evtSource.onerror = () => evtSource.close();
```

**SSE vs WebSocket 选择**：
- 只需要服务器推送（股票行情、新闻推送、AI 流式输出）→ SSE（更简单）
- 需要双向通信（聊天、游戏）→ WebSocket

**SSE 现在的热门应用**：ChatGPT 流式输出就是用 SSE 实现的，边生成边推送 token。

---

###### 6. HTTP 如何实现断点续传？

断点续传允许下载中断后从中断处继续，不需要重新下载整个文件。

**核心机制**：`Range` 请求头 + `206 Partial Content` 响应。

**完整流程**：

1. **第一步**：客户端用 HEAD 请求获取文件信息：
```
HEAD /large-file.zip HTTP/1.1
← Content-Length: 104857600   (100MB)
← Accept-Ranges: bytes         (服务器支持断点续传)
← ETag: "abc123"
```

2. **第二步**：客户端发 Range 请求：
```
GET /large-file.zip HTTP/1.1
Range: bytes=0-1048575          (请求前 1MB)
```

3. **第三步**：服务器响应 206：
```
HTTP/1.1 206 Partial Content
Content-Range: bytes 0-1048575/104857600
Content-Length: 1048576
```

4. **断点续传**：下载到 50MB 时中断，续传时：
```
Range: bytes=52428800-          (从 50MB 处继续到结尾)
```

**多线程下载（分片下载）**：

把文件分成多块，每块用不同的 Range 请求并行下载，下完后合并。下载工具（IDM、aria2）就是这么实现的：

```
线程1: Range: bytes=0-25165823
线程2: Range: bytes=25165824-50331647
线程3: Range: bytes=50331648-75497471
线程4: Range: bytes=75497472-104857599
```

---

###### 7. Range 请求头的作用是什么？

`Range` 请求头指定**只请求资源的某一部分**，配合 `206 Partial Content` 状态码使用。

**格式**：

```
Range: bytes=<start>-<end>        # 请求指定范围（闭区间）
Range: bytes=<start>-             # 从 start 到结尾
Range: bytes=-<suffix>            # 最后 suffix 字节
Range: bytes=0-1023, 2048-4095    # 多范围请求（少用）
```

**常见使用场景**：
- 断点续传（如上题）
- 视频分段播放：视频播放器不下载整个视频，只下当前播放位置附近的数据
- 大文件多线程下载
- 日志文件尾部读取（只取最新部分）

**服务端响应**：
- 请求范围有效 → `206 Partial Content` + `Content-Range: bytes <start>-<end>/<total>`
- 请求范围超出文件大小 → `416 Range Not Satisfiable`
- 服务器不支持 Range → `200 OK` 返回完整文件

---

###### 8. 什么是分块传输编码（Chunked Transfer Encoding）？

分块传输允许服务器**在不知道响应体总大小的情况下**，把响应体分块发送。

**使用场景**：
- 动态生成内容（边计算边发送，不等全部算完才响应）
- 大文件流式传输
- SSE（Server-Sent Events）

**请求头**：`Transfer-Encoding: chunked`（不带 `Content-Length`）

**数据格式**：每个块包含块大小（十六进制）+ 换行 + 块数据 + 换行，最后一个块大小为 0 表示结束。

```
HTTP/1.1 200 OK
Transfer-Encoding: chunked

7\r\n
Mozilla\r\n
9\r\n
Developer\r\n
7\r\n
Network\r\n
0\r\n
\r\n
```

**与 Content-Length 的对比**：
- `Content-Length`：提前知道大小，客户端可以显示进度条
- `chunked`：不需要提前知道大小，适合流式响应，但客户端无法显示进度

**HTTP/2 的改进**：HTTP/2 的 DATA 帧本身支持流式传输，`Transfer-Encoding: chunked` 在 HTTP/2 中被禁止使用（HTTP/2 有自己的流控机制）。

---

###### 9. 什么是 HTTP 长轮询（Long Polling）？

长轮询是在 WebSocket 出现之前，实现"准实时"服务器推送的变通方案。

**普通轮询**：客户端每隔固定时间（如 1 秒）发一次请求，大量请求可能都是空的（没有新数据）。

**长轮询**：
1. 客户端发请求
2. 服务器**有数据时才响应**，没有数据就保持连接，一直等
3. 服务器推数据（或超时）后响应
4. 客户端收到响应后，**立即发下一个请求**，循环往复

```
客户端 → GET /events (等待)
服务器 → (等了 5 秒，有新消息了) 200 OK + 消息数据
客户端 → GET /events (立即再次请求)
...
```

**与短轮询/WebSocket 对比**：

| 方式 | 实时性 | 服务器压力 | 适用场景 |
|------|--------|-----------|----------|
| 短轮询 | 差（有延迟） | 高（大量无效请求） | 实时要求低 |
| 长轮询 | 接近实时 | 中（有效请求，但连接数多） | 简单实时需求 |
| SSE | 接近实时 | 低（持久连接单向推） | 服务器→客户端推送 |
| WebSocket | 实时 | 低（持久全双工连接） | 双向实时通信 |

---

###### 10. HTTP 和 WebSocket 的应用场景有什么不同？

**用 HTTP（REST API）**：
- 数据查询、增删改操作（天然是请求-响应模式）
- 不需要实时性的场景（几秒钟延迟可以接受）
- 无状态的操作（每次请求独立）
- 需要缓存的场景（HTTP 缓存体系成熟）

**用 WebSocket**：
- 聊天室、私信系统（双向、实时）
- 实时游戏（状态同步、用户操作实时传输）
- 在线协作（共享文档、白板，多人同时编辑）
- 实时数据推送（股票行情、物流追踪、监控大屏）
- IoT 设备双向控制（设备上报 + 服务器下发指令）

**混合使用（最常见）**：

大多数应用是 HTTP + WebSocket 混合架构：
- HTTP：登录认证、获取历史数据、表单提交
- WebSocket：实时消息推送、在线状态

比如微信 Web 版：HTTP 获取会话列表、WebSocket 接收新消息推送。

---

###### 11. 【追问】在 Spring Boot 中如何实现 WebSocket？

Spring Boot 原生支持 WebSocket，两种方式：

**方式一：Spring WebSocket（基于 HTTP Session）**

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").withSockJS();  // SockJS 兼容降级
    }
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue");  // 消息代理
        registry.setApplicationDestinationPrefixes("/app");
    }
}

@Controller
public class ChatController {
    @MessageMapping("/chat")              // 客户端发往 /app/chat
    @SendTo("/topic/messages")            // 广播到订阅了 /topic/messages 的客户端
    public ChatMessage handleMessage(ChatMessage msg) {
        return msg;
    }
}
```

**方式二：javax WebSocket（更底层）**

```java
@ServerEndpoint("/ws/{userId}")
@Component
public class WebSocketServer {
    @OnOpen
    public void onOpen(Session session, @PathParam("userId") String userId) { }
    
    @OnMessage
    public void onMessage(String message, Session session) { }
    
    @OnClose
    public void onClose(Session session) { }
}
```

关联知识：[[35_NettyKnowledge/05_协议与实战应用]]（Netty 实现 WebSocket 服务器）
