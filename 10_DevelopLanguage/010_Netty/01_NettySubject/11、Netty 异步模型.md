# 11、Netty 异步模型

---

## 1. Netty 的异步模型是怎样的？

Netty 的异步模型是高性能架构的基石，核心思想是：**任何耗时的操作都不应阻塞 I/O 线程，而是通过异步方式处理，结果通过 Future 或回调通知。**

**四大核心组件**：

**① 异步 I/O 操作**：所有网络操作（connect、bind、read、write、close）都设计为异步。调用后立即返回 `ChannelFuture`，操作在后台由 EventLoop 执行。

**② EventLoop（调度中心）**：每个 EventLoop 绑定一个线程，不断轮询两个队列：
- **任务队列**：用户提交的普通任务（Runnable）和定时任务
- **I/O 事件队列**：通过 Selector 轮询就绪的 I/O 事件

**③ Future/Promise 机制**：
- `ChannelFuture`：代表异步操作的只读结果，可以检查完成状态、等待完成、注册监听器
- `ChannelPromise`：可写的 Future，Netty 内部用它来设置操作的完成状态

**④ 回调与事件传播**：当事件（数据到达、连接建立）发生时，Netty 调用 Pipeline 中相应 Handler 的回调方法（`channelRead`、`channelActive`等）。

**一个完整的异步工作流**：
1. 客户端调 `bootstrap.connect(host, port)`，立即返回 ChannelFuture
2. EventLoop 处理 OP_CONNECT 事件完成连接
3. 连接成功后，Netty 调 Pipeline 的 `channelActive` 方法，并标记 ChannelFuture 为成功
4. 用户通过 `addListener()` 在连接成功后发登录请求
5. 整个过程 0 阻塞

---

## 2. Future 和 Promise 的区别是什么？

一句话区分：**Future 是给调用者用的（读），Promise 是给执行者用的（写）**。

**Future（消费者视角）**：
- 表示一个异步操作的**只读结果**
- 可以查询状态、等待完成、获取结果、注册监听器
- 不能修改操作结果
- 类比：**提货单**，你可以看货是否到了，但不能改变货物状态

**Promise（生产者视角）**：
- 继承自 Future，增加了**设置结果**的能力
- 可以调 `setSuccess()`、`setFailure()` 标记完成状态
- 通常由异步操作的执行者（框架内部）持有和设置
- 类比：**仓库管理员**，他创建提货单（Future）给你，货到时在提货单上盖章（setSuccess）

**Netty 中的继承关系**：
```
ChannelPromise extends ChannelFuture
ChannelPromise extends Promise<Void>
DefaultChannelPromise extends DefaultPromise<Void> implements ChannelPromise
```

**使用场景**：在 `AbstractChannelHandlerContext.write()` 里，Netty 创建一个 `DefaultChannelPromise` 传给底层，底层写操作完成时调 `promise.setSuccess()`，你在外层拿到的 `ChannelFuture` 就变成已完成状态，触发你注册的监听器。

---

## 3. 如何使用 ChannelFuture 进行异步编程？

有两种主要模式，**强烈推荐异步回调**。

**模式一：同步等待（谨慎使用）**：
```java
ChannelFuture future = channel.writeAndFlush(message);
future.sync(); // 阻塞当前线程，直到操作完成（可能抛异常）
// 或
future.await(); // 不抛异常，需手动检查 isSuccess()
if (future.isSuccess()) {
    System.out.println("发送成功");
} else {
    future.cause().printStackTrace();
}
```
⚠️ **警告**：绝对不要在 EventLoop 线程中调 `sync()`，这会死锁——EventLoop 等待自己完成操作。

**模式二：异步回调（推荐）**：
```java
channel.writeAndFlush(message).addListener((ChannelFutureListener) future -> {
    if (future.isSuccess()) {
        // 发送成功，可以发下一条
        sendNextMessage();
    } else {
        // 发送失败处理：重试、记录日志
        log.error("发送失败", future.cause());
        future.channel().close();
    }
});
```

**Netty 内置监听器**：
```java
// 操作失败时自动关闭连接
future.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
// 操作完成后关闭连接（无论成功失败）
future.addListener(ChannelFutureListener.CLOSE);
```

