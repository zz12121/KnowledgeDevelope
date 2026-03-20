# Exchange组件

## 概述

Exchange（信息交换层）是Dubbo Remoting的最上层，封装了请求-响应的语义，实现"一问一答"的交互模式，并将同步调用转为异步。

## 核心设计

以**Request**和**Response**为中心，针对Channel、ChannelHandler、Client、RemotingServer等接口进行实现。

## 核心组件

### Request和Response

- **Request**：封装请求信息，包含ID、版本、双向通信标识、事件标识、数据负载
- **Response**：封装响应信息，包含与请求对应的ID、状态码、错误消息、响应结果

### ExchangeChannel

```java
public interface ExchangeChannel {
    // 异步请求，返回CompletableFuture
    CompletableFuture<Object> request(Object request) throws RemotingException;
}
```

**HeaderExchangeChannel**是其实现，装饰底层Channel，负责发送请求并创建`DefaultFuture`。

### DefaultFuture

继承`CompletableFuture`，管理请求与响应的映射关系：

- **CHANNELS**：维护请求ID与Channel的映射
- **FUTURES**：维护请求ID与DefaultFuture的映射
- **超时检测**：通过时间轮（HashedWheelTimer）实现

### HeaderExchangeHandler

处理接收到的消息：
- 只读请求：标记Channel为只读
- 双向请求：委托上层处理，并返回响应
- 单向请求：直接委托上层处理
- 响应消息：通知对应的DefaultFuture完成

### HeaderExchangeClient

装饰Client实现，提供：
- **心跳机制**：定时发送心跳请求
- **重连机制**：定时检测连接状态
- **优雅关闭**：等待所有请求完成

### HeaderExchangeServer

装饰Server实现，支持：
- 定时关闭空闲连接
- 优雅关闭流程

### HeaderExchanger

通过SPI机制加载，是Exchange层的入口，在Transport层基础上添加装饰器。

### ExchangeCodec

负责协议编解码，支持Dubbo协议头（16字节）的解析与构造，包含魔数、请求/响应标志、序列化方式、请求ID、数据长度等。

## 工作流程

1. 上层调用`ExchangeChannel.request()`
2. 创建`DefaultFuture`并注册
3. 通过底层Channel发送请求
4. 收到响应后，`HeaderExchangeHandler`通知对应的`DefaultFuture`完成
5. 客户端获取结果

## 设计优势

1. **同步转异步**：将同步RPC调用转为异步
2. **超时控制**：通过时间轮实现精确超时检测
3. **心跳维持**：定期检测连接状态
4. **优雅关闭**：确保请求完整处理

## 总结

Dubbo Exchange层通过装饰器模式和异步Future机制，将底层传输细节封装为高效的请求-响应模型，并提供了心跳、重连、超时控制等企业级特性。
