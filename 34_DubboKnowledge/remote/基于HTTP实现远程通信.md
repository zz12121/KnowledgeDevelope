# 基于HTTP实现远程通信

## 概述

Dubbo HTTP协议是基于HTTP表单的远程调用协议，采用Spring的HttpInvoker实现。该协议适用于需要HTTP协议透传的场景，如跨防火墙调用或需要同时给应用程序和浏览器JS使用的服务。

## 协议特性

- **传输协议**：HTTP
- **连接方式**：短连接（每请求建立一次连接）
- **同步传输**：阻塞式调用
- **默认端口**：80（可配置）
- **跨防火墙**：支持，适合需要穿透防火墙的场景

## 核心组件

### HttpProtocol

服务暴露和服务引用的核心入口：

```java
public class HttpProtocol extends AbstractHttpProtocol {
    // 服务暴露
    <T> Exporter<T> export(Invoker<T> invoker) {
        // 通过HttpBinder绑定HTTP服务器
        // 创建HttpInvokerServiceExporter
    }
    
    // 服务引用
    <T> Invoker<T> refer(Class<T> type, URL url) {
        // 创建HttpInvokerProxyFactoryBean
    }
}
```

### HttpRemoteInvocation

扩展Spring的`RemoteInvocation`，附加Dubbo特有的参数：

- **RPC上下文**：通过`RpcContext.getContext().getAttachments()`传递隐式参数
- **泛化调用标识**：支持GenericService调用
- **序列化方式**：将调用信息序列化为HTTP Body

### InternalHandler

HTTP请求的入口处理器：

1. 从请求URI中映射到具体的Exporter
2. 将客户端IP和端口设置到RpcContext中
3. 委托给对应的HttpInvokerServiceExporter执行调用

## 调用流程

### 消费者端

1. 通过动态代理调用接口方法
2. 生成`HttpRemoteInvocation`对象（包含Dubbo附件和泛化标识）
3. 通过配置的HTTP客户端发送POST请求到服务提供者

### 服务端

1. `InternalHandler`接收请求，根据URI路由到对应的Exporter
2. Exporter反序列化请求，恢复RPC上下文
3. 通过反射调用真实实现类
4. 结果通过HTTP响应返回

## HTTP客户端实现

Dubbo HTTP协议支持两种HTTP客户端：

| 模式 | 实现类 | 说明 |
|------|--------|------|
| simple | SimpleHttpInvokerRequestExecutor | 使用JDK原生HttpURLConnection |
| commons | HttpComponentsHttpInvokerRequestExecutor | 使用Apache HttpClient |

通过URL参数选择：`http://host:port/?http.client=simple`

## 配置示例

### Spring Boot配置

```yaml
dubbo:
  protocol:
    name: http
    port: 8080
```

### XML配置

```xml
<!-- 服务提供者 -->
<dubbo:protocol name="http" port="8080" />

<!-- 服务消费者 -->
<dubbo:reference interface="com.example.DemoService" url="http://10.20.153.10:8080/" />
```

### 代码配置

```java
// 服务暴露
ProtocolConfig protocolConfig = new ProtocolConfig();
protocolConfig.setName("http");
protocolConfig.setPort(8080);
```

## 特性支持

### 泛化调用

通过路径`/service/generic`和`GenericService`接口实现：

```
http://provider-host:port/com.example.DemoService/generic
```

### 隐式传参

利用RpcContext的Attachments传递跨服务参数（如TraceID）：

```java
// 消费者端
RpcContext.getContext().setAttachment("traceId", "123456");

// 服务端
String traceId = RpcContext.getContext().getAttachment("traceId");
```

### 超时控制

通过URL参数设置：
- `connect.timeout`：连接超时时间
- `timeout`：读取超时时间

```java
url = "http://host:port/service?timeout=3000&connect.timeout=1000"
```

## 与Dubbo协议对比

| 特性 | HTTP协议 | Dubbo协议 |
|------|----------|-----------|
| 性能 | 较低 | 高 |
| 跨防火墙 | 好 | 一般 |
| 跨语言 | 好 | 差 |
| 适用场景 | Web/移动端 | Java内部服务 |
| 连接方式 | 短连接 | 长连接 |

## 适用场景

1. **跨防火墙调用**：需要穿透防火墙的公网环境
2. **多语言混合**：消费者可能是PHP、Python等非Java语言
3. **Web和移动端**：需要同时给浏览器JS或移动App提供服务
4. **简单集成**：不想引入复杂依赖的轻量级场景

## 总结

Dubbo HTTP协议基于Spring HttpInvoker实现，提供了兼容HTTP协议的远程调用能力。虽然性能不如Dubbo默认的TCP私有协议，但其跨防火墙、跨语言的特性使其在特定场景下是理想的选择。
