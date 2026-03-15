### 四、Netty 启动流程

###### 1. 简单说说 Netty 服务端初始化并启动过程

Netty 服务端启动围绕 ServerBootstrap 展开，分为**配置、初始化、注册、绑定**四个阶段。

**第一步：创建并配置组件**
```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
 .channel(NioServerSocketChannel.class)
 .option(ChannelOption.SO_BACKLOG, 128)           // 给 ServerSocketChannel 配置
 .childOption(ChannelOption.TCP_NODELAY, true)     // 给子 Channel 配置
 .childHandler(new ChannelInitializer<SocketChannel>() {
     @Override
     public void initChannel(SocketChannel ch) {
         ch.pipeline().addLast(new MyServerHandler());
     }
 });
```

**第二步：调用 bind(port) 触发初始化**

- 通过反射创建 NioServerSocketChannel，构造函数中创建底层 JDK ServerSocketChannel、设置非阻塞模式、创建 Pipeline 和 Unsafe
- 调用 ServerBootstrap.init(channel)，关键动作是向 ServerSocketChannel 的 Pipeline 添加一个 **ServerBootstrapAcceptor**

**第三步：注册到 EventLoop**

调用 AbstractUnsafe.register()，将 ServerSocketChannel 注册到 bossGroup 的一个 EventLoop。底层 JDK SelectableChannel 注册到 Selector，触发 ChannelRegistered 事件。

**第四步：绑定端口**

注册完成后异步调用 doBind0()，最终调用 JDK 的 ServerSocketChannel.bind() 绑定端口。绑定成功后触发 ChannelActive 事件，同时向 Selector 注册 OP_ACCEPT 兴趣事件。

**启动完成**：bossGroup 的 EventLoop 开始轮询 ACCEPT 事件，等待客户端连接。

---

###### 2. 说说 Netty 的整体工作机制

Netty 整体工作机制基于 **Reactor 线程模型 + 事件驱动**，Channel、EventLoop、Pipeline、Handler 四者协同运转。

**整体流程**：

**1. 启动与监听**：ServerSocketChannel 注册到 bossGroup 的 EventLoop，监听 ACCEPT 事件。

**2. 事件循环（核心驱动）**：每个 EventLoop 在独立线程中运行，循环三件事：
```java
for (;;) {
    selector.select(timeout);       // 轮询 I/O 事件
    processSelectedKeys();          // 处理就绪事件
    runAllTasks();                  // 执行任务队列
}
```

**3. 连接接入**：bossGroup 轮询到 ACCEPT 事件，ServerBootstrapAcceptor 创建子 Channel，注册到 workerGroup。

**4. 数据读取（入站）**：workerGroup 轮询到 OP_READ，读取数据到 ByteBuf，触发 pipeline.fireChannelRead()，数据从 HeadContext 流经所有 InboundHandler（解码→业务处理）。

**5. 数据写出（出站）**：业务代码调用 ctx.write(msg)，若不是 EventLoop 线程则封装成任务入队；EventLoop 执行时触发 Pipeline 的 write 事件，数据从当前位置逆序流经 OutboundHandler（业务处理→编码），最终 HeadContext 写入发送缓冲区，flush() 真正发出。

**6. 内存管理**：数据以 ByteBuf 形式流动，PooledByteBufAllocator 高效分配，引用计数确保及时释放。

**7. 异步与 Future**：所有 I/O 操作返回 ChannelFuture，通过 Listener 或 sync() 获取结果。

---

###### 3. Netty 客户端启动流程是怎样的？

客户端比服务端简单，使用 Bootstrap，只需一个 EventLoopGroup。

**配置阶段**：
```java
EventLoopGroup group = new NioEventLoopGroup();
Bootstrap b = new Bootstrap();
b.group(group)
 .channel(NioSocketChannel.class)
 .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
 .handler(new ChannelInitializer<SocketChannel>() {
     @Override
     public void initChannel(SocketChannel ch) {
         ch.pipeline().addLast(new MyClientHandler());
     }
 });
```

**调用 connect(host, port) 触发流程**：

