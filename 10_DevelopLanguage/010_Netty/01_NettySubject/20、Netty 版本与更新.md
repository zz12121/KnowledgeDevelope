# 二十、Netty 版本与更新

###### 1. Netty 3.x 和 Netty 4.x 的主要区别是什么？

Netty 3.x → 4.x 是一次彻底重构，涉及线程模型、Pipeline 传播方向、内存管理、API 设计等核心层面。

**1. 线程模型：最核心的变化**

**Netty 3.x**：Boss-Worker 模型，但有严重缺陷。Boss 线程 accept 新连接后，**这条连接上的所有 I/O 事件仍由同一个 Boss 线程处理**。如果某个连接的 Handler 阻塞，会影响该 Boss 线程负责的所有其他连接。

**Netty 4.x**：引入了 `EventLoop` 模型，职责清晰分离：
- `BossGroup`：**只负责 accept 新连接**，建立后立即注册到 WorkerGroup
- `WorkerGroup`：**每个 Channel 终身绑定一个 EventLoop**，该 EventLoop 的线程处理这条连接的所有 I/O 事件

核心设计原则：**"一个线程处理多个连接，一个连接只由一个线程处理"**，实现了无锁化串行处理，既没有线程竞争，又保证了事件处理的顺序性。

**2. Pipeline 事件传播方向**

**Netty 3.x**：入站和出站事件统一从 Head 向 Tail 传播，方向一样，概念比较混乱。

**Netty 4.x**：双向传播，更符合数据流直觉：
- 入站事件（`channelRead`、`channelActive`）：Head → Tail
- 出站事件（`write`、`connect`）：Tail → Head

**3. 内存管理：ByteBuf 革命**

**Netty 3.x**：`ChannelBuffer` 基于 `byte[]`，依赖 JVM GC，碎片多、GC 压力大。

**Netty 4.x**：
- 引入全新的 `ByteBuf` 抽象，支持堆内存/直接内存，`readerIndex`/`writerIndex` 双指针设计更灵活
- 引入 `PooledByteBufAllocator`，借鉴 jemalloc 思想，PoolArena → PoolChunk → PoolSubpage 三级结构管理大块内存，极大减少内存碎片和 GC 压力
- 基于**引用计数**（`ReferenceCounted`接口）管理 ByteBuf 生命周期，需要手动管理或通过 `SimpleChannelInboundHandler` 自动释放

**4. API 设计更清晰**

- 包名从 `org.jboss.netty` 改为 `io.netty`
- `ChannelUpstreamHandler`/`ChannelDownstreamHandler` → 统一为 `ChannelInboundHandlerAdapter`/`ChannelOutboundHandlerAdapter`，入站/出站语义更明确
- `ChannelFuture` 扩展更完善，引入 `ChannelPromise`（可写的 Future），支持更丰富的监听器
- `@Sharable` 注解标记可在多个 Channel 间共享的 Handler

**5. 开箱即用的高级功能**

- `IdleStateHandler`：空闲连接检测
- `ChannelTrafficShapingHandler`：流量整形
- `EpollEventLoopGroup`：Linux epoll 本地传输，性能更高
- `ByteToMessageDecoder`、`MessageToMessageDecoder` 等基类，简化编解码器开发

**总结：Netty 4.x 通过重构线程模型解决了 3.x 的潜在阻塞问题，通过双向 Pipeline 和池化内存管理提供了更高的性能，通过更清晰的 API 降低了使用门槛。**

---

###### 2. Netty 4.x 和 Netty 5.x 的区别是什么？

**Netty 5.x 从未正式发布，且官方已永久放弃。**

**为什么被废弃：**

Netty 5 开发始于 2013 年，在经过数年开发后，2016 年左右官方正式宣布放弃：

1. **架构过于复杂**：为了追求极致异步，引入了大量新的抽象和概念，代码库臃肿难以维护
2. **迁移成本极高**：从 4.x 迁移到 5 需要重写大量代码，且收益不明确
3. **社区不买账**：大量用户对 Netty 4.x 非常满意，没有强烈的升级动机

**Netty 5 曾尝试的设计（了解即可）：**

- **完全异步化 API**：几乎所有阻塞点都被移除，所有操作返回 Future
- **使用 ForkJoinPool**：探索用 ForkJoinPool 替代 EventLoopGroup，更好利用多核
- **改进的 ByteBuf 内存管理**：更激进的内存池优化
- **进一步明确 Promise/Future 分离**：Future 只读，Promise 可写

**为什么选 Netty 4.x 更好：**

- **成熟稳定**：10+ 年生产验证，被 gRPC-Java、Elasticsearch、Dubbo、Apache Cassandra 等顶级项目使用
- **持续维护**：定期发布 4.1.x 版本（如 4.1.108.Final），修复漏洞、引入改进
- **生态丰富**：所有主流编解码器、协议实现都围绕 4.x 构建
- **心智负担低**：EventLoop 模型清晰易理解，相比 Netty 5 的完全异步模型更容易调试

**结论：Netty 5 是一个有趣但失败的技术探索，其部分思想被选择性地反向移植到了 Netty 4.x。对于所有项目，Netty 4.x 是唯一正确的选择。**

---

###### 3. 如何从 Netty 3.x 迁移到 Netty 4.x？

