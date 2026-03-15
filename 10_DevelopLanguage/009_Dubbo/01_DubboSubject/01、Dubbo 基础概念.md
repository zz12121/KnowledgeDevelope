###### 1. Dubbo 是什么？它主要解决什么问题？
Dubbo 是阿里巴巴开源、后来捐献给 Apache 的一个高性能 Java RPC 框架，核心目标是让远程服务调用像调用本地方法一样简单，同时提供完善的服务治理能力。

它具体解决了这几件事：
- **透明化远程调用**：通过动态代理屏蔽网络通信、序列化/反序列化等复杂细节，开发者只需面向接口编程。
- **服务注册与发现**：Provider 启动后自动向注册中心注册，Consumer 从注册中心订阅服务地址，动态感知服务上下线。
- **负载均衡**：在多个 Provider 实例间分配请求，内置随机、轮询、最少活跃等策略，支持自定义。
- **集群容错**：调用失败时提供 Failover（重试切换）、Failfast（快速失败）、Failsafe（失败安全）等容错机制。
- **服务监控与治理**：提供调用统计监控，以及路由规则、动态配置、服务降级等治理能力。

从源码角度看，核心抽象是 `Invoker`（统一调用模型）和 `URL`（统一配置模型），所有组件通过 SPI 机制组装，每层都可独立替换。

###### 2. Apache Dubbo 与阿里巴巴 Dubbo 有什么区别？
本质是同一套代码在不同发展阶段的产物。

- **阿里巴巴 Dubbo**：2011 年开源，2014 年一度停止维护，2017 年重启，包名 `com.alibaba.dubbo`。
- **Apache Dubbo**：2018 年捐给 Apache 软件基金会，2019 年成为顶级项目，包名改为 `org.apache.dubbo`，两者**不兼容**。

Apache Dubbo 发展更快更开放：积极拥抱云原生（K8s 原生注册、Dubbo Mesh）、引入 Triple 协议（基于 HTTP/2+Protobuf）、支持应用级服务发现，社区也更国际化。

简单记法：捐给 Apache 之前叫阿里版，之后叫 Apache 版，包名是最直观的区分标志。

###### 3. Dubbo 的核心组件有哪些？
Dubbo 核心组件构成服务治理的基础：
- **Provider**：服务提供方，暴露服务的容器。
- **Consumer**：服务消费方，调用远程服务的客户端。
- **Registry**：注册中心，负责服务地址的注册与订阅，支持 ZooKeeper、Nacos、Consul 等。
- **Monitor**：监控中心，统计调用次数和耗时，非强制依赖。
- **Container**：服务运行容器，最常用是 Spring Container。

从代码抽象层面，核心 SPI 接口有：`Protocol`（通信协议）、`Invoker`（调用执行体）、`Exporter`（暴露的服务）、`ProxyFactory`（代理工厂）、`Cluster`（集群容错）、`LoadBalance`（负载均衡）、`Router`（路由规则）、`RegistryFactory`（注册中心工厂）。这些组件通过 SPI 机制灵活扩展和组装。

###### 4. Dubbo 的架构设计是怎样的？
Dubbo 采用**分层架构 + 微内核 + 富插件**的设计，整体分 10 层，核心流程如下：

**消费者端调用链路**：
1. `Proxy` 层为接口生成客户端代理，屏蔽远程调用细节。
2. 调用经过 `Filter` 链做拦截增强（日志、监控、限流等）。
3. `Cluster` 层从 `Directory` 获取 Invoker 列表 → `Router` 路由过滤 → `LoadBalance` 选出一个 Invoker。
4. `Protocol` 层将调用封装成请求，通过 Netty 网络传输。

**提供者端处理链路**：
1. 网络层收到请求，`Protocol` 层解码。
2. 找到对应 `Exporter` → 本地 `Invoker` → 经过 `Filter` 链。
3. 调用真实业务实现，结果原路返回。

**注册中心**作为协调者：Provider 注册，Consumer 订阅，变更时主动通知消费者更新本地 `Directory`。

整个设计中 `Invoker` 是核心模型，`URL` 是贯穿全链路的统一配置模型。

###### 5. 说说 Dubbo 的设计思路
Dubbo 的设计思路可以概括为五个关键点：

1. **透明化与高性能**：动态代理让调用对开发者透明；Netty NIO + 自定义二进制协议（16字节头）+ 单连接多路复用，追求极致性能。

