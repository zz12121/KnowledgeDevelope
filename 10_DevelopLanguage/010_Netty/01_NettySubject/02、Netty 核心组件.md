### 二、Netty 核心组件

###### 1. Netty 的核心组件有哪些？简述其作用

Netty 的核心组件构成了其异步事件驱动架构的基石：

- **EventLoop（事件循环）**：核心执行引擎。本质是一个单线程执行器，内部绑定一个 Selector，不断轮询注册在其上的 Channel 的 I/O 事件，并执行提交给它的任务。一个 EventLoop 服务多个 Channel，但一个 Channel 终身只注册到一个 EventLoop，保证了线程安全。
- **Channel（通道）**：网络连接的抽象。代表一个可以进行 I/O 操作（读/写/连接/绑定）的实体，比 JDK 原生 Channel 功能更丰富，包括异步 I/O、关联的 Pipeline、连接状态等。
- **ChannelPipeline（流水线）**：由 ChannelHandler 组成的**双向责任链**。所有入站和出站事件都流经这条流水线，由链上的 Handler 依次处理，提供极高的可扩展性。
- **ChannelHandler（处理器）**：处理 I/O 事件或拦截 I/O 操作的逻辑单元。开发者通过实现 ChannelHandler 定义业务逻辑（编解码、日志、鉴权等）。
- **ByteBuf（字节缓冲区）**：Netty 的字节容器，替代 JDK ByteBuffer。读写索引分离、动态扩容、支持内存池和堆外内存，是高性能数据传输的基础。
- **Bootstrap / ServerBootstrap（引导类）**：Netty 应用的配置和启动入口，用于组装核心组件并绑定端口或发起连接。

---

###### 2. Netty 中有哪些重要组件？

除了上面说的六大核心组件，还有几个重要配角：

- **EventLoopGroup**：EventLoop 的集合，负责管理 EventLoop 的生命周期，并为新连接的 Channel 分配 EventLoop。服务端通常要两个：bossGroup 负责接受连接，workerGroup 负责处理 I/O。
- **ChannelFuture**：Netty 中所有 I/O 操作都是异步的，操作调用立即返回一个 ChannelFuture，通过它注册监听器或同步等待来获取结果。
- **ChannelHandlerContext**：Handler 和 Pipeline 之间的纽带。Handler 被添加到 Pipeline 时会创建一个关联的 Context，提供触发事件传播和执行 I/O 操作的方法。
- **ChannelOption 和 AttributeKey**：前者用于配置网络参数（如 SO_KEEPALIVE、TCP_NODELAY），后者用于在 Channel 上安全存储用户自定义状态。
- **Codec（编解码器）**：特殊的 ChannelHandler，实现协议编码和解码，如 StringEncoder/Decoder、HttpServerCodec 等。

---

###### 3. 说说 Netty 中网络通信的核心组件有哪些？

网络通信最直接相关的是 **Channel、EventLoop、ChannelPipeline 和 ChannelHandler** 的四件套协同：

1. **Channel** 是通信的端点实体，所有数据进出都通过它
2. **EventLoop** 是驱动通信的动力源，不断检查 Channel 是否有 I/O 事件就绪
3. **ChannelPipeline** 是处理骨架，**ChannelHandler** 是处理肌肉

数据从网络到达 Channel 时，触发入站事件，在 Pipeline 中从 Head 向 Tail 流经所有 InboundHandler（解码、业务处理）；向网络发送数据时，触发出站事件，从 Tail 向 Head 流经所有 OutboundHandler（编码、加密），最终由 HeadContext 写入网络。

---

###### 4. 什么是 Channel？它的作用是什么？

Channel 是 Netty 网络通信的**核心抽象**，对 Java NIO SelectableChannel、OIO Socket 等底层 API 的统一增强封装。