迁移涉及包名、启动代码、Handler API、内存管理等多个维度，需要系统性处理。

**第一步：更新依赖和包名**

```xml
<!-- 旧 -->
<dependency>
    <groupId>org.jboss.netty</groupId>
    <artifactId>netty</artifactId>
</dependency>

<!-- 新 -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.86.Final</version>
</dependency>
```

用 IDE 全局替换：`import org.jboss.netty.` → `import io.netty.`

**第二步：重构启动代码**

```java
// Netty 3.x
ServerBootstrap bootstrap = new ServerBootstrap(
    new NioServerSocketChannelFactory(
        Executors.newCachedThreadPool(), // boss
        Executors.newCachedThreadPool()  // worker
    )
);

// Netty 4.x
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
     .channel(NioServerSocketChannel.class)
     .childHandler(new YourChannelInitializer());
    ChannelFuture f = b.bind(port).sync();
    f.channel().closeFuture().sync();
} finally {
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```

**第三步：重构 Handler**

```java
// Netty 3.x（上行 + 下行分离）
public class OldHandler implements ChannelUpstreamHandler, ChannelDownstreamHandler {
    public void handleUpstream(ChannelHandlerContext ctx, ChannelEvent e) { }
    public void handleDownstream(ChannelHandlerContext ctx, ChannelEvent e) { }
}

// Netty 4.x（统一 Handler 基类）
public class NewHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 处理入站数据
        ctx.fireChannelRead(msg); // 传递给下一个 Handler
    }
    
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

**第四步：适配内存管理（最容易踩坑）**

```java
// Netty 3.x：ChannelBuffer
ChannelBuffer buffer = ChannelBuffers.dynamicBuffer();

// Netty 4.x：ByteBuf（必须注意引用计数！）
ByteBuf buffer = ctx.alloc().buffer();
// 用完后记得释放，否则内存泄漏
ReferenceCountUtil.release(buffer);

// 推荐做法：继承 SimpleChannelInboundHandler，父类自动释放
public class NewHandler extends SimpleChannelInboundHandler<ByteBuf> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) {
        // msg 在方法结束后自动 release
    }
}
```

**第五步：事件监听适配**

```java
// Netty 3.x
ChannelFuture future = channel.close();
future.addListener(new ChannelFutureListener() {
    public void operationComplete(ChannelFuture future) { }
});

// Netty 4.x（Lambda 风格）
channel.close().addListener(future -> {
    if (future.isSuccess()) {
        System.out.println("Channel closed");
    }
});
```

**第六步：Pipeline 配置改造**

```java
// Netty 3.x：在 ServerBootstrap.setPipelineFactory() 中配置
bootstrap.setPipelineFactory(() -> {
    ChannelPipeline pipeline = Channels.pipeline();
    pipeline.addLast("handler", new OldHandler());
    return pipeline;
});

// Netty 4.x：用 ChannelInitializer
b.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) {
        ch.pipeline().addLast(new NewHandler());
    }
});
```

**迁移注意事项：**

- **内存泄漏是最大风险**：迁移完成后立即开启 `-Dio.netty.leakDetection.level=PARANOID`，确保所有 ByteBuf 都被正确释放
- **Pipeline 事件方向变了**：原来出站也从 Head 到 Tail，现在出站从 Tail 到 Head，检查自定义 Handler 的传播逻辑
- **线程模型变了**：4.x 的 Handler 方法默认在 EventLoop 线程中执行，确保没有阻塞操作

---

###### 4. 高频追问：Netty 4.1.x 相比 4.0.x 有哪些重要改进？

Netty 4.1.x 是在 4.0.x 基础上的持续演进，核心改进集中在**性能优化、功能完善、问题修复**三个方向。

**主要改进：**

**性能优化：**

- **`FastThreadLocal` 成熟**：完全取代 JDK `ThreadLocal`，通过数组下标 O(1) 查找，减少哈希冲突和内存开销
- **`HashedWheelTimer` 优化**：心跳超时、连接空闲检测的调度性能大幅提升
- **`PooledByteBufAllocator` 改进**：内存池算法持续优化，减少内存碎片
- **内部线程数据结构**：大量从 `LinkedBlockingQueue` 换成 `MpscQueue`（多生产者单消费者），减少锁竞争

**新功能：**

- **HTTP/2 支持**：添加完整的 HTTP/2 帧处理支持（`Http2FrameCodec`、`Http2MultiplexHandler`）
- **`SslHandler` 改进**：支持 OpenSSL ALPN，可用于 HTTP/2 协议协商
- **`EpollEventLoopGroup` 增强**：更好的 epoll ET 模式支持，Linux 环境下性能更高
- **`ChannelGroup` 改进**：更好的批量操作支持
- **`ProxyHandler` 系列**：内置 SOCKS 和 HTTP CONNECT 代理支持

**问题修复：**

- epoll 空轮询 Bug 的修复在 4.0.x 末期 / 4.1.x 早期完成
- 各种 `ByteBuf` 内存泄漏边界场景的修复
- 压测场景下的线程安全 Bug 修复

**当前推荐版本：选最新的 4.1.x 稳定版**，享受所有历史 Bug 修复和性能优化，API 与 4.0.x 完全兼容。截至本文时，4.1.108.Final 是最新稳定版本。