2. **微内核 + 富插件（SPI）**：比 Java 标准 SPI 更强大，支持扩展点自动包装（AOP）、自适应扩展（`@Adaptive`）、条件激活（`@Activate`）。`Protocol`、`Cluster`、`LoadBalance`、`Registry` 等核心组件全部是扩展点，可任意替换。

3. **分层与模块化**：10 层清晰分层，每层职责单一，依赖清晰，可按需引入。

4. **领域模型驱动**：`Invoker`（统一调用体）和 `URL`（统一配置/上下文传递）是两大核心模型，`URL` 贯穿整个调用链，是组件协作的血管。

5. **服务治理为中心**：不只是 RPC 框架，更是服务治理框架。服务发现、负载均衡、容错、路由、监控都是内建核心模块。

###### 6. Dubbo 的分层架构是怎样的？
Dubbo 自上而下分 10 层，上层依赖下层，下层不感知上层：

| 层次 | 作用 |
|---|---|
| **Service** | 业务接口和实现 |
| **Config** | 配置层，`ServiceConfig`/`ReferenceConfig` 等 |
| **Proxy** | 代理层，`ProxyFactory`，Consumer 侧生成代理，Provider 侧封装实现 |
| **Registry** | 注册中心层，`RegistryFactory`/`Registry` |
| **Cluster** | 路由层，封装多 Provider 的路由/负载均衡/容错，对上层伪装成单个 Invoker |
| **Monitor** | 监控层，`MonitorFactory`/`Monitor` |
| **Protocol** | 远程调用层，核心，`Protocol`/`Invoker`/`Exporter` |
| **Exchange** | 信息交换层，封装请求-响应模式，同步转异步 |
| **Transport** | 网络传输层，抽象 Netty/Mina 等为统一接口 |
| **Serialize** | 序列化层，Hessian2/Kryo/Protobuf 等 |

###### 7. Dubbo 的核心功能有哪些？
- **透明远程调用**：支持 Dubbo/Triple/HTTP/gRPC 等多种协议。
- **软负载均衡**：内置 Random/RoundRobin/LeastActive/ConsistentHash 四种算法。
- **集群容错**：Failover/Failfast/Failsafe/Failback/Forking/Broadcast 六种策略。
- **自动注册与发现**：服务自动上下线感知。
- **高度可扩展**：SPI 机制，几乎所有核心组件均可替换。
- **流量调度与治理**：条件路由、标签路由，支持灰度发布、蓝绿部署。
- **动态配置**：通过配置中心（Nacos/Apollo）动态调整超时、权重、路由规则等，无需重启。
- **服务降级与 Mock**：非关键服务不可用时返回 Mock 数据，保护核心链路。
- **可视化治理**：`dubbo-admin` 控制台管理服务、流量、依赖关系。
- **高性能通信**：Netty NIO 异步通信，单一长连接，序列化高效，协议头紧凑。

###### 8. Dubbo 用到哪些设计模式？
Dubbo 大量运用了经典设计模式，是其架构优雅和可扩展的关键：

- **工厂模式**：`ProxyFactory`、`RegistryFactory`、`ExtensionLoader` 都是工厂，`ExtensionLoader.getExtension(name)` 就是工厂方法。
- **代理模式**：核心模式，`JdkProxyFactory` 和 `JavassistProxyFactory` 为 Consumer 创建远程代理。
- **装饰器模式**：`Filter` 链是典型装饰器，每个 Filter 包装 Invoker，在调用前后执行增强逻辑。`ProtocolFilterWrapper.buildInvokerChain()` 清晰展示了这一点。
- **适配器模式**：`ProtocolFilterWrapper`、`ProtocolListenerWrapper` 为 `Protocol` 添加过滤器和监听器，是适配器的体现。
- **观察者模式**：注册中心是典型观察者模式，Provider 变更时注册中心通知所有订阅的 Consumer，`RegistryDirectory.notify()` 是入口。
- **模板方法模式**：`AbstractClusterInvoker.invoke()` 定义选 Invoker、调用、容错的骨架，子类（如 `FailoverClusterInvoker`）实现具体容错逻辑。
- **策略模式**：`LoadBalance`、`Cluster`、`Serialization` 都是策略接口，运行时按配置选择具体实现。
- **单例模式**：`ExtensionLoader` 为每个扩展接口维护单例实例，用于缓存。
- **门面模式**：`DubboBootstrap` 为用户提供统一简化入口，封装复杂初始化过程。