**主要作用**：
- **连接管理**：封装底层网络连接状态（活跃/打开/关闭）和元数据（本地/远程地址）
- **异步 I/O 操作**：提供 read/write/connect/bind/close 方法，所有操作返回 ChannelFuture
- **关联 Pipeline**：每个 Channel 拥有唯一的 ChannelPipeline，所有进出事件都通过它处理
- **绑定 EventLoop**：创建后注册到一个 EventLoop，终身不变，保证线程安全
- **配置与状态**：通过 ChannelOption 配置网络参数，通过 AttributeKey 存储会话状态

**源码角度**：`NioSocketChannel` 内部封装了 JDK 的 SocketChannel 和 SelectionKey，通过 AbstractNioChannel 等类将 NIO 事件桥接到 Netty 事件模型中。

---

###### 5. 什么是 ChannelHandler？它有哪些类型？

ChannelHandler 是处理 I/O 事件或拦截 I/O 操作的**业务逻辑载体**。

**两大类型，按事件流向划分**：

**ChannelInboundHandler（入站处理器）**：处理由外部触发的入站事件。
- 核心方法：channelActive（连接激活）、channelRead（数据可读）、channelReadComplete（读完成）、exceptionCaught（发生异常）、channelInactive（连接断开）
- 常用适配器：ChannelInboundHandlerAdapter，只覆盖需要的方法

**ChannelOutboundHandler（出站处理器）**：处理由应用程序主动触发的出站事件。
- 核心方法：bind（绑定地址）、connect（发起连接）、write（写数据）、flush（刷新）、close（关闭连接）
- 常用适配器：ChannelOutboundHandlerAdapter

**还有几个常用的特殊类型**：
- `SimpleChannelInboundHandler`：泛型化，会自动释放入站消息，强制指定消息类型，推荐在业务 Handler 中使用
- `ByteToMessageDecoder`：将字节解码为消息对象的基类
- `MessageToByteEncoder`：将消息对象编码为字节的基类
- 各种 Codec：如 HttpServerCodec，同时实现了 Inbound 和 Outbound

---

###### 6. Netty 中的 ChannelHandler 和 ChannelPipeline 的关系是什么？

**Handler 是处理单元，Pipeline 是容器和调度者**，类似插件与插槽总线的关系。

- **安装关系**：Handler 实例通过 ChannelInitializer 在 Channel 注册时安装到 Pipeline，形成处理链
- **事件传播**：Pipeline 负责在事件发生时，按既定顺序调用链上对应类型的 Handler（入站从 Head 到 Tail，出站从 Tail 到 Head）
- **上下文隔离**：Handler 不直接与 Pipeline 交互，而是通过 ChannelHandlerContext 这个中间人

典型处理流程：`网络数据 → Channel → 触发入站事件 → Pipeline 从 HeadContext 开始 → 依次调用各 InboundHandler.channelRead() → 每个 Handler 处理完调用 ctx.fireChannelRead() 传递到下一个 → 到达业务 Handler`

---

###### 7. 说说 ChannelPipeline 的作用和工作原理

**作用**：ChannelPipeline 是 Netty 的**事件处理编排框架**，为处理网络协议栈和业务逻辑提供高度模块化的机制。

**工作原理**：

**数据结构**：Pipeline 内部维护了一个由 ChannelHandlerContext 组成的**双向链表**。链表有固定的头节点（HeadContext）和尾节点（TailContext），分别负责与底层传输交互和未处理事件的终结。

**事件传播方向**：
- **Inbound 事件**：从 Head 向 Tail 传播，如 channelRead、channelActive
- **Outbound 事件**：从 Tail 向 Head 传播，如 write、connect

**触发机制**：
- 入站：底层数据可读时，HeadContext 读取数据，调用 fireChannelRead 传递给下一个 InboundHandler；每个 Handler 若希望事件继续，必须调用 ctx.fireChannelRead(msg)，否则传播链在此中断
- 出站：用户代码调用 channel.write(msg) 时，从 TailContext 开始，逆序传播到 HeadContext，HeadContext 调用底层 Unsafe.write() 写入发送缓冲区

**典型 Pipeline 组织**：`Head → LengthFieldBasedFrameDecoder（粘包处理）→ 自定义解码器 → 业务 Handler → 自定义编码器 → Tail`

