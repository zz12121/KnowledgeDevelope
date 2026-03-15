# 01 - Dubbo 架构与核心原理

> **关联面试题** → `10_DevelopLanguage/009_Dubbo/01_DubboSubject/01、Dubbo基础.md`
> **关联面试题** → `10_DevelopLanguage/009_Dubbo/01_DubboSubject/11、SPI机制.md`

---

## 一、Dubbo 整体架构

### 1.1 十层分层架构

Dubbo 的分层架构从上到下，每层职责单一、可替换：

```
┌─────────────────────────────────────┐
│  Service（业务层）                    │  开发者编写的接口和实现
├─────────────────────────────────────┤
│  Config（配置层）                     │  @DubboService/@DubboReference，ServiceConfig
├─────────────────────────────────────┤
│  Proxy（代理层）                      │  ProxyFactory → 透明化 RPC 调用
├─────────────────────────────────────┤
│  Registry（注册层）                   │  Registry/RegistryFactory → 服务注册与发现
├─────────────────────────────────────┤
│  Cluster（集群层）                    │  Cluster/LoadBalance/Router → 服务治理
├─────────────────────────────────────┤
│  Monitor（监控层）                    │  MonitorFilter → 统计调用次数和耗时
├─────────────────────────────────────┤
│  Protocol（远程调用层）               │  Protocol/Invoker/Exporter → 封装调用
├─────────────────────────────────────┤
│  Exchange（信息交换层）               │  Request/Response → 封装请求响应模型
├─────────────────────────────────────┤
│  Transport（网络传输层）              │  Netty/Mina → 网络连接管理
└─────────────────────────────────────┘
│  Serialize（数据序列化层）            │  Hessian2/Kryo/Protobuf
└─────────────────────────────────────┘
```

**微内核 + 富插件设计**：所有层都是 SPI 扩展点，核心框架只定义接口，实现可完全替换。这是 Dubbo 高度可扩展性的根基。

### 1.2 Invoker 统一调用模型

`Invoker<T>` 是 Dubbo 最核心的抽象，代表"一个可以执行调用的对象"：

```java
public interface Invoker<T> extends Node {
    Class<T> getInterface();
    Result invoke(Invocation invocation) throws RpcException;
}
```

**设计精髓**：无论是本地调用还是远程调用，都统一为 `Invoker.invoke(Invocation)`，这使得：
- `ClusterInvoker`可以在不感知协议的情况下做负载均衡和容错
- `FilterInvoker`可以在调用前后插入任意切面逻辑
- `MockClusterInvoker`可以在调用失败时自动降级

**消费者端调用链**：
```
Proxy（代理对象）
  → MockClusterInvoker（Mock 降级）
    → AbstractClusterInvoker（负载均衡 + 容错）
      → FilterInvoker 链（Filter 责任链）
        → DubboInvoker（网络调用）
          → Netty Channel（实际网络传输）
```

### 1.3 URL 统一配置模型

`URL` 是 Dubbo 的"统一配置总线"，贯穿注册、发现、调用的全链路：

```
dubbo://192.168.1.1:20880/com.example.UserService?
  application=user-provider&
  timeout=3000&
  retries=2&
  version=1.0.0&
  group=user-core&
  serialization=hessian2&
  threads=200
```

一个 URL 包含了服务的所有配置信息。当配置发生变更（如超时），注册中心推送新 URL，框架解析 URL 中的新参数，**零重启生效**。

---

## 二、SPI 机制详解

### 2.1 为什么不用 JDK SPI？

**JDK SPI 的问题**：
- `ServiceLoader.load()` 每次都会遍历并实例化所有实现类，**无法按需加载**
- 无法根据参数动态选择不同实现
- 不支持依赖注入（实现类之间有依赖关系时无解）

**Dubbo SPI 的增强**：

| 特性 | JDK SPI | Dubbo SPI |
|---|---|---|
| 加载方式 | 全量加载所有实现 | 按名称按需加载 + 缓存 |
| 实例缓存 | 无 | 有（每个 name 对应一个实例） |
| 依赖注入 | 无 | 支持 Setter 注入 |
| 自适应扩展 | 无 | `@Adaptive`：运行时根据 URL 参数选实现 |
| 条件激活 | 无 | `@Activate`：根据条件自动激活（如 Filter 链）|
| AOP 增强 | 无 | Wrapper 自动包装（装饰器模式）|

### 2.2 ExtensionLoader 核心机制

```java
// 加载指定名称的扩展实现（按需 + 缓存）
ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);
Protocol dubboProtocol = loader.getExtension("dubbo");
```

**加载流程**：
1. 读取 `META-INF/dubbo/internal/` 和 `META-INF/dubbo/` 下的配置文件
2. 解析 `name=className` 格式的键值对
3. 按名称实例化，注入 Setter 依赖（`AdaptiveExtensionFactory` 从 SPI + Spring 两处查找）
4. 自动识别 Wrapper 类（构造函数接受同接口参数），对所有扩展自动包装（AOP）
5. 结果缓存，下次直接返回

**配置文件示例**（`META-INF/dubbo/org.apache.dubbo.rpc.Protocol`）：
```
dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
tri=org.apache.dubbo.rpc.protocol.tri.TripleProtocol
registry=org.apache.dubbo.registry.integration.RegistryProtocol
filter=org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper    # Wrapper
listener=org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper # Wrapper
```

### 2.3 @Adaptive 自适应扩展

**用途**：在运行时根据 URL 参数动态选择扩展实现，不需要在编译时确定。

```java
@SPI("dubbo")
public interface Protocol {
    @Adaptive({Constants.PROTOCOL_KEY})  // 根据 URL 中的 protocol 参数选择实现
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
}
```

