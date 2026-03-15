---
links: "[[34_DubboKnowledge]], [[30_RocketMQKnowledge]]"
---

# 十七、Netty 实战应用

###### 1. 如何使用 Netty 实现一个简单的 HTTP 服务器？

核心就三步：搭 ServerBootstrap、配 Pipeline、写业务 Handler。

**Pipeline 配置：**

```java
public class HttpServerInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    protected void initChannel(SocketChannel ch) {
        ch.pipeline()
          .addLast(new HttpServerCodec())          // HTTP 编解码
          .addLast(new HttpObjectAggregator(65536)) // 聚合成 FullHttpRequest
          .addLast(new HttpServerHandler());
    }
}
```

**业务处理器：**

```java
public class HttpServerHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) {
        String uri = request.uri();
        HttpMethod method = request.method();
        String requestBody = request.content().toString(CharsetUtil.UTF_8);
        
        FullHttpResponse response = new DefaultFullHttpResponse(
            HttpVersion.HTTP_1_1,
            HttpResponseStatus.OK,
            Unpooled.copiedBuffer("Hello from Netty HTTP Server", CharsetUtil.UTF_8));
        response.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/plain; charset=UTF-8");
        response.headers().set(HttpHeaderNames.CONTENT_LENGTH, response.content().readableBytes());
        
        if (HttpUtil.isKeepAlive(request)) {
            response.headers().set(HttpHeaderNames.CONNECTION, HttpHeaderValues.KEEP_ALIVE);
        }
        ctx.writeAndFlush(response);
        
        if (!HttpUtil.isKeepAlive(request)) {
            ctx.writeAndFlush(Unpooled.EMPTY_BUFFER).addListener(ChannelFutureListener.CLOSE);
        }
    }
    
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

**关键点：**

- `HttpServerCodec` = `HttpRequestDecoder` + `HttpResponseEncoder`，一个顶俩
- `HttpObjectAggregator` 的 `maxContentLength` 要合理设置，防止超大请求体撑爆内存
- `Content-Length` 头必须正确设置，否则客户端不知道消息体结束位置
- Keep-Alive 连接需要正确处理，不能在每次响应后都关闭连接

**启动代码：**

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
     .channel(NioServerSocketChannel.class)
     .childHandler(new HttpServerInitializer());
    ChannelFuture f = b.bind(8080).sync();
    f.channel().closeFuture().sync();
} finally {
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```

---

###### 2. 如何使用 Netty 实现一个 WebSocket 服务器？

WebSocket 服务器建在 HTTP 之上，初始化 Pipeline 先加 HTTP 支持，再加 `WebSocketServerProtocolHandler` 处理协议升级。

**Pipeline 配置：**

```java
public class WebSocketServerInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    protected void initChannel(SocketChannel ch) {
        ch.pipeline()
          .addLast(new HttpServerCodec())
          .addLast(new HttpObjectAggregator(65536))
          .addLast(new WebSocketServerProtocolHandler("/ws")) // 指定 WS 路径
          .addLast(new WebSocketServerHandler());
    }
}
```

**业务 Handler：**

```java
public class WebSocketServerHandler extends SimpleChannelInboundHandler<WebSocketFrame> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, WebSocketFrame frame) {
        if (frame instanceof TextWebSocketFrame) {
            String text = ((TextWebSocketFrame) frame).text();
            ctx.channel().writeAndFlush(new TextWebSocketFrame("Echo: " + text));
        } else if (frame instanceof PingWebSocketFrame) {
            ctx.channel().writeAndFlush(new PongWebSocketFrame(frame.content().retain()));
        } else if (frame instanceof CloseWebSocketFrame) {
            ctx.channel().close();
        }
    }
    
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt == WebSocketServerProtocolHandler.ServerHandshakeStateEvent.HANDSHAKE_COMPLETE) {
            // 握手成功，移除不再需要的 HTTP 聚合器
            ctx.pipeline().remove(HttpObjectAggregator.class);
            System.out.println("WebSocket 连接建立: " + ctx.channel().remoteAddress());
        } else {
            ctx.fireUserEventTriggered(evt);
        }
    }
    
    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        System.out.println("WebSocket 连接断开: " + ctx.channel().remoteAddress());
    }
}
```

