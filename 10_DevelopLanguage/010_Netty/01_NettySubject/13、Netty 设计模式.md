# 13、Netty 设计模式

---

## 1. Netty 中使用了哪些设计模式？

Netty 是 GoF 设计模式的优秀实践范本，以下是最核心的几个（记忆口诀：**责工单观建适模对象迭策**）：

**① 责任链（Chain of Responsibility）**：最核心的模式。`ChannelPipeline` 和 `ChannelHandler` 构成处理流水线，事件在 Pipeline 中依次传递，每个 Handler 有机会处理。灵活增删 Handler 即可定制处理逻辑。

**② 工厂（Factory）**：`ChannelFactory` 创建 Channel（`ReflectiveChannelFactory` 用反射实例化）；`EventLoopGroup` 作为 EventLoop 的工厂；`ByteBufAllocator` 作为 ByteBuf 的分配工厂。

**③ 单例（Singleton）**：`GlobalEventExecutor.INSTANCE`、`PooledByteBufAllocator.DEFAULT`、`UnpooledByteBufAllocator.DEFAULT` 等全局共享实例，都是单例。

**④ 观察者（Observer）**：`Future`/`Promise` + `Listener` 机制。`ChannelFuture` 允许添加多个 `GenericFutureListener`，操作完成时所有监听器被通知。

**⑤ 建造者（Builder）**：`ServerBootstrap` 和 `Bootstrap` 是典型建造者，通过链式调用配置参数，最后 `bind()`/`connect()` 构建并启动 Channel。

**⑥ 适配器（Adapter）**：`ChannelInboundHandlerAdapter` 和 `ChannelOutboundHandlerAdapter` 提供 `ChannelHandler` 接口的默认空实现，用户只需覆盖感兴趣的方法。

**⑦ 模板方法（Template Method）**：`ByteToMessageDecoder` 定义了 `channelRead` 的骨架（累积字节、调用 `decode()`、传播结果），具体解码逻辑留给子类实现。

**⑧ 对象池（Object Pool）**：`PooledByteBufAllocator`（内存池）和 `Recycler`（对象池）大量使用，重用 ByteBuf、Entry 等对象，减少 GC 压力。

**⑨ 迭代器（Iterator）**：`CompositeByteBuf` 的 `iterator()` 方法遍历其中组合的多个 ByteBuf。

**⑩ 策略（Strategy）**：`EventLoopGroup.next()` 选择下一个 EventLoop 的策略（默认轮询）；`ByteBufAllocator` 定义内存分配策略（池化/非池化两种实现）。

---

## 2. 责任链模式在 Netty 中的应用

责任链是 Netty 最核心的设计模式，通过 `ChannelPipeline` + `ChannelHandler` 实现。

**模式结构**：
- **处理者接口**：`ChannelHandler`（入站 / 出站两类）
- **处理链**：`ChannelPipeline` 维护 `ChannelHandlerContext` 双向链表，每个节点包装一个 Handler

**事件传播方向**：
- **入站事件**（数据到达、连接建立等）：从 `HeadContext` → 各 `InboundHandler` → `TailContext`
- **出站事件**（写数据、关闭连接等）：从 `TailContext` → 各 `OutboundHandler` → `HeadContext`

**源码核心（DefaultChannelPipeline）**：
```java
// 触发入站 channelRead 事件
@Override
public ChannelPipeline fireChannelRead(Object msg) {
    AbstractChannelHandlerContext.invokeChannelRead(head, msg);
    return this;
}

// 在 Context 链中查找下一个 Inbound Handler，调用其 channelRead
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
    next.invokeChannelRead(msg); // 调用当前 Handler，Handler 再决定是否 fireChannelRead 给下一个
}
```

**Handler 如何决定传播**：
```java
public class LoggingHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        log.info("收到消息: {}", msg);
        ctx.fireChannelRead(msg); // 调用这个才会传给下一个 Handler；不调则责任链中断
    }
}
```

**优势**：解耦（复杂逻辑分解成小 Handler）、灵活（动态增删 Handler）、可扩展（加密/压缩/日志等功能随插随用）。

---

## 3. 单例模式在 Netty 中的应用

Netty 中的单例用于全局共享的、无状态的工具类或资源管理器，典型例子：

**`GlobalEventExecutor.INSTANCE`（饿汉式）**：
```java
public final class GlobalEventExecutor extends AbstractScheduledEventExecutor {
    public static final GlobalEventExecutor INSTANCE = new GlobalEventExecutor();
    private GlobalEventExecutor() { /* 私有构造器 */ }
}
```
用途：执行不需要绑定特定线程的后台任务，如 `DefaultChannelGroup` 用它管理所有 Channel 的关闭。

**`ByteBufAllocator` 的默认实例**：
- `PooledByteBufAllocator.DEFAULT`：静态常量，类加载时初始化，根据系统配置决定是否启用直接内存
- `UnpooledByteBufAllocator.DEFAULT`：同上

**设计考量**：
- 这些实例线程安全（要么无状态，要么内部做了并发控制）
- 避免重复创建开销，减少 GC 压力
- 提供全局统一的访问点，配置改一处，全局生效

