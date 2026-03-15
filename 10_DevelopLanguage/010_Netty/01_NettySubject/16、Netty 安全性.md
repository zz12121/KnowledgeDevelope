# 十六、Netty 安全性

###### 1. 如何在 Netty 中实现 SSL/TLS 加密？

核心就是用 `SslHandler`，它是 Netty 把 JDK `SSLEngine` 适配到 Pipeline 事件驱动模型里的封装。

**完整实现步骤：**

**a. 构建 SslContext（线程安全，全局共享）**

```java
// 服务端：加载真实证书
SslContext sslCtx = SslContextBuilder
    .forServer(new File("server.crt"), new File("server.pkcs8"))
    .sslProvider(SslProvider.OPENSSL)        // 推荐 OpenSSL，比 JDK 实现快 2-3 倍
    .sessionCacheSize(1024 * 10)             // 缓存 Session，减少重复握手
    .sessionTimeout(3600)
    .build();

// 客户端：指定信任的 CA 证书
SslContext sslCtx = SslContextBuilder.forClient()
    .trustManager(new File("ca.crt"))
    .build();
```

**b. 在 Pipeline 中添加 SslHandler（必须 addFirst！）**

```java
@Override
protected void initChannel(SocketChannel ch) {
    SSLEngine engine = sslCtx.newEngine(ch.alloc());
    engine.setUseClientMode(false);
    engine.setNeedClientAuth(true);  // 开启双向认证
    engine.setEnabledProtocols(new String[]{"TLSv1.2", "TLSv1.3"});  // 禁用旧版本
    
    SslHandler sslHandler = new SslHandler(engine);
    sslHandler.setHandshakeTimeout(30, TimeUnit.SECONDS);
    
    ch.pipeline().addFirst("ssl", sslHandler);  // 第一个！先解密
    ch.pipeline().addLast(new StringDecoder());
    ch.pipeline().addLast(new MyBusinessHandler());
}
```

**c. 监听握手完成（可选但推荐）**

```java
// 方式一：监听 Future
sslHandler.handshakeFuture().addListener(f -> {
    if (f.isSuccess()) {
        log.info("SSL 握手成功，协议版本: {}", 
            sslHandler.engine().getSession().getProtocol());
    } else {
        log.error("SSL 握手失败", f.cause());
        f.channel().close();
    }
});
```

**高级配置：**

- **SNI 多域名**：根据客户端的 SNI 扩展选择对应域名的证书
- **ALPN 协商**：TLS 握手时协商应用层协议（h2 / http/1.1），实现 HTTP/2 支持
- **证书热更新**：自定义 `KeyManagerFactory`，支持不重启服务动态加载新证书

**性能优化：**

- 使用 `SslProvider.OPENSSL`，通过 JNI 调用原生 OpenSSL，性能显著优于 JDK 默认实现
- 启用 Session 缓存（`sessionCacheSize`/`sessionTimeout`），避免频繁重新握手
- 开启 Session Tickets，客户端复用 Session 时可跳过完整握手

---

###### 2. SslHandler 的作用是什么？

`SslHandler` 是 Netty 中 SSL/TLS 安全通信的核心组件，作用是在 Pipeline 中**透明地提供加解密、身份认证和完整性保护**，让业务 Handler 完全感觉不到加密层的存在。

**核心作用分解：**

**1. 协议适配**：把面向块的 `SSLEngine` API 适配成 Netty 面向流的 `ChannelHandler` API，处理所有 `SSLStatus`（BUFFER_OVERFLOW、BUFFER_UNDERFLOW、CLOSED 等）。

**2. 双向数据转换：**
- 入站：`ByteToMessageDecoder` 角色 → 调用 `SSLEngine.unwrap()` 解密密文，传给下游明文 ByteBuf
- 出站：`ChannelOutboundHandler` 角色 → 调用 `SSLEngine.wrap()` 加密明文，传给网络密文 ByteBuf

**3. 握手管理**：自动驱动完整的 TLS 握手流程（包括客户端认证），提供 `handshakeFuture()` 让业务方监听握手结果。