**关键点：**

- `WebSocketServerProtocolHandler` 会自动处理握手、Ping/Pong，你只需关注业务帧
- 握手成功通过 `userEventTriggered` 通知，可以在这里做用户注册、权限验证
- IM 场景需要维护一个 `ConcurrentHashMap<userId, Channel>` 用于消息路由

---

###### 3. 如何使用 Netty 实现 RPC 框架？

Netty 作为 RPC 框架的传输层，核心是三部分：**协议设计 + 编解码 + 代理调用**。

**协议消息定义：**

```java
@Data
public class RpcMessage {
    private int magicCode = 0xCAFEBABE;  // 魔数
    private byte version = 1;
    private byte messageType;  // 0=请求, 1=响应
    private byte serializationType;  // 0=JSON, 1=Protobuf
    private int requestId;  // 请求 ID，用于匹配响应
    private int bodyLength;
    private Object body;  // RpcRequest 或 RpcResponse
}
```

**编解码器（核心在于处理粘包）：**

解码时，先读固定头部，验证魔数，再根据 `bodyLength` 判断当前 ByteBuf 是否有完整包；编码时，先写长度再写数据体。

**客户端实现要点：**

- 用 `ConcurrentHashMap<Integer, CompletableFuture<RpcResponse>>` 维护 requestId → Future 的映射
- 收到 Response 后，根据 requestId 找到对应的 Future，调用 `complete(response)` 唤醒等待线程
- 用 JDK 动态代理封装发送逻辑，对调用方完全透明

**服务端实现要点：**

- 收到 `RpcRequest`，通过反射找到本地服务实现，执行方法
- 将结果或异常封装成 `RpcResponse`，设置对应的 `requestId` 写回客户端

**设计要点总结：**

1. **协议必须包含长度字段**，交给 `LengthFieldBasedFrameDecoder` 预处理，解码器只需处理一个完整包
2. **requestId 是异步匹配的关键**，雪花算法或自增序列都可以
3. **序列化选 Protobuf 或 Kryo**，不要用 Java 原生序列化（太慢太大）
4. **超时处理**：客户端 Future 要设置超时，防止服务端挂掉导致调用方永久阻塞

---

###### 4. 如何使用 Netty 实现 IM 即时通讯系统？

IM 系统的核心挑战是：**海量长连接管理 + 消息实时推送 + 分布式路由**。

**1. 设计通信协议**

自定义二进制协议，消息类型包括：登录（LOGIN）、登出（LOGOUT）、单聊（CHAT）、群聊（GROUP_CHAT）、心跳（PING/PONG）、ACK。

协议头至少包含：魔数、版本、命令字、序列化方式、requestId、长度、数据体。

**2. 连接管理与认证**

```java
// 客户端连接后，第一条消息必须是登录请求
// 验证通过后绑定 userId 与 Channel
channel.attr(USER_ID_KEY).set(userId);
// 全局路由表：userId → Channel（单机版）
userChannelMap.put(userId, channel);
// 分布式版：userId → 服务器节点（存 Redis）
```

**3. 心跳保活**

```java
pipeline.addLast(new IdleStateHandler(150, 0, 0, TimeUnit.SECONDS));
// 150秒读空闲（客户端每30秒发心跳包），触发后关闭连接，清理路由表
```

**4. 消息收发与路由**

```java
// 服务端收到聊天消息，获取目标用户 Channel
Channel toChannel = userChannelMap.get(toUserId);
if (toChannel != null && toChannel.isActive()) {
    toChannel.writeAndFlush(chatMessage);  // 在线：直接推送
} else {
    messageStore.save(toUserId, chatMessage);  // 离线：落库，上线后推送
}
```

**5. 群聊广播**

```java
// 维护群组成员的 ChannelGroup
ChannelGroup groupChannels = channelGroupMap.get(groupId);
groupChannels.writeAndFlush(groupMessage);  // 自动广播给所有成员
```

**6. 消息可靠性（ACK 机制）**

重要消息客户端收到后回 ACK，服务端未收到 ACK 则重推；消息持久化到 DB 防止丢失。

**分布式扩展：**

