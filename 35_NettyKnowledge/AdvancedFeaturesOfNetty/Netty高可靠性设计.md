# Netty高可靠性设计

## 概述

在分布式系统和网络通信中，高可靠性是核心需求。Netty提供了完善的心跳检测和断线重连机制，确保长连接的稳定性。

## 心跳机制

### 什么是心跳？

心跳是TCP长连接中，客户端和服务器定期发送的特殊数据包，用于确认对方在线，确保连接有效性。

### 为什么需要心跳？

1. 网络故障（如网线拔出、突然断电）可能导致连接中断
2. 若双方无数据交互，无法快速发现断线
3. 通过心跳可以实时检测连接状态

### 实现方式

| 方式 | 说明 |
|------|------|
| TCP Keepalive | 系统层面，默认关闭，间隔长（2小时） |
| 应用层心跳 | 通过IdleStateHandler自定义，灵活可控 |

## IdleStateHandler

Netty通过IdleStateHandler实现心跳检测：

```java
// 参数：读超时、写超时、全超时（单位：秒）
IdleStateHandler idleStateHandler = new IdleStateHandler(5, 5, 10);
```

- **READER_IDLE**：指定时间内未收到数据
- **WRITER_IDLE**：指定时间内未发送数据  
- **ALL_IDLE**：读写均空闲

## 心跳处理

```java
public class HeartbeatHandler extends SimpleChannelInboundHandler<IdleStateEvent> {
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent e = (IdleStateEvent) evt;
            switch (e.state()) {
                case READER_IDLE:
                    // 读空闲，可能是客户端掉线
                    ctx.channel().close();
                    break;
                case WRITER_IDLE:
                    // 写空闲，发送心跳
                    ctx.writeAndFlush(new PingMessage());
                    break;
                case ALL_IDLE:
                    // 读写空闲，发送心跳
                    ctx.writeAndFlush(new PingMessage());
                    break;
            }
        }
    }
}
```

## 断线重连

### 重连触发

```java
public class ReconnectHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        // 触发重连
        doConnect();
    }
    
    private void doConnect() {
        ChannelFuture future = bootstrap.connect(host, port);
        future.addListener((ChannelFutureListener) f -> {
            if (f.isSuccess()) {
                System.out.println("连接成功");
            } else {
                // 失败后延时重连
                f.channel().eventLoop().schedule(this::doConnect, 10, TimeUnit.SECONDS);
            }
        });
    }
}
```

### 重连策略

```java
// 指数退避重连
public void reconnect() {
    int retry = 0;
    long delay = Math.min(MAX_DELAY, BASE_DELAY * (1 << retry));
    eventLoop.schedule(this::doConnect, delay, TimeUnit.SECONDS);
}
```

## Pipeline配置

```java
bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) {
        ChannelPipeline pipeline = ch.pipeline();
        // 心跳检测
        pipeline.addLast("idleStateHandler", 
            new IdleStateHandler(5, 5, 10));
        pipeline.addLast("heartbeatHandler", new HeartbeatHandler());
        // 业务处理器
        pipeline.addLast("handler", new BusinessHandler());
    }
});
```

## 设计要点

1. **心跳间隔**：服务端READER_IDLE > 客户端ALL_IDLE
2. **资源释放**：重连时确保旧连接关闭
3. **重连策略**：设置最大重试次数和间隔
4. **日志记录**：记录连接状态变化

## 适用场景

1. **即时通讯**：IM聊天应用
2. **实时推送**：消息推送服务
3. **物联网**：设备长连接
4. **金融交易**：高频交易系统

## 总结

Netty通过IdleStateHandler和断线重连机制，提供了完善的高可靠性保障。合理配置心跳间隔和重连策略，可以确保长连接的稳定性和应用的可用性。
