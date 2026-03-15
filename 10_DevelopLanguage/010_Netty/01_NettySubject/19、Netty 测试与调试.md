# 十九、Netty 测试与调试

###### 1. 如何对 Netty 应用进行单元测试？

Netty 测试的核心工具是 **`EmbeddedChannel`**，它能在不启动真实网络的情况下，在内存中模拟完整的 Channel 环境，对 Handler 和编解码器进行单元测试。

**测试 ChannelHandler：**

```java
@Test
public void testChannelRead() {
    MyBusinessHandler handler = new MyBusinessHandler();
    EmbeddedChannel channel = new EmbeddedChannel(handler);

    // 写入入站数据（模拟收到网络数据）
    ByteBuf input = Unpooled.copiedBuffer("Test Data", CharsetUtil.UTF_8);
    assertTrue(channel.writeInbound(input));

    // 读取出站数据（模拟 Handler 处理后的响应）
    ByteBuf output = channel.readOutbound();
    assertNotNull(output);
    assertEquals("Processed: Test Data", output.toString(CharsetUtil.UTF_8));

    output.release();
    assertTrue(channel.finish()); // 检查是否所有消息都已处理，自动释放残留资源
}
```

**测试编解码器：**

```java
@Test
public void testCodec() {
    EmbeddedChannel channel = new EmbeddedChannel(
        new MyMessageToByteEncoder(),
        new MyByteToMessageDecoder()
    );

    MyProtocol request = new MyProtocol(1, "Hello");
    assertTrue(channel.writeOutbound(request));   // 触发编码
    ByteBuf encoded = channel.readOutbound();
    assertEquals(0xCAFEBABE, encoded.readInt());   // 验证魔数

    // 将编码结果写回，验证能否正确解码
    assertTrue(channel.writeInbound(encoded));
    MyProtocol decoded = channel.readInbound();
    assertEquals(request.getId(), decoded.getId());
    assertEquals(request.getBody(), decoded.getBody());
}
```

**测试异常处理：**

```java
@Test
public void testExceptionCaught() {
    MyBusinessHandler handler = new MyBusinessHandler();
    EmbeddedChannel channel = new EmbeddedChannel(handler);

    channel.pipeline().fireExceptionCaught(new RuntimeException("Test exception"));
    assertTrue(handler.exceptionCaught);   // 验证异常被处理
    assertFalse(channel.isActive());       // 验证 Channel 被关闭
}
```

**使用 Mockito 隔离外部依赖：**

```java
@Test
public void testHandlerWithService() {
    DatabaseService mockDb = Mockito.mock(DatabaseService.class);
    Mockito.when(mockDb.query(anyString())).thenReturn("MockResult");

    MyDataHandler handler = new MyDataHandler(mockDb);
    EmbeddedChannel channel = new EmbeddedChannel(handler);

    channel.writeInbound(new QueryCommand("select * from table"));
    Mockito.verify(mockDb).query("select * from table");  // 验证调用了外部服务
}
```

**测试空闲事件（IdleStateHandler）：**

```java
@Test
public void testIdleStateHandler() {
    IdleStateHandler idleHandler = new IdleStateHandler(1, 0, 0, TimeUnit.SECONDS);
    TestIdleEventHandler testHandler = new TestIdleEventHandler();
    EmbeddedChannel channel = new EmbeddedChannel(idleHandler, testHandler);

    channel.runScheduledPendingTasks(); // 推进调度任务，触发空闲检测
    assertTrue(testHandler.idleTriggered);
}
```

**最佳实践：**

- 每个测试方法创建独立的 `EmbeddedChannel` 实例
- 始终在测试末尾调用 `finish()` 或 `finishAndReleaseAll()`，检查残留消息并释放资源
- 将编解码逻辑、业务逻辑、异常处理分拆到独立的 Handler，便于单独测试

---

###### 2. EmbeddedChannel 的作用是什么？

`EmbeddedChannel` 是 Netty 专门为**单元测试**设计的 Channel 实现，让你在**不依赖真实网络**的情况下测试 Handler 和 Pipeline 逻辑。

**核心原理：**

`EmbeddedChannel` 继承 `AbstractChannel`，有完整的 `ChannelPipeline`、`ChannelConfig` 和一个特殊的 `EmbeddedEventLoop`。

`EmbeddedEventLoop` 的 `execute(Runnable task)` 方法是**同步执行**的（直接调用 `task.run()`），测试代码因此是线性的，不需要处理异步回调。

**内部维护两个队列：**

- `inboundMessages`：存放入站处理结果
- `outboundMessages`：存放出站处理结果

**核心 API：**