---

###### 8. 说说 inbound 和 outbound 的区别

核心区别在于**事件或数据的流向**：

**Inbound（入站）**：
- 触发源：外部网络/系统底层（远程发来数据、建立了连接）
- 数据流向：流向应用程序，将字节解码成业务对象
- Pipeline 传播方向：**Head → Tail**
- 对应接口：ChannelInboundHandler
- 典型方法：channelActive、channelRead、exceptionCaught

**Outbound（出站）**：
- 触发源：应用程序自身（业务代码调用 write/connect/close）
- 数据流向：流向网络，将业务对象编码成字节
- Pipeline 传播方向：**Tail → Head**
- 对应接口：ChannelOutboundHandler
- 典型方法：write、flush、connect、close

**关键点**：一个 Handler 可以同时实现两个接口，编解码器就是典型例子。Handler 的添加顺序很重要：Inbound 必须先添加解码器才能处理业务；Outbound 必须先经过业务处理器再经过编码器。

---

###### 9. 什么是 ChannelHandlerContext？

ChannelHandlerContext 是 Handler 和 Pipeline 之间的**关联上下文**，每个添加到 Pipeline 的 Handler 都会创建一个与之绑定的唯一 Context 对象。

**主要作用**：

1. **信息存储与获取**：保存了 Handler 运行时依赖的环境，如关联的 Channel、EventLoop、Handler 实例本身
2. **事件传播控制**：提供 fireChannelRead()、fireChannelActive() 等方法，将事件传递到下一个 Handler
3. **I/O 操作入口**：提供 write()、flush()、connect()、close() 方法。**注意**：通过 ctx 调用的操作从**当前 Handler 位置**开始传播，而通过 channel 调用则从 Tail 开始，起点不同，提供了更精细的控制
4. **Handler 间通信**：通过 ctx.attr(KEY).set(value) 在同一 Channel 的不同 Handler 间安全传递数据

**源码角度**：ChannelHandlerContext 接口的默认实现是 DefaultChannelHandlerContext，内部持有 Channel、Pipeline、EventExecutor 和 Handler 本身的引用。

---

###### 10. EventLoopGroup 了解么？和 EventLoop 啥关系？

**EventLoopGroup 是 EventLoop 的集合**，负责管理一组 EventLoop 的生命周期，并为新创建的 Channel 分配 EventLoop。

**关系**：
- **管理关系**：EventLoopGroup 相当于线程池（虽然不只是线程池），EventLoop 就是池中的工作线程
- **分配关系**：当新 Channel 需要注册时，EventLoopGroup 通过 next() 方法（默认轮询策略）选择一个 EventLoop，此后该 Channel 的所有 I/O 事件都由这个 EventLoop 处理
- **继承关系**：EventLoop 接口**继承了** EventLoopGroup 接口。设计意图是让一个 EventLoop 可以看作只包含自身的特殊 Group，使 API 更灵活

**服务器端典型用法**：
```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);    // 接受连接
EventLoopGroup workerGroup = new NioEventLoopGroup();   // 处理 I/O
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
 .channel(NioServerSocketChannel.class);
```
bossGroup 的 EventLoop 轮询 ACCEPT 事件，接受新连接后，将子 Channel 注册到 workerGroup 的某个 EventLoop。

---

###### 11. 什么是 EventLoop？它的作用是什么？

EventLoop 是 Netty 的**核心执行器**，一个线程处理多个 Channel 的所有 I/O 事件和任务，是 Reactor 线程模型的核心体现。

**核心作用**：

1. **I/O 事件轮询与处理**：EventLoop 的 run() 方法是一个无限循环，循环做三件事：
   - **select**：轮询注册在其上所有 Channel 的 I/O 事件
   - **processSelectedKeys**：处理就绪的 I/O 事件，触发 Pipeline 的 channelRead 等事件传播
   - **runAllTasks**：执行任务队列中的普通任务和定时任务

2. **任务执行**：处理用户通过 eventLoop.execute(Runnable) 提交的异步任务