**4. Session 管理**：支持 Session 恢复和 Session Tickets，减少重复握手开销。

**5. 安全关闭**：发送 `close_notify` 警报，等待对端确认，防止截断攻击。

**源码核心逻辑（简化版）：**

解密流程（`decode` 方法）：
```
读取 TLS 记录头（5字节）→ 判断是否有完整记录 → 调用 SSLEngine.unwrap() → 
如果触发握手则发送握手数据 → 将明文传给下一个 Handler
```

加密流程（`write` 方法）：
```
拦截出站 ByteBuf → 放入 outboundUnencrypted 队列 → 
循环调用 SSLEngine.wrap() → 将密文放入 outboundEncrypted → 
ctx.writeAndFlush() 发送
```

**性能考量：**

- 默认使用堆外直接内存做加解密，避免额外内存拷贝
- 高并发建议用 `OpenSslEngine`，CPU 利用率明显降低
- 可通过 `setWrapDataSize()` 调整加密块大小，平衡延迟与吞吐

---

###### 3. 如何防止 Netty 中的 DDoS 攻击？

DDoS 防御没有银弹，需要网络层 + 应用层多层防护。Netty 应用层能做的主要是**连接管理、流量整形、资源限制**这三块。

**1. 连接层防护**

**限制连接总数和速率**（自定义 ConnectionLimitHandler）：

```java
b.option(ChannelOption.SO_BACKLOG, 1024)  // 全连接队列大小
 .childHandler(new ChannelInitializer<SocketChannel>() {
     protected void initChannel(SocketChannel ch) {
         ch.pipeline().addFirst(new ConnectionLimitHandler(maxConn, maxRatePerSec));
     }
 });
```

**IP 黑名单过滤**：

```java
public class IpFilterHandler extends ChannelInboundHandlerAdapter {
    private static final Set<String> BLACKLIST = ConcurrentHashMap.newKeySet();
    
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        String ip = ((InetSocketAddress) ctx.channel().remoteAddress())
                        .getAddress().getHostAddress();
        if (BLACKLIST.contains(ip)) {
            ctx.close();
            return;
        }
        ctx.fireChannelActive();
    }
}
```

**2. 流量层防护**

**全局流量整形**（令牌桶算法限速）：

```java
GlobalTrafficShapingHandler trafficShaping = new GlobalTrafficShapingHandler(
    eventLoopGroup,
    1024 * 1024,   // 写限制 1MB/s
    1024 * 1024,   // 读限制 1MB/s
    1000           // 检查间隔 1秒
);
pipeline.addLast("trafficShaping", trafficShaping);
```

**单连接数据包频率限制**：维护时间窗口内的包计数，超过阈值断开连接。

**3. 应用层防护**

**空闲连接快速释放**：

```java
pipeline.addLast(new IdleStateHandler(30, 0, 0, TimeUnit.SECONDS));
pipeline.addLast(new IdleEventHandler()); // 读空闲30秒则关闭连接
```

**请求速率限制**（基于 Guava RateLimiter）：

```java
public class RateLimitHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
    private final RateLimiter rateLimiter = RateLimiter.create(100); // 100请求/秒
    
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) {
        if (!rateLimiter.tryAcquire()) {
            sendTooManyRequests(ctx);
            return;
        }
        ctx.fireChannelRead(request);
    }
}
```

**SSL 握手超时**：`sslHandler.setHandshakeTimeout(10, TimeUnit.SECONDS)`，防止大量未完成握手占用资源。

**4. 资源与监控**

- `HttpObjectAggregator` 设置合理的 `maxContentLength`，防止超大请求体耗尽内存
- 监控 `ChannelOutboundBuffer` 积压情况，防止写缓冲区爆炸
- 通过 `GlobalEventExecutor` 定期上报关键指标：连接数、新建速率、流量、错误数

**5. 架构层防护**

- 在 Netty 前面部署 LVS/Nginx，用其连接限制、WAF、IP 黑名单做第一道过滤
- 使用云厂商 DDoS 高防 IP + 流量清洗服务
- 结合监控实现弹性扩容