###### 9. 说说你对 Dubbo 中 Invoker 的理解
`Invoker` 是 Dubbo 领域模型中最核心的抽象，代表一个**可执行的调用单元**，核心方法是 `Result invoke(Invocation invocation)`。它统一了本地调用和远程调用，是 Dubbo 屏蔽调用细节的基石。

**两大类型**：
- **本地 Invoker**（Provider 侧）：业务实现被包装成 `AbstractProxyInvoker`，请求到达后通过反射调用真实业务方法。
- **远程 Invoker**（Consumer 侧）：通过 `Protocol.refer()` 创建，如 `DubboInvoker`，内部封装了网络通信客户端，`invoke()` 时序列化请求并通过网络发送。

**在调用链中的位置**：Invoker 会被层层包装形成调用链。从内到外大致是：`业务Invoker → Filter链 → ClusterInvoker（内含 Directory + LoadBalance）`。Cluster 层返回的是一个集群 Invoker，对上层透明。

**与 URL 的关系**：每个 Invoker 都持有一个 `URL`，包含协议、地址、端口、方法参数等全部元数据，是 Invoker 的身份证。

**源码验证**：`DubboInvoker.doInvoke()` 通过 `getClients(url)` 获取 `ExchangeClient`，调用 `request()` 发送请求，这就是远程 Invoker 的本质。`AbstractProxyInvoker.doInvoke()` 则是通过 `wrapper` 反射调用业务实现。

理解了 Invoker，就理解了 Dubbo 调用模型的骨架。

###### 10. Dubbo 3 带来了哪些重要新特性？
Dubbo 3 是一次大版本重构，核心变化：

1. **Triple 协议**：基于 HTTP/2 + Protobuf 的新一代协议，完全兼容 gRPC，支持 Unary/Server Streaming/Client Streaming/Bidirectional Streaming 四种调用模式。穿透防火墙能力强，天然跨语言，是 Dubbo 3 的推荐协议。

2. **应用级服务发现**：传统 Dubbo 是接口级服务发现（注册中心存每个接口的 URL），一个应用有 100 个接口就存 100 条记录，注册中心压力大。Dubbo 3 改为**应用级服务发现**：注册中心只存应用实例（IP+端口），详细元数据推送到元数据中心，大幅减轻注册中心压力，在 K8s 场景下也与原生 Service 机制对齐。

3. **Dubbo Mesh（Proxyless）**：支持 Dubbo 服务直接与 Istio 控制面对接，Consumer 侧 SDK 直接与 xDS 协议交互，无需额外部署 Envoy Sidecar，降低性能损耗和运维复杂度。

4. **响应式编程支持**：Triple 协议原生支持 Reactive Streams（基于 RxJava/Reactor），适合流式数据处理场景。

5. **统一的启动 API**：`DubboBootstrap` 提供流式 API，大幅简化 Dubbo 应用的启动配置代码。

###### 11. Dubbo 与 gRPC 有什么区别和联系？

| 对比维度 | Dubbo | gRPC |
|---|---|---|
| **定位** | 完整的 RPC + 服务治理框架 | 纯 RPC 通信框架 |
| **协议** | Dubbo/Triple/HTTP/gRPC 等多协议 | 仅 HTTP/2 + Protobuf |
| **服务治理** | 内置负载均衡、熔断、路由、注册发现 | 自身不提供，需配合 Envoy/Istio |
| **语言支持** | Java 为主，多语言能力弱 | 官方支持 10+ 语言 |
| **接口定义** | Java 接口，无需 IDL | 必须写 `.proto` 文件 |
| **服务发现** | 依赖注册中心（Nacos/ZK） | 依赖外部（Consul/K8s DNS） |
| **流式通信** | Triple 协议支持（Dubbo 3） | 原生支持，是设计核心 |

**联系**：Dubbo 3 的 Triple 协议完全兼容 gRPC，Dubbo 服务可直接被 gRPC 客户端调用。如果只是做跨语言 RPC 通信，gRPC 更纯粹；如果需要完整的服务治理体系，Dubbo 更合适；两者在云原生场景下也可通过 Mesh 方案协同工作。