3. **线程绑定**：遵循"一个 Channel 一生只由一个 EventLoop 处理，一个 EventLoop 可以服务多个 Channel"的原则，从根本上避免多线程并发问题

**源码核心**（NioEventLoop.run 简化）：
```java
protected void run() {
    for (;;) {
        selector.select(timeoutMillis);     // 1. 轮询 I/O 事件
        processSelectedKeys();              // 2. 处理就绪事件
        runAllTasks(ioRatio);               // 3. 处理任务队列
    }
}
```

---

###### 12. Bootstrap 和 ServerBootstrap 了解么？

**Bootstrap** 和 **ServerBootstrap** 是 Netty 的**启动引导类**，简化了组件的配置和启动。

- **Bootstrap**：用于客户端，只需一个 EventLoopGroup
- **ServerBootstrap**：用于服务端，需要 bossGroup 和 workerGroup 两个 EventLoopGroup

**核心配置方法**：
- `.group()`：设置 EventLoopGroup
- `.channel()`：指定 Channel 类型（NioSocketChannel、NioServerSocketChannel）
- `.option()` / `.childOption()`：ServerBootstrap 中，option() 给 ServerSocketChannel 设参数（如 SO_BACKLOG），childOption() 给接收到的子 Channel 设参数（如 TCP_NODELAY）
- `.handler()` / `.childHandler()`：ServerBootstrap 中，handler() 给 bossGroup 用，childHandler() 给 workerGroup 用（这是业务流水线的主战场）

**启动**：
- 客户端：`Bootstrap.connect(host, port)` 发起连接
- 服务端：`ServerBootstrap.bind(port)` 绑定端口开始监听

**源码视角**：ServerBootstrap.init() 方法会向 ServerSocketChannel 的 Pipeline 添加一个 ServerBootstrapAcceptor。当接受新连接时，这个 Acceptor 的 channelRead() 会调用 childGroup.register(child)，把子 Channel 注册到 workerGroup。

---

###### 13. 什么是 ChannelFuture？它的作用是什么？

ChannelFuture 是 Netty 中**所有异步 I/O 操作的结果**。Netty 的 I/O 操作（write/connect/bind）全是非阻塞异步的，调用立即返回，结果通过 ChannelFuture 获取。

**核心作用**：
1. **状态查询**：isDone()、isSuccess()、isCancelled()、cause()
2. **同步等待**：sync() 阻塞到操作完成，失败时抛异常；await() 不抛异常
3. **异步回调（推荐）**：addListener() 注册监听器，操作完成时 Netty 回调 operationComplete()

```java
ChannelFuture future = channel.writeAndFlush(message);
future.addListener(f -> {
    if (f.isSuccess()) {
        System.out.println("发送成功");
    } else {
        f.cause().printStackTrace();
    }
});
```

**设计模式**：ChannelFuture 继承自 JDK Future，其实现 DefaultChannelPromise 还实现了 Promise 接口。Promise 是可写的 Future，Netty 内部执行完操作后调用 setSuccess() 或 setFailure() 标记完成状态，这是经典的 **Promise/Future 模式**。

---

###### 14. 说说 ByteBuf 有什么特点

ByteBuf 是 Netty 设计的**新一代字节容器**，相比 JDK ByteBuffer 有革命性提升。

1. **读写索引分离**：维护独立的 readerIndex 和 writerIndex，无需像 ByteBuffer 那样频繁 flip()/rewind()
   - 可读区域：`[readerIndex, writerIndex)`
   - 可写区域：`[writerIndex, capacity)`
   - 可丢弃区域：`[0, readerIndex)`，调用 discardReadBytes() 可回收

2. **动态自动扩容**：写入时空间不足会自动扩容，直到 maxCapacity，无需手动处理

3. **CompositeByteBuf**：将多个 ByteBuf 逻辑组合成一个视图，不发生内存拷贝，实现零拷贝聚合

4. **内存池化**：PooledByteBufAllocator 默认启用，从池中重用 ByteBuf，大幅减少 GC 压力

