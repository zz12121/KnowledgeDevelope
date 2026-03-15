### 三、Netty 线程模型

> 关联知识库：[[34_DubboKnowledge/03_通信与序列化]] — Dubbo 在 Netty 线程模型基础上扩展了 Dispatcher 和业务线程池策略

###### 1. 说说 Netty 的线程模型

Netty 的线程模型是其高性能的核心，本质是**基于 Reactor 模式的主从多线程模型**，结合了"每个 Channel 绑定到单一 EventLoop 线程"的串行化设计。

**服务端标准模型**：
1. **Boss EventLoopGroup（主 Reactor）**：通常包含 1 个 EventLoop。专门监听端口、接受客户端连接请求。接到新连接后，将 SocketChannel **注册**到 Worker Group 的某个 EventLoop
2. **Worker EventLoopGroup（从 Reactor）**：包含多个 EventLoop（默认 CPU 核数 × 2）。每个 EventLoop 上注册多个已建立的 SocketChannel，负责处理这些连接的所有 I/O 事件和业务任务

**客户端模型**：通常只用一个 EventLoopGroup，既发起连接，也处理 I/O。

**三个核心设计原则**：
- 一个 EventLoop 对应一个线程（终身绑定）
- 一个 Channel 只注册到一个 EventLoop（终身绑定）
- 一个 EventLoop 可以服务多个 Channel（多路复用）

这种设计实现了**串行化处理，天然避免锁竞争**——Channel 上的所有操作都在同一个线程中顺序执行，ChannelHandler 里不需要加任何锁。

---

###### 2. Netty 的 Reactor 线程模型是什么？

Reactor 模式的核心是"事件分发器 + 事件处理器"，Netty 用 EventLoop 实现了这两者。

**三种 Reactor 变体**，Netty 都支持配置：

**单 Reactor 单线程**：一个 EventLoop 处理所有 Channel（包括 ServerSocketChannel 和 SocketChannel）的所有事件。连接建立和数据处理串行执行，生产环境基本不用。

```java
// 单 Reactor 单线程（不推荐）
EventLoopGroup group = new NioEventLoopGroup(1);
new ServerBootstrap().group(group, group)...
```

**单 Reactor 多线程**：只有一个 EventLoop 负责监听和事件分发，但耗时业务逻辑提交到独立的业务线程池执行。

**主从 Reactor 多线程（推荐）**：bossGroup 作为主 Reactor，专门处理 accept；workerGroup 作为从 Reactor 池，处理已建立连接的 I/O。有效分离了连接建立和数据读写两种不同负载特性的任务。

```java
// 主从 Reactor 多线程（标准配置）
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup(); // CPU*2
new ServerBootstrap().group(bossGroup, workerGroup)...
```

**EventLoop run() 核心循环**：
```java
for (;;) {
    // 1. 轮询 Selector 获取就绪 I/O 事件
    selector.select(timeoutMillis);
    // 2. 处理就绪的 I/O 事件
    processSelectedKeys();
    // 3. 处理任务队列中的所有任务
    runAllTasks();
}
```

---

###### 3. 默认情况 Netty 起多少线程？何时启动？

**线程数量**：
- **BossGroup**：`new NioEventLoopGroup()` 不传参时，默认 **1 个**线程。Accept 操作很轻，1 个足够
- **WorkerGroup**：默认 **CPU 核心数 × 2**。逻辑在 `MultithreadEventLoopGroup` 静态块中：`NettyRuntime.availableProcessors() * 2`。8 核机器默认 16 个 EventLoop

**启动时机（懒加载）**：线程不是 EventLoopGroup 创建时就启动，而是**懒加载**的：
1. 当有 Channel 需要注册到该 EventLoop 时（bossGroup 将新连接注册到 workerGroup 时触发）
2. 当有任务提交到该 EventLoop 执行时

启动逻辑在 `SingleThreadEventExecutor.startThread()` → `doStartThread()` 中，每个 EventLoop 的线程只启动一次，之后进入事件循环永不退出（直到关闭）。

---

###### 4. Netty 的 EventLoop 如何与线程绑定？

