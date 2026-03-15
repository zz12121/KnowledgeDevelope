# 四、HTTP 连接管理

###### 1. 什么是持久连接（Keep-Alive）？

持久连接是指：**一次 TCP 连接可以复用，处理多个 HTTP 请求，不用每次都重新建立连接**。

这是为了解决 HTTP/1.0 的短连接性能问题而引入的。建立 TCP 连接需要三次握手，断开需要四次挥手，如果每个请求都这样走一遍，性能开销非常大。

**HTTP/1.0**：默认短连接，每次请求完就关闭，想复用需要手动加 `Connection: keep-alive` 头。

**HTTP/1.1**：默认持久连接，不想复用才加 `Connection: close`。

持久连接的工作方式：
```
[一次 TCP 握手]
→ GET /index.html     → 200 OK
→ GET /style.css      → 200 OK
→ GET /script.js      → 200 OK
[一次 TCP 挥手]
```

相关参数（Nginx 配置示例）：
```nginx
keepalive_timeout 65;      # 连接空闲超时时间（秒）
keepalive_requests 1000;   # 单个连接最多处理的请求数
```

持久连接的限制：虽然复用了连接，但 HTTP/1.1 的请求还是串行的（一个请求收到响应后才发下一个），这就引出了"队头阻塞"问题，要到 HTTP/2 多路复用才能解决。

---

###### 2. HTTP/1.0 和 HTTP/1.1 在连接管理上有什么区别？

**HTTP/1.0**：
- 默认短连接，每次请求/响应后关闭 TCP
- 想复用连接需要显式加 `Connection: keep-alive`（非标准，浏览器扩展）
- 不支持管道化（Pipelining）

**HTTP/1.1**：
- 默认持久连接（Keep-Alive），显著减少连接建立/销毁的开销
- 支持管道化（Pipeline）——可以不等上一个响应就发下一个请求（但响应必须按顺序返回）
- 引入了 `Host` 头（必须携带），支持虚拟主机（一台服务器托管多个域名）
- 支持分块传输编码（Chunked Transfer Encoding），可以边生成边传输，不需要提前知道 Content-Length
- 新增了更多方法：PUT、DELETE、OPTIONS、TRACE 等

连接管理的核心升级就是**持久连接 + Host 头**，这两个改进奠定了 HTTP/1.1 20 多年统治地位的基础。

---

###### 3. 什么是 HTTP 管道化（Pipelining）？

管道化允许客户端**在收到上一个响应之前就发送下一个请求**，把请求堆叠发出去，减少等待时间。

没有管道化：
```
发 req1 → 等 resp1 → 发 req2 → 等 resp2
```

有管道化：
```
发 req1 → 发 req2 → 发 req3 → 等 resp1 → resp2 → resp3
```

理论上性能更好，但管道化有个致命缺陷：**响应必须按照请求的顺序返回**。如果 req1 处理很慢，req2、req3 的响应就算已经准备好了也要排队等 req1，这就是 **HTTP/1.1 的队头阻塞（Head-of-Line Blocking）**。

正因为这个问题，现代浏览器（Chrome、Firefox 等）都**默认禁用了管道化**，转而用多个 TCP 连接并发（一般同一域名 6 条连接）来提高并发性。

HTTP/2 的多路复用从根本上解决了这个问题——请求和响应都是独立的帧，可以乱序发送，不再有队头阻塞。

---

###### 4. HTTP 连接的超时机制是什么？

HTTP 连接超时分几个维度：

**连接超时（Connect Timeout）**：建立 TCP 连接的最大等待时间。超过这个时间还没完成三次握手，就报连接超时。通常设置 1-5 秒。

**读取超时（Read Timeout / Socket Timeout）**：连接建立后，等待服务器返回数据的最大时间。超过这个时间没有收到任何数据，就报读取超时。通常设置 10-60 秒，取决于接口处理时长。

**空闲超时（Idle Timeout / Keep-Alive Timeout）**：持久连接在没有请求的情况下，保持连接的最长时间。超时后服务器会主动关闭连接。Nginx 默认 65 秒。

**HTTP 响应超时**：从发出请求到收到完整响应的总时间限制。

Spring 中 RestTemplate / HttpClient 配置示例：
```java
RequestConfig config = RequestConfig.custom()
    .setConnectTimeout(3000)        // 连接超时 3s
    .setSocketTimeout(30000)        // 读取超时 30s
    .setConnectionRequestTimeout(1000) // 从连接池获取连接超时 1s
    .build();
```

服务端（Nginx）：
```nginx
proxy_connect_timeout 5s;
proxy_read_timeout 60s;
proxy_send_timeout 60s;
keepalive_timeout 65s;
```

---

###### 5. Connection 请求头的作用是什么？

`Connection` 头用于控制**当前连接的行为**，不会被代理转发（逐跳头，Hop-by-Hop Header）。

**常见取值**：

`Connection: keep-alive`：要求复用当前 TCP 连接（HTTP/1.0 时显式指定，HTTP/1.1 默认就是 keep-alive）。

`Connection: close`：请求完成后关闭连接（用于显式告知服务器这次完事就断）。

`Connection: Upgrade`：配合 `Upgrade` 头，请求协议升级。WebSocket 握手就是这么做的：
```
GET /ws HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
```

**逐跳头的含义**：`Connection` 头里列出的字段只在当前连接有效，代理不应该转发到下一个连接。比如 `Connection: keep-alive` 表示"我和代理之间保持连接"，不影响代理和下一个节点之间的连接。

---

###### 6. 【追问】在高并发场景下，如何优化 HTTP 连接管理？

**客户端侧**：
- **连接池**：复用 TCP 连接，避免频繁建立/销毁。Java 中 Apache HttpClient、OkHttp 都有连接池机制，设置合理的最大连接数和每 Host 最大连接数
- **升级 HTTP/2**：天然多路复用，一条连接并发处理多个请求，减少连接数
- **合理的超时配置**：避免因为超时设置不合理导致连接积压

**服务端侧（Nginx）**：
- `keepalive_timeout 65`：合理的空闲连接超时，太长浪费资源，太短失去复用意义
- `keepalive_requests 1000`：限制单连接最大请求数，防止单个连接独占太久
- `worker_connections`：每个 Worker 进程的最大连接数，结合 CPU 核数配置
- `upstream keepalive 32`：和上游服务（如 Spring Boot）保持长连接，避免每次请求都新建 TCP

**架构侧**：
- 合理设置 TCP 参数：`tcp_nopush`（减少小包发送次数）、`tcp_nodelay`（禁用 Nagle 算法，降低延迟）
- 使用长连接时注意 TIME_WAIT 积压问题，通过 `SO_REUSEADDR` 加快端口复用