5. **堆内/堆外内存**：堆外 Direct Buffer 在网络传输时避免一次额外拷贝，Netty Socket 读写优先使用 Direct Buffer

6. **引用计数**：基于 ReferenceCounted 接口，retain() 增加计数，release() 减少计数，归零时释放到池。SimpleChannelInboundHandler 会自动释放入站消息

7. **丰富 API**：链式调用、搜索、比较、slice/duplicate/copy、十六进制输出等

---

###### 15. ByteBuf 和 NIO 的 ByteBuffer 有什么区别？

**API 易用性**：ByteBuf 读写索引分离，无需 flip()/rewind() 切换模式，支持链式调用；ByteBuffer 读写共用 position，需手动管理状态，极易出错。

**容量扩展**：ByteBuf 支持动态自动扩容；ByteBuffer 固定容量，需手动创建更大的 Buffer 并拷贝。

**内存管理**：ByteBuf 支持池化（PooledByteBufAllocator）和非池化；ByteBuffer 只有非池化，完全依赖 JVM GC。

**零拷贝支持**：ByteBuf 提供 CompositeByteBuf 实现逻辑零拷贝，slice()/duplicate() 共享底层数据；ByteBuffer 的 slice() 功能相对有限。

**引用计数**：ByteBuf 支持，精细控制内存释放，配合池化提升性能；ByteBuffer 不支持，完全靠 GC。

**核心优势**：ByteBuf 通过读写指针分离、动态扩容、内存池和引用计数，从根本上解决了 ByteBuffer 的易用性和性能瓶颈，是 Netty 高性能网络传输的基石。

---

###### 16. ByteBuf 的分类有哪些？

**按内存位置分**：
- **堆内缓冲区（Heap ByteBuf）**：数据在 JVM 堆内，分配快；但 Socket I/O 时需先拷贝到堆外，多一次拷贝。实现类：UnpooledHeapByteBuf、PooledHeapByteBuf
- **堆外缓冲区（Direct ByteBuf）**：数据在 OS 内存，I/O 时无需额外拷贝；但分配和回收成本高，不受 JVM GC 直接管理。实现类：UnpooledDirectByteBuf、PooledDirectByteBuf
- **复合缓冲区（CompositeByteBuf）**：多个 ByteBuf 的逻辑视图，不拷贝数据，专门用于零拷贝聚合

**按内存管理方式分**：
- **池化（Pooled）**：从预分配的内存池获取和释放，极大减少分配和 GC 压力，高性能场景默认选择，通过 PooledByteBufAllocator.DEFAULT 分配
- **非池化（Unpooled）**：每次创建新实例，用完等待 GC，简单但性能低，通过 Unpooled 工具类分配

**按访问方式分**：
- 可读写缓冲区（最常见）
- 只读缓冲区：通过 ByteBuf.asReadOnly() 创建，修改内容会抛 ReadOnlyBufferException，用于安全共享数据

---

###### 17. 【高频追问】@Sharable 注解的作用是什么？什么时候需要加？

默认情况下，每次 ChannelInitializer.initChannel() 都会创建新的 ChannelHandler 实例，每个 Channel 拥有独立的 Handler 状态，天然线程安全，无需考虑并发。

**@Sharable 的作用**：如果你想在多个 Channel 的 Pipeline 中共享**同一个 Handler 实例**（节省资源），必须用 @Sharable 注解标记该 Handler 类。Netty 以此作为"该 Handler 是线程安全的"声明。

**什么时候需要加**：
- Handler 是**无状态的**（没有实例变量），如日志 Handler、统计 Handler
- Handler 的状态是线程安全的（使用 ThreadLocal 或并发容器维护状态）

**什么时候不能加**：
- Handler 中有非线程安全的实例变量（如 StringBuilder、计数器），多个 Channel 并发访问会出现数据竞争

Netty 不会自动检查 @Sharable Handler 是否真的线程安全，**这个责任在开发者**。未加 @Sharable 的 Handler 实例被添加到多个 Pipeline 时，Netty 会在运行时抛出异常。
