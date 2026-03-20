# 基于Netty实现远程通信

## 概述

Netty是Dubbo默认的网络通信框架，提供了高性能、异步非阻塞的NIO通信能力。Dubbo通过封装Netty，实现了消费者与提供者之间的高效远程调用。

## 核心架构

Dubbo远程通信层结构：
- **Transporter层**：网络传输抽象（Netty、Mina的统一接口）
- **Exchange层**：请求响应模式封装
- **DubboProtocol层**：RPC协议处理

## 消费者端（NettyClient）

### 初始化流程

1. **创建代理对象**
   - 在`ReferenceBean.init()`中，通过`refprotocol.refer()`获取Invoker
   - 通过`proxyFactory.getProxy()`生成动态代理

2. **建立Netty连接**
   - 调用`DubboProtocol.getClients()`初始化NettyClient
   - 执行`NettyClient.doOpen()`方法配置Netty的Bootstrap

3. **Bootstrap配置**
   ```java
   // 设置TCP参数
   bootstrap.option(ChannelOption.SO_KEEPALIVE, true)
            .option(ChannelOption.TCP_NODELAY, true);
   
   // 添加编解码器
   pipeline.addLast("decoder", new NettyDecoder())
           .addLast("encoder", new NettyEncoder())
           .addLast("handler", new NettyClientHandler());
   ```

### 请求发送流程

1. 代理对象调用方法时，触发`DubboInvoker.invoke()`
2. 在`invoke`方法中，将方法调用封装为`RpcInvocation`
3. 调用`DubboInvoker.doInvoke()`
4. 根据异步/单向配置，调用`currentClient.send()`或`currentClient.request()`
5. 通过Netty的Channel将消息发送至Provider

## 提供者端（NettyServer）

### 启动流程

1. **服务导出**
   - Provider启动时，`ServiceBean`监听Spring上下文刷新事件
   - 调用`export()`方法导出服务

2. **创建NettyServer**
   - 在`DubboProtocol.createServer()`中实例化`NettyServer`
   - 执行`NettyServer.doOpen()`方法

3. **ServerBootstrap配置**
   ```java
   // 创建Boss/Worker线程池
   NioEventLoopGroup bossGroup = new NioEventLoopGroup();
   NioEventLoopGroup workerGroup = new NioEventLoopGroup();
   
   // 配置ServerBootstrap
   ServerBootstrap bootstrap = new ServerBootstrap();
   bootstrap.group(bossGroup, workerGroup)
           .channel(NioServerSocketChannel.class)
           .childHandler(new ChannelInitializer<SocketChannel>() {
               @Override
               protected void initChannel(SocketChannel ch) throws Exception {
                   ch.pipeline()
                       .addLast("decoder", new NettyDecoder())
                       .addLast("encoder", new NettyEncoder())
                       .addLast("handler", new NettyHandler());
               }
           });
   
   // 绑定端口
   bootstrap.bind(port).sync();
   ```

### 请求处理流程

1. NettyServer收到请求后，由`NettyHandler`处理连接、断开、消息接收等事件
2. 消息经解码后，交给`ExchangeHandler`执行本地服务调用
3. 将结果返回给Consumer

## 编解码机制

Dubbo自定义了编解码器处理RPC请求：

### 编码器（NettyEncoder）

```java
public class NettyEncoder extends MessageToByteEncoder {
    @Override
    protected void encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) throws Exception {
        // 使用DubboCodec进行编码
        DubboCodec codec = new DubboCodec();
        codec.encode(ctx, msg, out);
    }
}
```

### 解码器（NettyDecoder）

```java
public class NettyDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        // 使用DubboCodec进行解码
        DubboCodec codec = new DubboCodec();
        Object result = codec.decode(ctx, in);
        if (result != null) {
            out.add(result);
        }
    }
}
```

## 线程模型

1. **Boss线程**：处理Accept事件，负责Accept新连接
2. **Worker线程**：处理Read/Write事件，负责IO读写
3. **业务线程池**：执行业务逻辑，避免阻塞IO线程

## 核心组件

| 组件 | 作用 |
|------|------|
| NettyClient | 客户端网络通信 |
| NettyServer | 服务端网络通信 |
| NettyCodecAdapter | 编解码适配器 |
| NettyHandler | 消息处理 handler |

## 总结

Dubbo通过Netty实现了高性能的远程通信，封装了复杂的网络编程细节。消费者通过NettyClient发送请求，服务提供者通过NettyServer监听和处理请求，配合自定义的编解码器，实现了高效可靠的RPC通信。