---

## 4. 工厂模式在 Netty 中的应用

**`ChannelFactory` / `ReflectiveChannelFactory`（工厂方法）**：

Bootstrap 的 `channel(Class)` 方法内部创建一个 `ReflectiveChannelFactory`：
```java
public B channel(Class<? extends C> channelClass) {
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
}

// ReflectiveChannelFactory 用反射创建 Channel
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {
    private final Constructor<? extends T> constructor;
    
    @Override
    public T newChannel() {
        return constructor.newInstance(); // 反射创建 Channel 实例
    }
}
```

**`EventLoopGroup` 作为 EventLoop 的工厂**：
```java
// NioEventLoopGroup 的 newChild 方法（工厂方法模式）
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    return new NioEventLoop(this, executor, (SelectorProvider) args[0]);
}
```

**`ByteBufAllocator` 作为 ByteBuf 的工厂**：
```java
ByteBuf buffer = allocator.buffer(1024); // 根据实现不同，返回池化或非池化 ByteBuf
```

**`ChannelInitializer` 作为 Pipeline 配置的工厂**：虽然不是严格意义的工厂，但它在 Channel 注册时通过 `initChannel()` 为每个新 Channel 初始化 Pipeline，模板化了 Handler 的装配过程。

**优势**：隐藏创建细节（客户端不需要知道 NioEventLoop 还是 EpollEventLoop）；易于扩展（换 EventLoopGroup 类型就换了底层实现）。

---

## 5. 观察者模式在 Netty 中的应用

Netty 的异步编程模型大量使用观察者模式，通过 `Future`/`Promise` + `Listener` 实现。

**模式角色**：
- **主题（Subject）**：`ChannelFuture` / `ChannelPromise`，代表一个异步操作
- **观察者（Observer）**：`GenericFutureListener` / `ChannelFutureListener`，在操作完成时被通知

**`DefaultPromise` 核心源码**：
```java
public class DefaultPromise<V> extends AbstractFuture<V> implements Promise<V> {
    private Object listeners; // 监听器列表

    @Override
    public Promise<V> addListener(GenericFutureListener listener) {
        synchronized (this) { addListener0(listener); }
        if (isDone()) notifyListeners(); // 已完成则立即通知（避免竞争条件）
        return this;
    }

    @Override
    public Promise<V> setSuccess(V result) {
        if (setSuccess0(result)) { // CAS 设置结果
            notifyListeners(); // 通知所有注册的监听器
            return this;
        }
        throw new IllegalStateException("complete already: " + this);
    }
}
```

**与 JDK Future 的区别**：JDK `Future` 需要轮询 `isDone()` 来获取结果（同步拉取），Netty `Future` 通过观察者模式主动推送完成通知（异步回调），更高效，不阻塞线程。

---

## 高频追问：Netty 的 @Sharable 注解是什么？什么情况下可以使用？

`@Sharable` 是 Netty 提供的一个标注在 `ChannelHandler` 类上的注解，表示**这个 Handler 的实例可以被多个 Channel 的 Pipeline 共享使用**。

**为什么需要这个注解**：
默认情况下，Netty 要求每个 Channel 有自己独立的 Handler 实例，因为 Handler 通常有**状态**（比如解码器的 `cumulation` 缓冲区）。如果多个 Channel 共享同一个 Handler 实例，状态就会混乱。

`@Sharable` 告诉 Netty：**这个 Handler 是无状态的或者内部做了线程安全处理，可以安全地在多个 Channel 之间共享**。

**可以使用 @Sharable 的条件**：
1. **无状态**：Handler 内没有任何成员变量（或成员变量是不可变的）
2. **线程安全**：如果有状态，必须做了并发安全处理（如使用 `ConcurrentHashMap`、`AtomicLong` 等）

**典型示例**：
```java
// 可以 @Sharable：无状态
@ChannelHandler.Sharable
public class AuthCheckHandler extends SimpleChannelInboundHandler<HttpRequest> {
    private final AuthService authService; // 无状态的服务，本身线程安全
    
    public AuthCheckHandler(AuthService authService) {
        this.authService = authService;
    }
    
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, HttpRequest msg) {
        if (!authService.check(msg)) {
            ctx.close();
            return;
        }
        ctx.fireChannelRead(msg);
    }
}

// 不能 @Sharable：有状态（cumulation 缓冲区）
public class MyDecoder extends ByteToMessageDecoder {
    // ByteToMessageDecoder 内部有 ByteBuf cumulation 状态，每个 Channel 必须独占实例
    // Netty 会在运行时检查，如果 ByteToMessageDecoder 子类加了 @Sharable 会抛异常
}
```

**Netty 的运行时检查**：`ChannelPipeline.addLast()` 时，如果 Handler 没有 `@Sharable` 注解但被多次添加，Netty 会抛出 `ChannelPipelineException`。反之，`ByteToMessageDecoder` 等有状态的基类会在构造时检查是否有 `@Sharable`，有则抛异常防止误用。