| 方法 | 作用 |
|------|------|
| `writeInbound(msg)` | 模拟收到网络数据，触发入站事件，处理结果放入 `inboundMessages` |
| `readInbound()` | 从 `inboundMessages` 读取一个消息，验证入站处理结果 |
| `writeOutbound(msg)` | 模拟应用写出数据，触发出站事件（编码等），结果放入 `outboundMessages` |
| `readOutbound()` | 从 `outboundMessages` 读取一个消息，验证编码/出站结果 |
| `finish()` | 检查两个队列是否为空（空=测试符合预期），自动释放资源 |
| `runScheduledPendingTasks()` | 执行所有调度任务（用于测试 IdleStateHandler 等定时逻辑） |

**适用场景：**

- 单元测试单个 ChannelHandler 逻辑
- 集成测试完整 Pipeline 流程
- 验证自定义编解码器的编解码正确性（包括粘包/拆包场景）
- 快速原型验证数据处理流程

**它是 Netty 中 TDD（测试驱动开发）的基石，使网络编程代码变得可测试。**

---

###### 3. 如何调试 Netty 应用？

Netty 的异步事件驱动模型给调试带来一定挑战，但有一套系统化的工具组合。

**1. 开启 Netty 内置日志（最直接）**

```xml
<!-- logback.xml -->
<logger name="io.netty" level="DEBUG" additivity="false">
    <appender-ref ref="STDOUT"/>
</logger>
```

- DEBUG 级别：Channel 生命周期、I/O 操作、Pipeline 事件
- TRACE 级别：ByteBuf 分配释放、EventLoop 调度（非常详细，仅定位棘手问题时开）
- **生产环境设为 WARN 或 ERROR**，大量 I/O 日志会严重拖慢性能

**2. 添加 LoggingHandler 到 Pipeline（数据流追踪神器）**

```java
pipeline.addFirst("logger", new LoggingHandler(LogLevel.DEBUG));
// 或指定 HEX_DUMP 格式，查看原始字节
pipeline.addFirst("logger", new LoggingHandler("MyApp", LogLevel.INFO, ByteBufFormat.HEX_DUMP));
```

能清晰看到数据在哪个 Handler 前后的状态变化，用 HEX_DUMP 格式能直观验证协议格式是否正确。**生产环境务必移除。**

**3. 开启内存泄漏检测（排查 OOM 神器）**

```bash
-Dio.netty.leakDetection.level=PARANOID   # 测试环境：每次分配都跟踪
-Dio.netty.leakDetection.level=ADVANCED   # 生产环境：采样检测
```

泄漏时日志输出访问堆栈，精确指向未释放代码位置。

**4. ChannelFuture 监听器诊断**

```java
channel.writeAndFlush(msg).addListener((ChannelFuture f) -> {
    if (!f.isSuccess()) {
        logger.error("写出失败，Channel 状态: isActive={}, isWritable={}", 
            f.channel().isActive(), f.channel().isWritable(), f.cause());
    }
});
```

**5. JVM 工具**

- **jstack**：检查 EventLoop 线程是否阻塞在某个方法（最常见问题：在 I/O 线程做了同步 DB 调用）
- **jmap + MAT**：分析堆快照，查看 ByteBuf 对象积累情况
- **Async-Profiler**：定位 CPU 和内存热点，找到真正的性能瓶颈

**6. 网络层调试**

- **Wireshark / tcpdump**：抓包验证协议格式是否正确，判断问题是应用层还是网络层
- **ss / netstat**：查看 TCP 连接状态，TIME_WAIT / CLOSE_WAIT 过多说明连接管理有问题

**常见问题排查思路：**

- **数据收不到/发不出**：Pipeline Handler 顺序是否正确？有没有 Handler 忘了调 `ctx.fireChannelRead(msg)` 往下传？用 `LoggingHandler` 看数据在哪个 Handler 消失
- **内存持续增长**：立即开启 PARANOID 级泄漏检测，看 `ResourceLeakDetector` 的日志
- **性能瓶颈**：用 Async-Profiler，检查是否在 I/O 线程执行了阻塞操作
- **连接异常断开**：监听 `exceptionCaught` 和 `channelInactive`，记录异常原因和关闭原因

---

###### 4. 如何使用 Netty 的日志记录功能？

Netty 内部基于 SLF4J 日志门面，可以无缝接入 Logback 或 Log4j2。

**1. 添加依赖**

```xml
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.86.Final</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.4.11</version>
</dependency>
```

**2. 配置 logback.xml（按模块精细控制）**

