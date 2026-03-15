# 12、Netty 常见问题处理

---

## 1. Netty 是如何解决 JDK 中的 Selector BUG 的？

Netty 主要解决了 JDK NIO 中著名的 **epoll 空轮询 bug**：在 Linux 下，`Selector.select()` 可能在没有就绪事件时立即返回（返回值为 0），导致 EventLoop 不断空转，CPU 飙升到 100%。

**Netty 的解决方案：空轮询检测 + Selector 重建**

```java
// NioEventLoop.run() 的核心循环（简化）
int selectCnt = 0;
for (;;) {
    // 执行 select
    int selectedKeys = selector.select(timeoutMillis);
    selectCnt++;
    
    // 处理就绪事件、执行任务队列...
    
    // 检测是否发生空轮询
    if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 
            && selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
        // select 次数超过阈值（默认 512）且没处理任何事件，判定为空轮询
        rebuildSelector(); // 重建 Selector！
        selector = this.selector;
        selector.selectNow();
        selectCnt = 1;
        break;
    }
}
```

**重建 Selector 的过程**（`rebuildSelector()` 方法）：
1. 创建一个新的 Selector
2. 把旧 Selector 上注册的所有 SelectionKey 对应的 Channel，重新注册到新 Selector，保持原有 interestOps
3. 关闭旧 Selector

重建频率极低，短暂阻塞 EventLoop 的影响可忽略不计。

**可通过系统属性调整**：`io.netty.selectorAutoRebuildThreshold` 设置阈值，设为 0 可禁用自动重建。

---

## 2. Netty 如何解决 epoll 空轮询 bug？

上题讲的是 NIO 传输的解决方案。Netty 还提供了原生 epoll 传输（`EpollEventLoop`），从根本上规避了这个 bug。

**两层保障**：

**NIO 传输**：通过检测 `selectCnt` 超过阈值（默认 512）后重建 Selector 来解决 JDK 的 bug。

**原生 epoll 传输**（Linux 专有）：
- 直接使用 Linux 的 `epoll_wait` 系统调用，绕过 JDK Selector 的封装
- 由于直接调用系统调用，不受 JDK 这个 bug 影响
- 使用 `eventfd` 进行线程间唤醒，比 JDK 用 `pipe` 更高效
- `EpollEventLoop` 的 `run()` 方法依然包含类似的循环次数监控作为防御性编程

**如何启用原生 epoll**：
```java
// 替换 NioEventLoopGroup 为 EpollEventLoopGroup
EventLoopGroup bossGroup = new EpollEventLoopGroup(1);
EventLoopGroup workerGroup = new EpollEventLoopGroup();
bootstrap.channel(EpollServerSocketChannel.class); // 替换 NioServerSocketChannel
```

---

## 3. 什么是 Netty 的空轮询 bug？

**现象**：在 Linux 上，`Selector.select()` 在没有任何就绪 I/O 事件的情况下立即返回（返回 0），EventLoop 无实际工作可做，但程序以极高频率（每秒数百万次）循环调 `select()`，导致单个 CPU 核心占用率达到 100%。

**根本原因**：JDK 对 epoll 系统调用的封装存在缺陷，在某些边缘情况（如对端突然关闭连接，触发中断信号）下，`epoll_wait` 收到虚假的就绪通知，JDK 的 `Selector.select()` 不加判断地返回，但 `selectedKeys` 为空。

**影响**：
- 一个 EventLoop 线程占满一个 CPU 核心
- 有用的任务得不到及时调度，延迟增加
- 系统整体吞吐量下降

**Netty 的视角**：Netty 本身不是这个 bug 的根源，但作为高性能框架必须处理底层 JDK 的缺陷。Netty 的检测与重建机制成为框架健壮性的重要体现。

---

## 4. 如何处理 Netty 中的异常？

Netty 的异常处理遵循**责任链模式**，通过 Pipeline 传播，核心方法是 `ChannelHandler.exceptionCaught()`。

**入站异常传播**：
```java
public class BusinessHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        if (cause instanceof IOException) {
            // I/O 异常：记录日志，关闭连接
            log.error("IO异常，关闭连接: {}", cause.getMessage());
            ctx.close();
        } else if (cause instanceof BusinessException) {
            // 业务异常：发送错误响应，不关闭连接
            ctx.writeAndFlush(new ErrorResponse(cause.getMessage()));
        } else {
            // 其他异常：传递给下一个 Handler
            ctx.fireExceptionCaught(cause);
        }
    }
}
```

**出站异常处理**：通过 ChannelFuture 的监听器：
```java
channel.writeAndFlush(msg).addListener((ChannelFutureListener) future -> {
    if (!future.isSuccess()) {
        log.error("发送失败", future.cause());
        // 可以重试或关闭连接
        future.channel().close();
    }
});
```

**异常传播规则**：
- 如果 Handler 处理了异常，不调 `ctx.fireExceptionCaught()`，异常就终止在这里
- 如果未处理，调 `ctx.fireExceptionCaught()` 往后传
- 入站异常最终到达 `TailContext`，默认**记录 WARN 日志**（如果未被处理）
- 出站异常最终到达 `HeadContext`，默认**关闭连接**

**最佳实践**：
- 在 Pipeline 末端放一个全局兜底 Handler，统一处理所有未被处理的异常，记录日志+上报监控
- 区分异常类型：`IOException` 通常意味着网络问题，需关闭连接；业务异常则看情况
- `exceptionCaught` 在 EventLoop 线程中执行，处理必须轻量，不能阻塞
- 有 ByteBuf 的情况下确保在异常路径上也能释放内存

