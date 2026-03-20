# 基于WebSocket协议的Netty开发

## 概述

WebSocket是一种在单个TCP连接上进行全双工通信的协议，解决了HTTP协议的半双工问题。Netty提供了完整的WebSocket支持，可以快速构建实时通信应用。

## WebSocket协议特点

1. **全双工通信**：客户端和服务端可同时发送数据
2. **持久连接**：一次握手，长连接
3. **服务端推送**：服务器可以主动向客户端推送数据

## 连接建立过程

### 握手阶段

1. 客户端发送HTTP请求，带有Upgrade头
2. 服务器返回101状态码（Switching Protocols）
3. 升级为WebSocket连接

### 数据传输阶段

- **数据帧**：Text（文本）、Binary（二进制）
- **控制帧**：Ping、Pong（心跳）、Close（关闭）

## Netty WebSocket开发

```java
public class WebSocketServer {
    public static void main(String[] args) {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ChannelPipeline pipeline = ch.pipeline();
                        // HTTP编解码（握手阶段需要）
                        pipeline.addLast("codec", new HttpServerCodec());
                        // 聚合HTTP消息
                        pipeline.addLast("aggregator", new HttpObjectAggregator(65536));
                        // WebSocket协议处理
                        pipeline.addLast("websocket", 
                            new WebSocketServerProtocolHandler("/ws"));
                        // 自定义业务处理器
                        pipeline.addLast("handler", new WebSocketFrameHandler());
                    }
                });
        
        bootstrap.bind(8080);
    }
}
```

### WebSocketFrameHandler

```java
public class WebSocketFrameHandler 
    extends SimpleChannelInboundHandler<WebSocketFrame> {
    
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, WebSocketFrame frame) {
        // 判断帧类型
        if (frame instanceof TextWebSocketFrame) {
            TextWebSocketFrame textFrame = (TextWebSocketFrame) frame;
            String content = textFrame.text();
            // 处理文本消息
            ctx.writeAndFlush(new TextWebSocketFrame("Echo: " + content));
        } else if (frame instanceof BinaryWebSocketFrame) {
            // 处理二进制消息
        }
    }
}
```

## 核心组件

| 组件 | 说明 |
|------|------|
| HttpServerCodec | HTTP编解码，握手阶段需要 |
| HttpObjectAggregator | 聚合HTTP消息 |
| WebSocketServerProtocolHandler | 处理WebSocket握手、控制帧 |
| TextWebSocketFrame | 文本帧 |
| BinaryWebSocketFrame | 二进制帧 |

## Pipeline配置顺序

```
HttpServerCodec 
    → HttpObjectAggregator 
        → WebSocketServerProtocolHandler 
            → WebSocketFrameHandler (业务处理)
```

## WebSocketServerProtocolHandler

核心功能：
- 自动处理WebSocket握手
- 处理Ping、Pong、Close控制帧
- 将Text/Binary帧传递给下游处理器

## 适用场景

1. **实时聊天应用**：IM系统、客服系统
2. **实时数据推送**：股票行情、新闻推送
3. **在线游戏**：实时游戏交互
4. **协同编辑**：文档多人协作

## 总结

Netty通过WebSocketServerProtocolHandler封装了复杂的协议细节，开发者只需关注业务逻辑，是构建实时通信应用的理想选择。