**链式异步操作**：
```java
bootstrap.connect(host, port).addListener(f -> {
    if (f.isSuccess()) {
        Channel channel = ((ChannelFuture) f).channel();
        channel.writeAndFlush(authMsg).addListener(f2 -> {
            if (f2.isSuccess()) startBusiness(channel);
            else channel.close();
        });
    } else {
        reconnect();
    }
});
```

---

## 4. Netty 中如何实现异步回调？

核心是 `GenericFutureListener` 接口和 `Future.addListener()` 方法。

**`GenericFutureListener`**：函数式接口，只有一个方法 `operationComplete(F future)`，当 Future 关联的操作完成时被调用。`ChannelFutureListener` 是其特化子接口。

**实现原理（源码剖析）**：
```java
// DefaultPromise 核心逻辑
public class DefaultPromise<V> extends AbstractFuture<V> implements Promise<V> {
    private Object listeners; // 存储监听器（单个或数组）
    private volatile Object result; // 操作结果（成功值或异常）

    @Override
    public Promise<V> addListener(GenericFutureListener listener) {
        synchronized (this) {
            addListener0(listener);
        }
        if (isDone()) {
            notifyListeners(); // 已完成则立即通知（防止在 addListener 前就完成的情况）
        }
        return this;
    }

    @Override
    public Promise<V> setSuccess(V result) {
        if (setSuccess0(result)) { // CAS 设置结果
            notifyListeners(); // 通知所有监听器
            return this;
        }
        throw new IllegalStateException("complete already");
    }

    private void notifyListeners() {
        EventExecutor executor = executor();
        if (executor.inEventLoop()) {
            notifyListenersNow(); // 在 EventLoop 线程直接执行
        } else {
            // 提交到 EventLoop 执行，保证监听器在正确线程中运行
            executor.execute(this::notifyListenersNow);
        }
    }
}
```

**在业务线程池中处理并异步写回**（重要模式）：
```java
public class BusinessHandler extends ChannelInboundHandlerAdapter {
    private final ExecutorService businessPool = Executors.newFixedThreadPool(10);

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 1. 提交耗时任务到业务线程池，释放 EventLoop
        businessPool.submit(() -> {
            Object result = doProcess(msg); // 如数据库查询、计算
            // 2. 处理完后，必须切回 EventLoop 执行写操作
            ctx.channel().eventLoop().execute(() -> {
                ctx.writeAndFlush(result);
            });
        });
    }
}
```

⚠️ 监听器代码在 EventLoop 线程中执行，必须轻量快速，不能有阻塞操作。

---

## 高频追问：如果在多个线程中写同一个 Channel，Netty 怎么保证线程安全？

这是 Netty 线程模型的核心设计之一，答案是：**通过任务队列序列化所有对 Channel 的操作**。

```java
// AbstractChannelHandlerContext.write() 的核心逻辑（简化版）
private void write(Object msg, boolean flush, ChannelPromise promise) {
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        // 当前已经在 EventLoop 线程，直接执行
        invokeWrite(msg, flush, promise);
    } else {
        // 不在 EventLoop 线程，封装成任务提交到 EventLoop 的任务队列
        executor.execute(() -> invokeWrite(msg, flush, promise));
    }
}
```

**关键点**：
1. 每个 Channel 绑定到唯一一个 EventLoop（且终身不变）
2. 所有对这个 Channel 的 I/O 操作，都必须在绑定的 EventLoop 线程中执行
3. 从非 EventLoop 线程调用 `channel.write()` 时，Netty 自动将操作封装成 Runnable 投入 EventLoop 的 **MpscQueue**（多生产者单消费者队列）
4. EventLoop 线程顺序消费队列中的任务，保证了 Channel 操作的串行化，无需外部加锁

这就是 Netty 的"**串行化无锁设计**"——通过队列而非锁来保证并发安全，既线程安全又高性能。

代价是：如果 EventLoop 任务队列积压太多，会增加延迟。这也是为什么要把耗时操作（数据库查询等）放到业务线程池，完成后再回 EventLoop 写出结果，而不是直接在 EventLoop 里做阻塞操作。