---

###### 4. 如何实现 Netty 的访问控制？

访问控制的核心思路：在 Pipeline 的**协议解码器之后、业务逻辑之前**插入认证/授权 Handler。

```
[SSL] → [Protocol Decoder] → [Auth Handler] → [Business Logic] → [Protocol Encoder]
```

Auth Handler 接收的是已解码的结构化对象（如 `FullHttpRequest`），方便提取认证信息。

**1. Token 认证（最常见）**

```java
public class TokenAuthHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) {
        String auth = request.headers().get(HttpHeaderNames.AUTHORIZATION);
        if (auth == null || !auth.startsWith("Bearer ")) {
            sendError(ctx, HttpResponseStatus.UNAUTHORIZED);
            return;
        }
        String token = auth.substring(7);
        UserInfo user = authService.validateToken(token);
        if (user == null) {
            sendError(ctx, HttpResponseStatus.UNAUTHORIZED);
            return;
        }
        // 将用户信息存储到 Channel Attribute，后续 Handler 可以取用
        ctx.channel().attr(USER_KEY).set(user);
        ctx.fireChannelRead(request);
    }
}
```

**2. IP 白名单**

```java
public class IpWhitelistHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        String ip = ((InetSocketAddress) ctx.channel().remoteAddress())
                        .getAddress().getHostAddress();
        if (!isAllowed(ip)) {
            ctx.close();
            return;
        }
        ctx.fireChannelActive();
    }
}
```

**3. Basic Auth**

提取 `Authorization: Basic xxx`，Base64 解码得到 `username:password`，验证通过后放行。

**4. 双向 TLS 客户端证书认证**

```java
public class ClientCertAuthHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt instanceof SslHandshakeCompletionEvent && ((SslHandshakeCompletionEvent)evt).isSuccess()) {
            SslHandler sslHandler = ctx.pipeline().get(SslHandler.class);
            X509Certificate[] certs = sslHandler.engine().getSession().getPeerCertificateChain();
            if (!isValidClientCert(certs[0])) {
                ctx.close();
                return;
            }
        }
        ctx.fireUserEventTriggered(evt);
    }
}
```

**5. 授权（权限检查）**

认证通过后，从 Channel Attribute 取出用户信息，再校验该用户是否有权访问当前路径/方法。

**性能优化：**

- 本地 JWT 验证（验证签名，无需远程调用）
- Token 验证结果缓存（Caffeine Cache），设置合理的 TTL
- 避免在 EventLoop 线程中做远程认证调用（如必须，用异步方式并暂停 Pipeline 传播）

---

###### 5. 高频追问：双向 TLS（mTLS）和单向 TLS 的区别是什么？在 Netty 中如何配置？

**区别：**

单向 TLS（最常见）：只有服务端有证书，客户端验证服务端身份。客户端匿名，不提供证书。适用于大多数公开 API 场景。

双向 TLS（mTLS）：服务端和客户端都有证书，互相验证身份。适用于微服务内部通信、高安全性的 B2B 接口。

**Netty 配置双向 TLS：**

服务端开启客户端认证：

```java
SSLEngine engine = sslCtx.newEngine(ch.alloc());
engine.setUseClientMode(false);
engine.setNeedClientAuth(true);    // 强制要求客户端提供证书（否则握手失败）
// 或者：engine.setWantClientAuth(true);  // 希望客户端提供，但不强制
```

客户端携带自己的证书：

```java
SslContext sslCtx = SslContextBuilder.forClient()
    .keyManager(clientCertFile, clientKeyFile)   // 客户端证书和私钥
    .trustManager(serverCaCertFile)              // 信任服务端的 CA
    .build();
```

握手完成后，服务端从 SSLSession 中取到客户端证书，进行额外的业务层校验（如检查 CN、有效期、是否在黑名单）。

**记忆口诀**：单向 TLS = 服务端亮证件；双向 TLS = 双方都亮证件，互相认证。
