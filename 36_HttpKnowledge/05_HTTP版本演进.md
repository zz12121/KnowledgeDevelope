# 05 HTTP 版本演进

## 5.1 版本对比总览

| 版本 | 发布 | 核心特性 | 底层协议 |
|------|------|---------|---------|
| HTTP/0.9 | 1991 | 只有 GET，只传 HTML | TCP |
| HTTP/1.0 | 1996 | 请求头/响应头/状态码/MIME | TCP（短连接） |
| HTTP/1.1 | 1997 | 持久连接/Host头/分块传输/缓存增强 | TCP（默认 Keep-Alive） |
| HTTP/2 | 2015 | 二进制分帧/多路复用/头部压缩/服务器推送 | TCP + TLS（实质必须）|
| HTTP/3 | 2022 | 彻底消除队头阻塞/0-RTT/连接迁移 | QUIC（UDP） |

---

## 5.2 HTTP/1.1 的性能瓶颈

1. **应用层队头阻塞**：持久连接内请求串行，一个慢请求阻塞后续所有请求
2. **多连接并发**：浏览器用每域名 6 个 TCP 连接绕过串行，但有握手开销
3. **Header 冗余**：每次请求重复发送大量 Header（User-Agent/Cookie 等），纯文本浪费带宽
4. **无服务器推送**：客户端必须主动请求，服务器无法预先推送

---

## 5.3 HTTP/2 核心机制

### 二进制分帧

HTTP/1.1 是文本协议，HTTP/2 把所有通信切分为二进制**帧（Frame）**：

```
+------+------+--------+
|Length| Type | Flags  |
+------+------+--------+
| Stream Identifier    |
+----------------------+
|   Frame Payload      |
+----------------------+
```

- **Stream（流）**：虚拟双向信道，每个请求/响应对占用独立 Stream（Stream ID 标识）
- **多路复用**：不同 Stream 的帧可以**交错传输**，接收端按 Stream ID 重组

### 多路复用

```
[HTTP/1.1 - 一条连接串行]
→ req1 → resp1 → req2 → resp2 ...

[HTTP/2 - 一条连接并发]
→ req1(S1) → req2(S2) → req3(S3)   // 同时发出
← resp2(S2) ← resp1(S1) ← resp3(S3) // 乱序返回，按 Stream ID 组装
```

**一条 TCP 连接搞定所有请求**，不需要每域名 6 个连接，节省大量握手开销。

### HPACK 头部压缩

**静态表**：内置 61 个常见 Header 键值对，用 1-2 字节索引表示（`2` = `method: GET`）

**动态表**：本次会话中出现过的 Header 会存入动态表，后续只发索引号

**效果**：Header 压缩率 85-95%，对 Cookie 密集的请求尤其显著

### 服务器推送（已基本废弃）

服务器可以主动推送资源，但实践中难以判断"推什么"，Chrome 已移除支持。
替代方案：`<link rel="preload">` 或 `103 Early Hints`。

---

## 5.4 HTTP/2 vs HTTP/1.1 关键对比

| 维度 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| 协议格式 | 文本 | 二进制 |
| 并发方式 | 多条 TCP 连接 | 单连接多路复用 |
| 队头阻塞 | 应用层有 | 解决应用层（TCP 层仍有） |
| Header | 每次全量重发 | HPACK 增量压缩 |
| 服务器推送 | 无 | 有（但实践中少用） |
| 加密 | 可选 | 主流浏览器要求 TLS |

---

## 5.5 HTTP/3 和 QUIC

### 为什么从 TCP 换 UDP？

TCP 协议栈在操作系统内核，改动极慢。QUIC 跑在用户态，迭代灵活。TCP 的有序字节流导致队头阻塞无法从根本上解决。

### QUIC 核心特性

| 特性 | 说明 |
|------|------|
| 基于 UDP | 自行实现可靠传输，但以 Stream 为单位 |
| 内置 TLS 1.3 | 1-RTT 握手，0-RTT 会话恢复 |
| 无队头阻塞 | 一个 Stream 丢包不影响其他 Stream |
| 连接迁移 | 连接 ID 标识，切换网络（Wi-Fi→4G）不断连 |
| 用户态实现 | 快速迭代拥塞控制算法 |

### 握手延迟对比

```
HTTP/1.1 + TLS 1.2: TCP(1.5RTT) + TLS(2RTT) = 3.5RTT
HTTP/2   + TLS 1.3: TCP(1.5RTT) + TLS(1RTT) = 2.5RTT
HTTP/3   + QUIC:    QUIC(1RTT)               = 1RTT
HTTP/3   + 0-RTT:   直接发数据               = 0RTT（重连）
```

---

## 5.6 队头阻塞（Head-of-Line Blocking）总结

| 层次 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|------|----------|--------|--------|
| 应用层 | **有**（请求串行） | **解决**（多路复用） | **解决** |
| TCP 层 | **有** | **有**（TCP 有序交付） | **解决**（QUIC 以 Stream 为单位） |

---

## 5.7 HTTP/2 实际部署

**Nginx 开启 HTTP/2**：
```nginx
server {
    listen 443 ssl http2;   # 一行搞定
    ssl_certificate cert.pem;
    ssl_certificate_key key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
}
```

**验证是否启用 HTTP/2**：
- Chrome DevTools → Network → Protocol 列看是否显示 `h2`
- curl：`curl -I --http2 https://example.com`

**HTTP/3 支持**：需要编译 `nginx-quic` 或使用支持 HTTP/3 的 CDN（Cloudflare 等）。
