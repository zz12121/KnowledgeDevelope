# 基于HTTP协议的Netty开发

## 概述

Netty内置了完整的HTTP协议支持，通过HttpServerCodec可以快速实现HTTP服务器和客户端开发。

## 核心组件

### HttpServerCodec

HttpServerCodec是HttpRequestDecoder和HttpResponseEncoder的组合：

```java
// 创建HttpServerCodec
HttpServerCodec codec = new HttpServerCodec();

// 添加到Pipeline
pipeline.addLast("codec", new HttpServerCodec());
pipeline.addLast("aggregator", new HttpObjectAggregator(65536));
pipeline.addLast("handler", new HttpRequestHandler());
```

### HttpObjectAggregator

用于聚合HTTP请求/响应的多个部分：

```java
// 聚合最大字节数
HttpObjectAggregator aggregator = new HttpObjectAggregator(65536);
```

## HTTP服务器开发

```java
public class HttpServer {
    public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ChannelPipeline pipeline = ch.pipeline();
                        // 添加HTTP编解码器
                        pipeline.addLast("codec", new HttpServerCodec());
                        // 聚合消息
                        pipeline.addLast("aggregator", new HttpObjectAggregator(65536));
                        // 添加业务处理器
                        pipeline.addLast("handler", new HttpRequestHandler());
                    }
                });
        
        ChannelFuture future = bootstrap.bind(8888).sync();
        future.channel().closeFuture().sync();
    }
}
```

### HttpRequestHandler

```java
public class HttpRequestHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest msg) {
        // 处理HTTP请求
        String uri = msg.uri();
        
        // 创建响应
        FullHttpResponse response = new DefaultFullHttpResponse(
            HTTP_1_1, OK, 
            Unpooled.copiedBuffer("Hello Netty".getBytes())
        );
        response.headers().set("Content-Type", "text/plain");
        
        ctx.writeAndFlush(response);
    }
}
```

## 核心类说明

| 类 | 说明 |
|------|------|
| HttpRequest | HTTP请求 |
| HttpResponse | HTTP响应 |
| FullHttpRequest | 完整HTTP请求（已聚合） |
| HttpContent | HTTP消息内容 |
| DefaultHttpResponse | 默认HTTP响应实现 |

## Pipeline中的顺序

```
ChannelPipeline:
    HttpServerCodec (Decoder + Encoder)
    HttpObjectAggregator (可选，聚合分段消息)
    HttpRequestHandler (业务处理)
```

## 适用场景

1. **HTTP服务器开发**：快速构建高性能HTTP服务
2. **REST API服务**：提供JSON格式的REST接口
3. **Web服务器**：静态文件服务
4. **微服务网关**：HTTP协议网关

## 总结

Netty通过HttpServerCodec提供了开箱即用的HTTP协议支持，配合Pipeline机制可以灵活处理HTTP请求，是构建高性能HTTP服务的理想选择。