Dubbo 在加载时会为 `Protocol` 接口**动态生成一个 Adaptive 代理类**（字节码生成），代理类的 `export()` 方法大致逻辑：

```java
// 动态生成的代理类（伪代码）
public class Protocol$Adaptive implements Protocol {
    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) {
        URL url = invoker.getUrl();
        String protocol = url.getProtocol(); // 从 URL 取 protocol 参数
        Protocol extension = ExtensionLoader.getExtensionLoader(Protocol.class)
                                            .getExtension(protocol);
        return extension.export(invoker);    // 委托给具体实现
    }
}
```

这样，传入不同协议的 URL，自动路由到不同的 `Protocol` 实现，实现了运行时多态。

### 2.4 @Activate 条件激活

**用途**：根据条件自动选择并激活一批扩展，典型应用是 Filter 链的自动组装。

```java
@Activate(group = {PROVIDER}, order = -110000)  // 在 Provider 端激活，order 控制排序
public class MonitorFilter implements Filter {
    // Provider 端的监控 Filter，自动加入 Filter 链
}

@Activate(group = {CONSUMER, PROVIDER})
public class TimeoutFilter implements Filter {
    // Consumer 和 Provider 端都激活
}
```

`ExtensionLoader.getActivateExtension(url, key, group)` 会扫描所有标注了 `@Activate` 的扩展，根据 group 和 URL 参数过滤，按 order 排序后返回列表。这就是 Filter 链自动组装的实现原理。

### 2.5 Wrapper 自动装饰

Dubbo SPI 会自动识别 Wrapper 类：如果一个扩展类的构造函数接受同接口类型的参数，它就是 Wrapper。

```java
public class ProtocolFilterWrapper implements Protocol {
    private final Protocol protocol;  // 接受 Protocol 类型参数 = Wrapper
    
    public ProtocolFilterWrapper(Protocol protocol) {
        this.protocol = protocol;
    }
    
    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) {
        // 在 export 前包裹 Filter 链
        return protocol.export(buildInvokerChain(invoker, ...));
    }
}
```

所有通过 `ExtensionLoader.getExtension()` 获取的扩展实例，都会自动被所有 Wrapper 包装（类似 AOP），且无需显式配置。

### 2.6 SPI 与 Spring IOC 的协作

`AdaptiveExtensionFactory` 是 `ExtensionFactory` 的自适应实现，整合了两个工厂：

```
AdaptiveExtensionFactory
  ├── SpiExtensionFactory    → 从 Dubbo SPI 容器查找
  └── SpringExtensionFactory → 从 Spring IOC 容器查找
```

当 Dubbo 需要为扩展类注入依赖（Setter 注入）时，会依次从 SPI 和 Spring 两个容器中查找，这使得 Dubbo 扩展点可以无缝依赖 Spring Bean。

`DubboSpringInitializer` 负责在 Spring 容器刷新完成时（`ContextRefreshedEvent`），将 Spring `ApplicationContext` 注册到 `SpringExtensionFactory`，使其能够被后续的 ExtensionLoader 查询。

---

## 三、服务暴露与引用流程

### 3.1 服务暴露（Provider 启动）

```
Spring 容器刷新完成
  → ServiceBean.onApplicationEvent(ContextRefreshedEvent)
    → ServiceConfig.export()
      → 检查 delay 配置（delay=-1 等 Spring 初始化完成）
        → doExport() → doExportUrls()
          → 遍历每个协议 → doExportUrlsFor1Protocol()
            → 构建服务 URL（含协议、地址、所有配置参数）
              → RegistryProtocol.export()
                → DubboProtocol.export()（启动 Netty Server，注册 Invoker）
                  → registry.register(url)（注册到注册中心）
```

### 3.2 服务引用（Consumer 启动）

```
Spring 容器处理 @DubboReference
  → ReferenceAnnotationBeanPostProcessor.postProcessPropertyValues()
    → ReferenceBean.getObject()
      → ReferenceConfig.init()
        → createProxy()
          → Protocol.refer()（从注册中心获取 Provider 列表）
            → RegistryProtocol.refer()
              → RegistryDirectory.subscribe()（监听注册中心变化）
                → Cluster.join(directory)（合并为 ClusterInvoker）
                  → ProxyFactory.getProxy(invoker)（生成动态代理）
                    → 返回代理对象，注入到 @DubboReference 字段
```

### 3.3 一次 RPC 调用的完整路径

```
消费者调用 proxy.method(args)
  → 动态代理类（Javassist 生成）
    → 封装 RpcInvocation（方法名、参数类型、参数值）
      → MockClusterInvoker.invoke()（检查是否强制 Mock）
        → AbstractClusterInvoker.invoke()
          → Router.route()（路由过滤）
            → LoadBalance.select()（负载均衡选择目标）
              → FilterInvoker 链（各种 Filter：监控/日志/超时等）
                → DubboInvoker.doInvoke()
                  → HeaderExchangeChannel.request()
                    → DefaultFuture（异步等待，TimeoutTask 超时检测）
                      → Netty 编码（DubboCodec）→ 网络发送

提供者接收请求
  → Netty 解码（DubboCodec）
    → Dispatcher.dispatch()（根据 dispatcher 策略决定线程）
      → 业务线程池（ThreadPoolExecutor）
        → FilterInvoker 链（Provider 端 Filter）
          → AbstractProxyInvoker.invoke()
            → 反射调用本地服务实现
              → 封装结果 → Netty 编码 → 返回给消费者

消费者收到响应
  → DefaultFuture.doReceived()（唤醒等待线程）
    → Result.recreate()（还原返回值类型）
      → 返回给调用方
```
