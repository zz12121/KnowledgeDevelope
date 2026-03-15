# 10、Netty 消息发送

---

## 1. Netty 发送消息有几种方式？

Netty 的所有发送操作都是异步非阻塞的，核心围绕 `Channel` 和 `ChannelHandlerContext`：

**通过 `Channel` 发送**（从 Pipeline 尾部 TailContext 开始传播）：
- `channel.write(msg)`：写入出站缓冲区，**不立即发送到网络**，等待 flush
- `channel.writeAndFlush(msg)`：写入缓冲区并**立即触发 flush**，最常用
- `channel.flush()`：手动触发，将缓冲区中已积累的数据刷到网络

**通过 `ChannelHandlerContext` 发送**（从当前 Handler 位置向前传播）：
- `ctx.write(msg)` 和 `ctx.writeAndFlush(msg)`：**关键区别**在于它从当前 Handler 在 Pipeline 中的位置开始，沿出站方向传播。当前 Handler 之后的 OutboundHandler 不会被调用，提供更精确的控制

**批量写操作**：多次 `write()` 后一次 `flush()`，减少系统调用次数，提高吞吐量。

所有这些方法都返回 `ChannelFuture`，用于异步获取操作结果或添加监听器。

---

## 2. write 和 writeAndFlush 的区别是什么？

核心区别：**`write` 是懒加载/延迟的，`writeAndFlush` 是立即执行的**。

**`write(msg)`**：
- 将消息添加到 `ChannelOutboundBuffer` 的尾部
- **不触发网络 I/O**，数据在缓冲区里等待
- 返回的 Future 表示"成功加入缓冲区"，不代表已发送
- 适合**批量发送**场景：积累多条消息后一次 flush，减少系统调用

**`writeAndFlush(msg)`**：
- 写入缓冲区后**立即触发 flush()**，尝试将数据写入 Socket
- 返回的 Future 表示"数据已被操作系统网络栈接受"（不代表到达对端）
- 适合**低延迟**场景，消息需要立即发出

**内部调用链**：
```
write()  → ... → HeadContext.write()  → Unsafe.write()（写入缓冲区）
writeAndFlush() → write() + flush() → ... → HeadContext.flush() → Unsafe.flush()（网络写入）
```

**最佳实践**：高吞吐场景用 `write()` + 最后一次 `flush()`；低延迟交互场景直接用 `writeAndFlush()`。

---

## 3. 如何处理 Netty 中的高并发写操作？

高并发写的核心挑战是**不阻塞 EventLoop 线程**，以及**防止写缓冲区积压导致 OOM**。

**1. 线程安全由 Netty 保证**：从多个业务线程并发调用 `channel.write()` 是安全的。Netty 会把非 EventLoop 线程的写操作封装成任务，放入 Channel 绑定的 EventLoop 任务队列中顺序执行。

**2. 监控写缓冲区水位线（关键）**：
```java
// 设置低水位 32KB，高水位 64KB
bootstrap.childOption(ChannelOption.WRITE_BUFFER_WATER_MARK, 
    new WriteBufferWaterMark(32 * 1024, 64 * 1024));

// Handler 中监听可写性变化
@Override
public void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception {
    if (!ctx.channel().isWritable()) {
        // 写缓冲超高水位，暂停读取，避免产生更多待发送数据
        ctx.channel().config().setAutoRead(false);
    } else {
        // 恢复读取
        ctx.channel().config().setAutoRead(true);
    }
    super.channelWritabilityChanged(ctx);
}
```

**3. 背压传播**：业务代码在调用 `write()` 前先检查 `channel.isWritable()`，不可写则暂停生产

**4. 用 `ChannelFuture` 监听器代替 `sync()`**：不要在 EventLoop 线程中同步等待写操作，应异步回调

**5. 批量写入与合并刷新**：高频小消息在应用层合并，减少系统调用

**6. 分离 I/O 线程与业务线程**：耗时的序列化、业务计算放在业务线程池，完成后提交给 EventLoop 执行写操作

**7. 流量整形**：用 `GlobalTrafficShapingHandler` 平滑发送速率。

---

## 4. Netty 的写缓冲区是如何工作的？

写缓冲区由 **`ChannelOutboundBuffer`** 实现，是 Channel 和底层网络之间的缓存层。

**内部结构**：
- `Entry` 对象（通过 `Recycler` 对象池复用）形成**单向链表**
- `unflushedEntry`：指向第一个未标记为 flush 的条目
- `flushedEntry`：指向第一个待刷新的条目
- `tailEntry`：链表尾部
- `totalPendingSize`：所有待发送消息的总字节数，用于水位线检查

**工作流程（三步走）**：

1. **`addMessage()`（write 操作）**：创建或复用 Entry，设置消息和 Promise，链到尾部，更新 `totalPendingSize`