- 用 Redis 存储 userId → 服务器节点的映射
- 收到消息，先查目标用户在哪台服务器，再通过 MQ 或直接 RPC 转发
- 与 `[[30_RocketMQKnowledge]]` 结合：消息先写 MQ，消费端推送，保证可靠性

**多线程操作 Channel 的安全保证：** 所有对 Channel 的写操作应通过 `channel.eventLoop().execute(() -> channel.writeAndFlush(msg))` 提交，确保在 Channel 绑定的 EventLoop 线程中执行，避免并发问题。

---

###### 5. 如何使用 Netty 实现文件传输？

两种主流方式：零拷贝大文件传输 + 分块自定义协议传输。

**方式一：零拷贝传输（大文件首选）**

```java
public void sendFile(Channel channel, File file) throws IOException {
    try (RandomAccessFile raf = new RandomAccessFile(file, "r")) {
        FileRegion region = new DefaultFileRegion(raf.getChannel(), 0, file.length());
        channel.writeAndFlush(region).addListener(future -> {
            if (!future.isSuccess()) {
                log.error("文件发送失败", future.cause());
            }
        });
    }
}
```

底层调用 Linux 的 `sendfile` 系统调用，文件数据直接从 PageCache 通过 DMA 拷贝到网卡缓冲区，全程不经过用户态，性能极高。

**注意**：`FileRegion` 只适合文件内容直接作为消息体的场景，如果前面还有自定义协议头，则需要结合 `CompositeByteBuf` 或改用方式二。

**方式二：分块传输（断点续传、进度通知）**

协议设计：`[魔数][类型][总块数][当前块索引][数据长度][数据]`

1. **发送端**：按 64KB 切块，每块构造协议对象发送，监听 `ChannelFuture` 控制发送速率（背压）
2. **接收端**：自定义 Decoder，根据块索引和总块数组装，收齐所有块后合并写文件

**关键点：**

- 必须用 `LengthFieldBasedFrameDecoder` 处理粘包，否则分块逻辑会混乱
- 大文件不要一次性加载进内存，用 `FileChannel.read(buffer, position)` 分批读取
- 零拷贝性能极高但灵活性差；分块方式可实现进度通知、断点续传，更通用

---

###### 6. Netty 在 Dubbo 中的应用

Dubbo 默认使用 Netty 4 作为底层 NIO 通信框架，相关代码在 `dubbo-remoting-netty4` 模块。

**网络模型：**

`NettyServer` 和 `NettyClient` 分别对应 `ServerBootstrap` 和 `Bootstrap`。Dubbo 在 Exchange 层抽象出 `NettyTransporter`，通过它创建 `Netty4Server` 和 `Netty4Client`，与 Netty API 直接交互。

**编解码：**

Dubbo 定义了自己的协议格式（16字节固定头部 + 业务数据）。`NettyCodecAdapter` 在 Pipeline 中添加 `InternalDecoder` 和 `InternalEncoder`，负责 Dubbo `Request`/`Response` 对象与字节流的相互转换。

**Handler 链：**

- **`NettyServerHandler`**：接收解码后的 Dubbo Request，调用 `DefaultExchangeHandler` 完成服务查找和反射调用
- **`NettyClientHandler`**：接收服务端返回的 Response，根据 requestId 找到对应的 `DefaultFuture`，设置结果触发异步回调

**线程模型调优：**

Dubbo 的 `Dispatcher` 配置（ALL/DIRECT/MESSAGE 等）控制是否将业务逻辑放到独立线程池，实现 I/O 线程与业务线程隔离，防止慢业务阻塞网络通信。

详见：`[[34_DubboKnowledge/03_通信与序列化]]`

---

###### 7. Netty 在 RocketMQ 中的应用

RocketMQ 的 NameServer、Broker、Producer、Consumer 之间的所有网络通信均基于 Netty 实现，相关代码在 `remoting` 模块。

**协议设计（RemotingCommand）：**

固定头部包含：code（请求码）、language（客户端语言）、version、opaque（requestId）、flag（标记位）、remark、extFields（扩展字段 HashMap），再加序列化后的消息体。

**编解码：**