绑定关系是**"一个 EventLoop 实例对应一个唯一线程，且是终身绑定"**，在 EventLoop 首次执行任务时确立。

**源码流程**（SingleThreadEventExecutor）：

```java
private void doStartThread() {
    executor.execute(new Runnable() {  // executor 是 ThreadPerTaskExecutor，每次创建新线程
        @Override
        public void run() {
            thread = Thread.currentThread();  // 关键：绑定线程引用
            try {
                SingleThreadEventExecutor.this.run();  // 进入事件循环
            } finally {
                // 清理工作
            }
        }
    });
}
```

**线程绑定检查**：`inEventLoop(Thread thread)` 方法用于判断当前线程是否是绑定线程：
```java
public boolean inEventLoop(Thread thread) {
    return thread == this.thread;  // 直接比较引用
}
```

**任务提交机制**（保证线程安全的核心）：
```java
public void execute(Runnable task) {
    if (inEventLoop()) {
        addTask(task);     // 已经是 EventLoop 线程，直接加队列
    } else {
        startThread();     // 确保线程已启动
        addTask(task);     // 非绑定线程，提交到任务队列，等 EventLoop 线程执行
        // 可能唤醒 selector
    }
}
```

这保证了所有针对同一 Channel 的操作，无论从哪个线程发起，最终都在同一个 EventLoop 线程中顺序执行，没有并发问题。

---

###### 5. Netty 如何保证线程安全？

Netty 通过精巧的架构设计，将大部分并发控制**内部化**，提供近乎无锁的编程模型。

**1. 串行化执行（核心原则）**

每个 Channel 的所有 I/O 事件和操作，都在其绑定的唯一 EventLoop 线程中顺序执行。ChannelHandler 中的 channelRead 等方法可以确信没有其他线程并发操作这个 Channel。

**2. 跨线程调用的自动转发**

如果从业务线程池（非 EventLoop 线程）调用 ctx.write() 或 channel.write()，Netty 会自动将调用封装成 WriteTask，提交到该 Channel 绑定的 EventLoop 任务队列：

```java
// AbstractChannelHandlerContext.write() 核心逻辑
EventExecutor executor = next.executor();
if (executor.inEventLoop()) {
    invokeWrite(msg, flush, promise);  // 当前就是 EventLoop 线程，直接执行
} else {
    executor.execute(() -> invokeWrite(msg, flush, promise));  // 否则提交任务
}
```

**3. @Sharable Handler 的显式声明**

默认每个 Channel 有独立的 Handler 实例，天然线程安全。若要共享 Handler，必须用 @Sharable 标注，开发者需自行保证其线程安全。

**4. 高性能并发容器**

EventLoop 的任务队列使用 `MpscQueue`（多生产者单消费者队列，基于 JCTools 库），针对多线程提交、单线程消费的场景极致优化，入队操作几乎无锁。

---

###### 6. 【高频追问】如果在 ChannelHandler 中执行了数据库查询等阻塞操作，会有什么问题？如何解决？

**问题**：EventLoop 是单线程的，一个 Channel 的 I/O 阻塞会"冻结"整个 EventLoop 上所有 Channel 的事件处理，导致其他连接的读写全部延迟，吞吐量断崖式下降。

**解决方案**：将阻塞任务提交到独立的业务线程池执行，完成后再回到 EventLoop 线程写响应：

```java
// 在 ChannelHandler 中
private static final ExecutorService businessExecutor = 
    new ThreadPoolExecutor(16, 64, 60L, TimeUnit.SECONDS, new LinkedBlockingQueue<>(1000));

@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    businessExecutor.submit(() -> {
        try {
            Object result = doHeavyQuery(msg);   // 阻塞操作在业务线程执行
            ctx.writeAndFlush(result);            // 写回时 Netty 自动转发到 EventLoop 线程
        } catch (Exception e) {
            ctx.fireExceptionCaught(e);
        }
    });
}
```

**或者在 ServerBootstrap 中为 Handler 指定独立执行器**（Netty 4.x 支持）：
```java
pipeline.addLast(businessExecutor, "businessHandler", new MyBusinessHandler());
```
这样 MyBusinessHandler 的所有方法都在 businessExecutor 中执行，不占用 EventLoop 线程。