1. 通过反射创建 NioSocketChannel（底层创建 JDK SocketChannel、配置非阻塞、创建 Pipeline）
2. 调用 Bootstrap.init(channel)，向 Pipeline 添加配置的 ChannelInitializer（无 ServerBootstrapAcceptor）
3. 将 NioSocketChannel 注册到 group 的一个 EventLoop
4. 调用底层 SocketChannel.connect() 发起异步连接
   - 本地连接：立即成功，触发 ChannelActive
   - 远程连接：返回 false，向 Selector 注册 OP_CONNECT

**连接完成**：EventLoop 轮询到 OP_CONNECT 就绪，调用 finishConnect() 完成三次握手，触发 ChannelActive，注册 OP_READ 开始监听读事件。

整个 connect 是异步的，返回 ChannelFuture，可用 sync() 同步等待或 addListener 异步回调。

---

###### 4. Netty 如何处理新连接接入？

新连接接入由 bossGroup 和 ServerBootstrapAcceptor 协同完成，这是服务端的核心功能。

**完整处理流程**：

**第一步：事件触发**

bossGroup 的 EventLoop 轮询到 ServerSocketChannel 的 OP_ACCEPT 就绪，processSelectedKey() 调用 unsafe.read()。

**第二步：接受连接**

AbstractNioMessageChannel.NioMessageUnsafe.read() 被调用，循环调用 doReadMessages()，底层调用 ServerSocketChannel.accept() 获取 JDK SocketChannel，包装成 Netty NioSocketChannel 对象。

**第三步：传播事件**

read() 方法最后调用 pipeline.fireChannelRead(nioSocketChannel)，将新 Channel 作为消息传给 Pipeline。

**第四步：Acceptor 处理**（核心）

ServerBootstrapAcceptor.channelRead() 被调用：
```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;
    child.pipeline().addLast(childHandler);          // 添加业务 Handler
    setChannelOptions(child, childOptions, logger);   // 设置 TCP 参数
    // 将子 Channel 注册到 workerGroup
    childGroup.register(child).addListener(future -> {
        if (!future.isSuccess()) {
            forceClose(child, future.cause());
        }
    });
}
```

**第五步：子 Channel 初始化**

childHandler（ChannelInitializer）在 channelRegistered 时执行 initChannel()，向子 Channel 的 Pipeline 添加用户自定义的业务处理器。

**关键点**：
- **职责分离**：bossGroup 只负责 accept，最轻量；繁重的 I/O 处理交给 workerGroup
- **负载均衡**：workerGroup.next() 默认轮询，新连接均匀分配到各 EventLoop
- **全程异步**：从内核 accept 到 Netty 子 Channel 注册，没有任何阻塞点

---

###### 5. 【高频追问】Netty 服务端的 Pipeline 中默认有哪些 Handler？ServerBootstrapAcceptor 是什么时候加进去的？

**ServerSocketChannel 的 Pipeline**（bossGroup 使用的）：

初始状态：`HeadContext ↔ TailContext`（Pipeline 固定的头尾节点）

bind 时，ServerBootstrap.init() 方法会在 Pipeline 中添加一个 ChannelInitializer，这个 ChannelInitializer 在 channelRegistered 事件中被调用，执行完毕后：
1. 向 Pipeline 添加用户通过 `.handler()` 配置的 Handler（如果有）
2. 向 Pipeline 添加 **ServerBootstrapAcceptor**
3. ChannelInitializer 自身从 Pipeline 移除

最终 Pipeline 结构：`HeadContext ↔ [用户 handler（可选）] ↔ ServerBootstrapAcceptor ↔ TailContext`

**ServerBootstrapAcceptor 的作用**：是一个 ChannelInboundHandler，专门处理 channelRead 事件（即新连接接入事件）。它把每个新 SocketChannel 注册到 childGroup（workerGroup），并为其添加 childHandler。可以说它是 boss 和 worker 之间的**"连接交接员"**。

**子 SocketChannel 的 Pipeline**（workerGroup 使用的）：

初始：`HeadContext ↔ TailContext`

channelRegistered 时，ChannelInitializer 执行 initChannel()，添加用户配置的业务 Handler 链，然后自身移除。

最终结构因业务而异，典型如：`HeadContext ↔ LengthFieldBasedFrameDecoder ↔ CustomDecoder ↔ BusinessHandler ↔ CustomEncoder ↔ TailContext`
