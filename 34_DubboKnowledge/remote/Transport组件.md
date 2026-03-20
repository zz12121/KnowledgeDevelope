# Transport组件

## 概述

Transport（传输层）是Dubbo网络通信的核心层次，负责将不同的网络框架（如Netty、Mina、Grizzly）抽象为统一接口，屏蔽底层差异。

## 核心组件

| 组件 | 说明 |
|------|------|
| **Transporter** | 统一的网络绑定（bind）和连接（connect）接口，通过SPI动态选择实现 |
| **Server** | 服务端抽象，负责监听端口、接收请求、返回响应 |
| **Client** | 客户端抽象，负责连接服务端、发送请求、接收响应 |
| **Channel** | 网络通道抽象，封装读写、开启/关闭等操作 |
| **ChannelHandler** | 处理网络事件（连接建立、断开、消息接收） |
| **Codec** | 编解码器，负责请求/响应的序列化与反序列化 |

## 核心接口

### Transporter

```java
public interface Transporter {
    // 服务端：绑定端口
    Server bind(URL url, ChannelHandler handler) throws RemotingException;
    
    // 客户端：连接服务端
    Client connect(URL url, ChannelHandler handler) throws RemotingException;
}
```

通过`@Adaptive`注解实现SPI动态选择，默认使用Netty4。

### Server/Client

- **Server**：服务端口监听，如`NettyServer`
- **Client**：客户端连接，如`NettyClient`

### ChannelHandler

```java
public interface ChannelHandler {
    void connected(Channel channel) throws RemotingException;
    void disconnected(Channel channel) throws RemotingException;
    void received(Channel channel, Object message) throws RemotingException;
    void caught(Channel channel, Throwable exception) throws RemotingException;
}
```

## 服务端启动流程

1. 从URL获取参数（端口、最大连接数等）
2. 调用`doOpen()`启动Netty服务
3. 创建boss/worker线程池
4. 配置ChannelPipeline（编解码器+业务处理器）
5. 绑定端口开始监听

## 客户端启动流程

1. 初始化重连参数
2. 调用`doOpen()`初始化客户端
3. 配置TCP参数（keepAlive、tcpNoDelay、connectTimeoutMillis）
4. 连接服务端，支持重连机制

## 设计原理

1. **SPI扩展机制**：通过`@Adaptive`注解实现动态适配
2. **统一通道模型**：无论Server/Client，均通过`Channel`收发消息
3. **线程模型分离**：Netty的boss/worker线程负责网络IO，业务处理交由Dubbo线程池
4. **编解码适配**：通过`NettyCodecAdapter`将Dubbo的`Codec2`适配到Netty的Pipeline

## 总结

Dubbo Transport层通过抽象接口+SPI扩展实现了网络通信的灵活切换，核心是围绕`Channel`和`ChannelHandler`构建事件驱动的网络模型。