- `NettyEncoder`：将 `RemotingCommand` 按格式写入 ByteBuf
- `NettyDecoder`：继承 `LengthFieldBasedFrameDecoder`，先按长度字段切帧解决粘包，再解析 `RemotingCommand`

**处理器：**

- **Broker 侧 `NettyServerHandler`**：根据 `RemotingCommand.code` 从 `ProcessorTable`（`HashMap<Integer, NettyRequestProcessor>`）找到对应处理器（如 `SendMessageProcessor`）异步执行，处理完写回响应
- **Client 侧 `NettyClientHandler`**：根据 `opaque`（requestId）找到对应的 `ResponseFuture`，唤醒等待线程或执行回调

**线程模型：**

不同类型请求有独立线程池（`publicExecutor`、`sendMessageExecutor`、`pullMessageExecutor`）。Netty I/O 线程只负责网络读写和编解码，解码后的任务快速分发到业务线程池，是保证高吞吐的关键设计。

详见：`[[30_RocketMQKnowledge]]`

---

###### 8. Netty 在 Elasticsearch 中的应用

ES 节点间通信（集群状态同步、数据复制）基于 Netty 的 `TcpTransport` 模块（`Netty4Transport`）。

**管道配置：**

核心业务处理器是 `MessageChannelHandler`，继承 `ByteToMessageDecoder`，负责根据长度字段读取完整报文，反序列化为 ES 内部消息对象（`InboundMessage`）。

**请求处理流程：**

1. `MessageChannelHandler` 收到解码后的 `InboundMessage`
2. 根据消息类型，调用 `TcpTransport.handleRequest` 或 `handleResponse`
3. 对于请求，根据 action 名称（如 `indices:data/write/bulk`）找到对应 `RequestHandler`，提交到对应业务线程池（bulk/search/management）
4. 对于响应，通过 requestId 找到 `TransportFuture` 完成异步回调

**线程模型亮点：**

ES 针对不同操作类型（bulk 写入、search 搜索、management 管理）配置了**完全独立**的业务线程池。Netty I/O 线程只做网络和编解码，解码后立即分发，避免耗时的搜索/索引操作阻塞网络线程，这是 ES 保证高吞吐的核心设计之一。

---

###### 9. 高频追问：Netty 实现 RPC 框架时，如何保证请求与响应的正确匹配？

关键是 **requestId + Future 映射表**，这是所有异步 RPC 框架的通用模式。

**实现原理：**

```java
// 客户端维护一个全局映射表
private final ConcurrentHashMap<Integer, CompletableFuture<RpcResponse>> pendingRequests 
    = new ConcurrentHashMap<>();

// 发送请求时
public CompletableFuture<RpcResponse> sendRequest(RpcRequest request) {
    int requestId = idGenerator.incrementAndGet();
    request.setRequestId(requestId);
    
    CompletableFuture<RpcResponse> future = new CompletableFuture<>();
    pendingRequests.put(requestId, future);  // 注册等待
    
    channel.writeAndFlush(request);
    return future;
}

// Handler 收到响应时
@Override
protected void channelRead0(ChannelHandlerContext ctx, RpcMessage msg) {
    if (msg.getMessageType() == RESPONSE) {
        RpcResponse response = (RpcResponse) msg.getBody();
        CompletableFuture<RpcResponse> future = pendingRequests.remove(response.getRequestId());
        if (future != null) {
            future.complete(response);  // 唤醒调用方
        }
    }
}
```

**requestId 生成方式：**

- 单机：AtomicInteger 自增，简单高效
- 分布式：雪花算法，保证全局唯一

**超时处理：**

```java
// 发送时设置超时
CompletableFuture<RpcResponse> future = new CompletableFuture<>();
pendingRequests.put(requestId, future);

// 超时后清理，防止内存泄漏
channel.eventLoop().schedule(() -> {
    CompletableFuture<RpcResponse> f = pendingRequests.remove(requestId);
    if (f != null) {
        f.completeExceptionally(new TimeoutException("RPC 调用超时"));
    }
}, timeout, TimeUnit.MILLISECONDS);
```

这个模式在 Dubbo（`DefaultFuture`）、RocketMQ（`ResponseFuture`）、Netty 自身的 `DefaultPromise` 中都有体现。