```xml
<configuration>
    <!-- Netty 内部通用：保持 WARN，只看重要信息 -->
    <logger name="io.netty.util.internal" level="WARN"/>
    <!-- 查看 Handler 级别的调试信息 -->
    <logger name="io.netty.handler" level="DEBUG"/>
    <!-- 查看内存分配（非常详细，谨慎开启） -->
    <logger name="io.netty.buffer" level="DEBUG"/>
    <!-- 泄漏检测日志 -->
    <logger name="io.netty.util.ResourceLeakDetector" level="INFO"/>
    <!-- 应用自己的日志 -->
    <logger name="com.yourcompany.netty" level="DEBUG"/>
    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

**3. 在 Handler 中正确使用日志**

```java
public class MyHandler extends ChannelInboundHandlerAdapter {
    private static final Logger logger = LoggerFactory.getLogger(MyHandler.class);
    
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 高频路径必须用 isDebugEnabled 判断，避免不必要的字符串拼接
        if (logger.isDebugEnabled()) {
            logger.debug("Received message from {}: {}", 
                ctx.channel().remoteAddress(), msg);
        }
        ctx.fireChannelRead(msg);
    }
    
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        logger.error("Exception on channel {}", ctx.channel().id(), cause);
        ctx.close();
    }
}
```

**4. 使用 LoggingHandler（调试神器）**

```java
pipeline.addLast(new LoggingHandler(LogLevel.DEBUG));
// 指定 ByteBuf 输出格式（HEX_DUMP 更直观）
pipeline.addLast(new LoggingHandler("MyApp", LogLevel.INFO, ByteBufFormat.HEX_DUMP));
```

**最佳实践总结：**

| 环境 | Netty 内部日志 | LoggingHandler | 泄漏检测 |
|------|------|------|------|
| 生产 | WARN/ERROR | 不加 | ADVANCED（采样） |
| 测试 | DEBUG | 加 | PARANOID（全追踪） |
| 开发 | DEBUG | 加 | ADVANCED |

**特别注意**：`LoggingHandler` 会打印数据内容，确保不会泄露敏感信息（密码、Token）。高频的 `channelRead` 路径里，日志输出前必须判断 `isDebugEnabled()`，否则字符串拼接开销不可忽视。

---

###### 5. 高频追问：如何用 EmbeddedChannel 测试粘包/拆包场景？

粘包/拆包是 Netty 解码器最容易出 Bug 的地方，`EmbeddedChannel` 非常适合在不同数据组合下做全面测试。

**测试思路：分批写入，验证能否正确组装**

```java
@Test
public void testPacketSplitting() {
    // 目标：验证 LengthFieldBasedFrameDecoder 能处理拆包场景
    EmbeddedChannel channel = new EmbeddedChannel(
        new LengthFieldBasedFrameDecoder(1024, 0, 4, 0, 4),
        new MyMessageDecoder()
    );
    
    // 构造一个完整消息的字节
    ByteBuf fullPacket = Unpooled.buffer();
    fullPacket.writeInt(5);           // 长度字段
    fullPacket.writeBytes("Hello".getBytes());
    
    // 模拟拆包：第一次只发前3字节
    channel.writeInbound(fullPacket.readRetainedSlice(3));
    assertNull(channel.readInbound()); // 数据不完整，应返回 null
    
    // 模拟拆包：第二次发剩余字节
    channel.writeInbound(fullPacket.readRetainedSlice(fullPacket.readableBytes()));
    MyMessage decoded = channel.readInbound();
    assertNotNull(decoded); // 现在应该能解出来了
    assertEquals("Hello", decoded.getContent());
}

@Test
public void testPacketMerging() {
    // 测试粘包：两个包粘在一起
    EmbeddedChannel channel = new EmbeddedChannel(
        new LengthFieldBasedFrameDecoder(1024, 0, 4, 0, 4),
        new MyMessageDecoder()
    );
    
    ByteBuf merged = Unpooled.buffer();
    // 第一个包
    merged.writeInt(5); merged.writeBytes("Hello".getBytes());
    // 第二个包紧跟在后面（粘包）
    merged.writeInt(5); merged.writeBytes("World".getBytes());
    
    channel.writeInbound(merged);
    
    MyMessage msg1 = channel.readInbound();
    MyMessage msg2 = channel.readInbound();
    assertEquals("Hello", msg1.getContent());
    assertEquals("World", msg2.getContent());
    assertNull(channel.readInbound()); // 没有多余的消息
}
```

**测试的边界场景：**

1. 单字节单字节地写（极端拆包）
2. 两个半包各一半
3. 三个包粘在一起
4. 含魔数验证失败时是否正确关闭连接
5. 数据长度字段超过最大限制时是否抛出异常
