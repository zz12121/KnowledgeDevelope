# 核心组件与 Pipeline

## 1. ChannelPipeline 责任链

`ChannelPipeline` 是一个双向链表，存储了若干个 `ChannelHandlerContext`，每个 Context 包装一个 `ChannelHandler`。

**入站事件流向（Head → Tail）：**

```
网络数据 → HeadContext → Handler1(In) → Handler2(In) → TailContext
```

**出站事件流向（Tail → Head）：**

```
TailContext → Handler3(Out) → Handler2(Out) → HeadContext → 网络
```

**事件传播方法：**

- 入站：`ctx.fireChannelRead(msg)` / `ctx.fireChannelActive()` 等
- 出站：`ctx.write(msg)` / `ctx.writeAndFlush(msg)` 等

**⚠️ 常见陷阱**：

Handler 不调用 `ctx.fireChannelRead(msg)` → 事件不再传播，后续 Handler 收不到消息。这是调试 Netty 问题时最常见的坑之一。

---

## 2. ChannelHandler 分类

**`ChannelInboundHandlerAdapter`**：处理入站事件的基类，常用方法：
- `channelActive(ctx)` — 连接建立
- `channelRead(ctx, msg)` — 数据到达
- `channelInactive(ctx)` — 连接断开
- `exceptionCaught(ctx, cause)` — 异常处理
- `userEventTriggered(ctx, evt)` — 用户自定义事件（如 IdleStateEvent）

**`ChannelOutboundHandlerAdapter`**：处理出站操作的基类，常用方法：
- `write(ctx, msg, promise)` — 写数据
- `flush(ctx)` — 刷新
- `connect(ctx, addr, ...)` — 发起连接
- `close(ctx, promise)` — 关闭连接

**`SimpleChannelInboundHandler<T>`**：泛型入站处理器，**自动 release 入站消息**（`autoRelease=true`），适合大多数业务 Handler。

**`ChannelDuplexHandler`**：同时处理入站和出站，适合需要拦截双向流量的场景（如流量监控、认证）。

---

## 3. @Sharable 注解

标记了 `@Sharable` 的 Handler 可以被**多个 Channel 共享同一个实例**，添加到多个 Pipeline 中。

**使用条件（必须同时满足）：**

1. Handler 内部**没有实例级状态**（无成员变量，或成员变量是线程安全的）
2. 所有操作是**无状态的**，不依赖 `ChannelHandlerContext` 绑定的特定 Channel 数据

**不满足条件的危险后果**：多个连接并发访问同一个 Handler 实例，成员变量出现数据竞争，导致难以复现的并发 Bug。

**Netty 的运行时检查**：Netty 在 `addLast()` 时会检查 Handler 是否标注了 `@Sharable`，如果同一个 Handler 实例被尝试添加到第二个 Pipeline 而没有 `@Sharable`，会抛出异常。

**典型的可共享 Handler**：`LoggingHandler`、`StringEncoder`、`StringDecoder`（无状态）。

**典型的不可共享 Handler**：`ByteToMessageDecoder` 的子类（有累积缓冲区 `cumulation`，每个连接独立）。

---

## 4. ByteBuf 核心设计

ByteBuf 是 Netty 的字节容器，相比 JDK `ByteBuffer` 的最大改进是**双指针设计**：

```
+-------------------+------------------+------------------+
| discardable bytes |  readable bytes  |  writable bytes  |
+-------------------+------------------+------------------+
|                   |                  |                  |
0      <=      readerIndex   <=   writerIndex    <=    capacity
```

- `readerIndex`：读指针，`readXxx()` 方法自动推进
- `writerIndex`：写指针，`writeXxx()` 方法自动推进
- `markReaderIndex()` / `resetReaderIndex()`：标记/重置读位置（解码器处理半包的关键）

**内存类型：**

| 类型 | 特点 | 使用场景 |
|---|---|---|
| `HeapByteBuf` | 堆内存，受 GC 管理 | 小对象、短生命周期 |
| `DirectByteBuf` | 直接内存，不受 GC | 网络 I/O，零拷贝 |
| `CompositeByteBuf` | 多个 ByteBuf 的逻辑视图，无需复制 | 组合协议头和数据体 |

**引用计数（ReferenceCounted）：**

ByteBuf 基于引用计数管理内存。`retain()` 引用+1，`release()` 引用-1，减到 0 时回收。

原则：**谁最后用，谁负责 release。**

---

## 5. write vs writeAndFlush

`write(msg)`：将消息写入 `ChannelOutboundBuffer`（写缓冲区），**不立即发送**。

`flush()`：将 `ChannelOutboundBuffer` 中的数据刷到操作系统内核缓冲区，触发实际网络发送。

`writeAndFlush(msg)` = `write(msg)` + `flush()`，一步完成。

**使用策略：**

- **低延迟场景**（IM、RPC）：直接 `writeAndFlush`，立即发送
- **高吞吐场景**（批量推送）：多次 `write` 积累，最后一次 `flush`，减少系统调用次数

**⚠️ 常见陷阱**：只 `write` 不 `flush` → 消息堆积在缓冲区永远不发出去。

---

## 6. Future 与 Promise

**`ChannelFuture`**（只读视图）：代表异步操作的结果，可以检查完成状态、等待完成、添加监听器，但不能设置结果。

**`ChannelPromise`**（可写视图）：继承 `ChannelFuture`，可以调用 `setSuccess()` / `setFailure()` 设置结果。通常由 Netty 内部创建并在操作完成后设置，外部消费者持有 `ChannelFuture` 只读视图。

**推荐写法（非阻塞）：**

```java
channel.writeAndFlush(msg).addListener(future -> {
    if (!future.isSuccess()) {
        log.error("发送失败", future.cause());
    }
});
```

**不推荐写法（阻塞 EventLoop 线程）：**

```java
// 在 EventLoop 线程中调用 sync() 会造成死锁！
channel.writeAndFlush(msg).sync();
```

---

## 参考资料

> 本章节对应的原始参考资料和深入学习资源：

- **[Netty主要组件源码分析](./E:/md/1/Netty/Netty主要组件源码分析/)** — Channel/EventLoop/Pipeline 源码
- **[Netty技术细节源码分析](./E:/md/1/Netty/Netty技术细节源码分析/)** — 底层实现细节
- **[Netty编解码](./E:/md/1/Netty/Netty编解码/)** — 编解码器实现
- **[参考资料索引](../参考资料索引.md)** — 所有参考资料的总索引

---

## 7. 相关链接

- [[35_NettyKnowledge/01_核心架构与基础概念]]
- [[35_NettyKnowledge/03_高性能设计]]
- [[35_NettyKnowledge/04_线程模型与内存管理]]