2. **`addFlush()`（flush 操作）**：从 `unflushedEntry` 开始遍历到 `tailEntry`，把每个条目标记为 FLUSHED，移动 `flushedEntry` 指针

3. **`doWrite()`（实际网络写入）**：循环调用 `SocketChannel.write(buffer)`，写成功则调 `remove()` 移除已发送条目并通知 Promise 成功；Socket 缓冲区满则注册 `OP_WRITE` 兴趣退出循环，等待下次可写事件

**关键优化**：
- 写自旋优化：通过 `setWriteSpinCount()` 控制一次 flush 最多尝试 write 的次数，避免 Socket 满时的空转
- Entry 对象池化（Recycler）：减少 GC
- ByteBuf 引用计数：写入时 `retain()`，发送完成或失败后 `release()`，确保内存正确管理

**与 Selector 协同**：`doWrite` 发现 Socket 发送缓冲区满（write 返回 0）时，设置 `OP_WRITE` 兴趣，等 Socket 再次可写时重新 flush；数据发完后取消 `OP_WRITE`，避免 busy loop。

---

## 5. 如何控制 Netty 的写速度？

**1. Netty 内置流量整形处理器（最直接）**：
```java
// 全局限速：写速率限制 1MB/s
GlobalTrafficShapingHandler trafficHandler = 
    new GlobalTrafficShapingHandler(group, 1024 * 1024L, 0L);
serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) {
        ch.pipeline().addFirst(trafficHandler);
        // 添加其他 Handler
    }
});

// 单 Channel 限速
pipeline.addFirst(new ChannelTrafficShapingHandler(1024 * 1024L, 0L));
```

流量整形通过**延迟发送**控制速率：检测到超速时把数据放入延迟队列，稍后发送。

**2. 水位线 + 可写性监听**：如问题3所述，超高水位时暂停生产，降到低水位时恢复。

**3. 应用层限速**：用 Guava `RateLimiter` 或令牌桶算法控制调用 `writeAndFlush()` 的频率。

**4. 背压传播至源头**：
- Kafka Consumer 缓冲积压时暂停 `poll()`
- HTTP 服务器写缓冲不可写时暂停读请求体（`setAutoRead(false)`）

**5. 调整 TCP 参数**：
- `SO_SNDBUF`：发送缓冲区大小，较小则更快感知背压
- `TCP_NODELAY`：禁用 Nagle 算法，减少小包延迟，但可能增加网络拥塞

---

## 6. ChannelOutboundBuffer 的作用是什么？

`ChannelOutboundBuffer` 的核心价值可以归纳为六点：

1. **异步缓冲**：作为 `write()` 操作和实际网络 I/O 之间的缓冲层，让应用快速返回，不必等慢速的网络 I/O，实现异步非阻塞写入模型

2. **批量聚合**：多次 `write()` 一次 `flush()`，将多个小数据包聚合，减少系统调用次数，提升吞吐量

3. **流量控制与背压**：通过 `totalPendingSize` 与 `WRITE_BUFFER_WATER_MARK` 配合，提供背压信号，防止生产者速度超过消费者（网络）速度导致 OOM

4. **保证写入顺序**：作为 FIFO 队列，消息发送顺序与 `write()` 调用顺序一致

5. **资源管理**：Entry 对象通过 Recycler 池化复用；ByteBuf 引用计数精准管理，防内存泄漏

6. **进度跟踪**：每个 Entry 关联一个 `ChannelPromise`，发送完成或失败时精准通知调用方

关键方法：`addMessage()`（加消息）、`addFlush()`（标记待刷新）、`remove()`（移除已发送条目）、`current()`（获取当前待发送条目）。

---

## 高频追问：write() 和 writeAndFlush() 哪个更好？什么情况该用哪个？

没有绝对的"更好"，取决于你的场景：

**用 `write()` + 最后一次 `flush()` 的场景**：
- **批量推送**：比如 IM 服务器广播消息，一次性向 1000 个 Channel 发送同一条消息，先全部 `write()` 再 `flush()`，系统调用次数从 1000 次降到少得多
- **请求-响应对**：先 `write(response_header)` 再 `write(response_body)` 最后 `writeAndFlush(lastChunk)`，保证 header 和 body 在同一个 flush 里发出，减少 TCP 分包
- **高吞吐数据流**：实时数据推送、日志流，允许轻微延迟以换取更高吞吐

**直接用 `writeAndFlush()` 的场景**：
- **低延迟交互**：RPC 调用的响应必须立即返回
- **简单的一发一收**：大部分业务代码直接 `writeAndFlush()` 最直观，出错概率最小
- **不确定什么时候 flush**：如果你不确定什么时候是"最后一条消息"，就直接 `writeAndFlush()`，安全第一

**常见陷阱**：调了 `write()` 忘了 `flush()`，导致数据卡在缓冲区迟迟不发出，这是新人最常踩的坑。如果你不是在做明确的批量优化，默认用 `writeAndFlush()` 最安全。