---

## 5. Netty 的优雅关闭是如何实现的？

优雅关闭的目标是：**处理完已有任务，释放所有资源，不造成数据丢失和客户端连接异常**。

**核心 API**：
```java
// 收到 SIGTERM 信号时调用
Future<?> bossFuture = bossGroup.shutdownGracefully(2, 15, TimeUnit.SECONDS);
Future<?> workerFuture = workerGroup.shutdownGracefully(2, 15, TimeUnit.SECONDS);
// 等待完全关闭
bossFuture.sync();
workerFuture.sync();
```

**`shutdownGracefully(quietPeriod, timeout, unit)` 的两个关键参数**：
- `quietPeriod`（默认 2 秒）：安静期。这段时间内如果没有新任务产生，就进入关闭阶段。如果有新任务，重新计时
- `timeout`（默认 15 秒）：总超时时间。即使安静期条件未满足，超时后强制关闭

**关闭流程**（源码 `confirmShutdown()` 方法）：
1. 状态从 `ST_STARTED` → `ST_SHUTTING_DOWN`，停止接受新任务
2. 执行任务队列和延迟队列中的剩余任务
3. 检查安静期：`quietPeriod` 内无新任务产生，进入下一步
4. 取消所有已调度的定时任务
5. 遍历所有注册的 Channel，逐个关闭
6. 关闭 Selector，释放底层资源
7. 线程终止，状态变为 `ST_TERMINATED`

**注意事项**：
- 长连接服务的 `quietPeriod` 应设置长一点，处理完可能的心跳和延迟消息
- 服务端关闭时，客户端会收到连接断开事件，客户端应有重连逻辑
- 确保所有 ByteBuf 池、业务线程池等外部资源也被正确关闭

---

## 6. 如何处理 Netty 中的半包读问题？

半包问题（也叫粘包/拆包）是 TCP 流式传输的固有问题，Netty 通过**解码器**（`ByteToMessageDecoder`）来解决，本质是根据应用层协议把字节流还原成完整的业务消息。

**四种内置解码器（选够用的那个）**：
1. `FixedLengthFrameDecoder`：固定长度，最简单
2. `LineBasedFrameDecoder`：以 `\n` 或 `\r\n` 分隔，适合文本协议
3. `DelimiterBasedFrameDecoder`：自定义分隔符
4. `LengthFieldBasedFrameDecoder`：**最通用**，基于长度字段

**自定义解码器示例**：
```java
public class MyDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        // 1. 检查头部是否完整到达
        if (in.readableBytes() < 2) return;
        
        in.markReaderIndex(); // 标记当前位置，数据不足时可回退
        short length = in.readShort(); // 读取长度字段（2字节）
        
        // 2. 检查数据体是否完整到达
        if (in.readableBytes() < length) {
            in.resetReaderIndex(); // 回退，等待更多数据到达
            return;
        }
        
        // 3. 数据完整，读取并构造业务对象
        byte[] data = new byte[length];
        in.readBytes(data);
        out.add(new MyMessage(data)); // 加入 out 传给下一个 Handler
    }
}
```

**ByteToMessageDecoder 的累积机制（源码要点）**：
- 内部维护一个 `ByteBuf` 类型的累加器 `cumulation`
- 每次 `channelRead` 触发时，把新读到的数据合并到 `cumulation`
- 调用子类的 `decode()` 方法，子类从 `cumulation` 里按协议格式解析
- 解析出完整消息后，`Netty` 将其加入 `out` 列表，逐个传给下一个 Handler
- 没有足够数据时，必须 `return`，Netty 保留未读数据等下次数据到来再解析
- 处理完后 Netty 自动压缩 `cumulation`，释放已读内存

**最佳实践**：在自定义解码器前加 `LengthFieldBasedFrameDecoder`，让它负责帧切割，你的 `decode()` 就可以放心地直接解析一个完整的帧，不用再处理数据不足的情况。

---

## 高频追问：exceptionCaught 在 Pipeline 中是怎么传播的？

`exceptionCaught` 是一个**入站事件**，从异常发生的 Handler 开始，向后（向 TailContext 方向）传播。

**触发时机**：
- 在 Handler 的入站方法（如 `channelRead`）中抛出未捕获异常时
- 在出站操作中发生异常时（通过 Promise 通知，也可能触发 exceptionCaught）

**传播规则**：
```
Handler A → Handler B（抛异常）→ exceptionCaught 从 B 开始传播
  → Handler C.exceptionCaught（如果调了 ctx.fireExceptionCaught）
  → Handler D.exceptionCaught
  → TailContext.exceptionCaught（默认：记录 WARN 日志）
```

**三种处理策略**：
1. **消化异常**（不调 `fireExceptionCaught`）：异常处理就此结束，后续 Handler 不感知
2. **部分处理后转发**：记录日志后调 `ctx.fireExceptionCaught(cause)` 继续传播
3. **不处理直接转发**：调 `super.exceptionCaught(ctx, cause)`（等同于 `ctx.fireExceptionCaught`）

**实际建议**：在 Pipeline 末端加一个全局兜底 Handler，捕获所有漏网的异常，统一记日志、上报告警和关闭连接。不要让异常静默地消失在 TailContext 的 WARN 日志里。
