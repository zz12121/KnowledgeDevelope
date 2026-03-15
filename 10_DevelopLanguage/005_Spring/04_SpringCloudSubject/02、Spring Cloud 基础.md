###### 1. 什么是 Spring Cloud？

Spring Cloud 是一套完整的**微服务解决方案工具集**，基于 Spring Boot 实现了分布式系统中的常见模式——配置管理、服务发现、断路器、路由、负载均衡等。

说白了，Spring Cloud 不是一个框架，而是把业界成熟组件（Netflix OSS、Consul、Alibaba 组件等）整合到一起的**工具容器**，让你用 Spring Boot 的方式快速搭起微服务基础设施。

核心设计哲学：遵循"约定优于配置"，一个 `@EnableEurekaServer` 注解就能启动注册中心，门槛很低。

📖 [[../../../../27_SpringCloudKnowledge/02_服务注册与发现/01、服务注册与发现#三、Nacos]]

---

###### 2. Spring Cloud 的核心组件有哪些？

按功能领域分：

- **服务治理**：Eureka / Nacos / Consul — 服务注册与发现
- **配置管理**：Spring Cloud Config / Nacos — 集中化管理配置，支持动态刷新
- **服务调用**：OpenFeign / RestTemplate — 声明式服务调用
- **负载均衡**：Ribbon / Spring Cloud LoadBalancer — 客户端负载均衡
- **API 网关**：Spring Cloud Gateway / Zuul — 统一入口，路由 + 安全 + 监控
- **容错保护**：Hystrix / Sentinel / Resilience4j — 熔断、降级，防雪崩
- **消息驱动**：Spring Cloud Stream — 抽象消息中间件（Kafka/RabbitMQ）
- **链路追踪**：Spring Cloud Sleuth + Zipkin — 分布式请求全链路追踪

📖 [[../../../../27_SpringCloudKnowledge/02_服务注册与发现/01、服务注册与发现#二、Eureka]]

---

###### 3. 说说 Spring Boot 和 Spring Cloud 的关系

两者是**基础与扩展**的关系：

- **Spring Boot**：解决的是单个微服务的快速开发问题——自动配置、起步依赖、内嵌容器
- **Spring Cloud**：解决的是多个微服务之间如何协作、治理的问题——服务发现、配置中心、熔断等
- **依赖关系**：Spring Cloud 必须基于 Spring Boot，Spring Boot 可以独立运行

类比：Spring Boot 是造每一辆汽车，Spring Cloud 是建高速公路网络把汽车连接起来。

```java
// Spring Cloud 基于 Spring Boot 的自动配置机制扩展
@Configuration
public class EurekaClientAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public EurekaClient eurekaClient(ApplicationInfoManager manager, EurekaClientConfig config) {
        return new CloudEurekaClient(manager, config);
    }
}
```

📖 [[../../../../27_SpringCloudKnowledge/02_服务注册与发现/01、服务注册与发现#二、Eureka]]

---

###### 4. Spring Cloud 的版本命名规则是什么？

Spring Cloud 版本命名经历了两个阶段：

**2020 年前**：用伦敦地铁站名命名，按字母顺序排列（Angel → Brixton → Camden → … → Hoxton），每个版本对应特定的 Spring Boot 版本范围。

**2020 年后**：改用**日历化版本**格式 `YYYY.MINOR.PATCH`：
- `2020.0.x`（Ilford）
- `2021.0.x`（Jubilee）
- `2022.0.x`（Kilburn）
- `2023.0.x`（Leyton）

**重要提醒**：Spring Cloud 和 Spring Boot 的版本必须匹配，具体对应关系看官方兼容性矩阵。版本选错了会遇到各种莫名其妙的问题。

📖 [[../../../../27_SpringCloudKnowledge/02_服务注册与发现/01、服务注册与发现#二、Eureka]]

---

###### 5. Spring Cloud 与 Dubbo 的区别是什么？

两者不是竞争关系，而是不同定位的解决方案：

**Spring Cloud**：全栈式微服务生态，通信用 HTTP REST（文本协议），更注重标准化和开放性，Spring 全家桶无缝集成。

**Apache Dubbo**：高性能 RPC 框架，通信用自定义 TCP 二进制协议，性能更高、吞吐量更大，但需要自行集成其他生态组件。

**选择建议**：
- 对性能要求极高、内部服务调用频繁 → 考虑 Dubbo
- Spring 技术栈、需要完整微服务生态 → Spring Cloud

**发展趋势**：Spring Cloud Alibaba 已将 Dubbo 整合进来，可以两者一起用。

📖 [[../../../../27_SpringCloudKnowledge/03_服务调用与负载均衡/01、服务调用与负载均衡#一、服务间通信方式]]

---

###### 6. Spring Cloud 的技术选型原则有哪些？

**1. 看维护状态**：Netflix 组件（Eureka、Hystrix、Zuul）已停更，新项目优先选 Spring Cloud Alibaba（Nacos、Sentinel、Gateway）或 Consul。

**2. 看 CAP 要求**：
- 高可用优先 → AP 模型 → Eureka/Nacos（临时实例）
- 一致性优先 → CP 模型 → Consul/Zookeeper/Nacos（非临时实例）

**3. 看性能要求**：高并发选 Nacos（长连接推送，感知更快）；消息密集型用 Spring Cloud Stream + Kafka。

**4. 看云原生兼容性**：Kubernetes 环境考虑与 Istio Service Mesh 整合；混合云要看多注册中心支持。

📖 [[../../../../27_SpringCloudKnowledge/01_微服务架构基础/01、微服务架构基础概念#二、CAP 理论]]

---

###### 7. Spring Cloud 如何实现服务的注册？

服务注册的核心机制是**心跳维持的注册表同步**，通过 `ServiceRegistry` 接口抽象，支持多种注册中心实现：

```java
// Eureka 注册流程（简化）
public void register(EurekaRegistration reg) {
    InstanceInfo instanceInfo = buildInstanceInfo(reg);        // 1. 构建实例信息
    reg.getApplicationInfoManager().setInstanceStatus(UP);     // 2. 发送注册请求
    reg.getEurekaClient().registerHealthCheck(healthHandler);  // 3. 启动心跳线程
}
```

Nacos 注册相比 Eureka 的增强：
- 实例信息可持久化到数据库，重启后不丢失
- 支持 TCP/HTTP 等多种主动健康检查方式
- 支持动态调整实例权重，灰度发布

📖 [[../../../../27_SpringCloudKnowledge/02_服务注册与发现/01、服务注册与发现#三、Nacos]]

---

###### 8. Spring Cloud 的生态组件有哪些？

**核心组件（Spring 官方）**：
- **Spring Cloud Config**：配置中心，基于 Git 存储
- **Spring Cloud Gateway**：API 网关，基于 WebFlux 非阻塞
- **Spring Cloud Sleuth**：链路追踪，生成 TraceID/SpanID

**Netflix 系（逐步停更）**：
- Eureka（注册中心）、Hystrix（熔断器）、Zuul（网关）、Ribbon（负载均衡）

**Alibaba 系（当前主流）**：
- Nacos（注册 + 配置一体）、Sentinel（流量控制）、Seata（分布式事务）

**组件选型趋势**：从 Netflix OSS 向 Spring Cloud Alibaba 和云原生方案迁移。

📖 [[../../../../27_SpringCloudKnowledge/04_熔断降级与网关/01、熔断降级与服务网关#四、Sentinel]]

---

###### 9. Spring Cloud Alibaba 与 Spring Cloud Netflix 的区别？

两套技术体系，功能对应关系：

- **注册中心**：Eureka（AP，已停更）vs Nacos（AP/CP 可切换，持续活跃）
- **配置中心**：Spring Cloud Config（需配合 Bus 刷新）vs Nacos（原生推送，一体化）
- **熔断降级**：Hystrix（停更）vs Sentinel（精细化流量控制，热点限流）
- **API 网关**：Zuul（停更）vs Spring Cloud Gateway（WebFlux，性能更好）
- **分布式事务**：Netflix 无官方方案 vs Seata（AT/TCC/Saga 模式）

**结论**：新项目无脑选 Spring Cloud Alibaba，现有 Netflix 项目逐步迁移。

📖 [[../../../../27_SpringCloudKnowledge/02_服务注册与发现/01、服务注册与发现#三、Nacos]]

---

###### 10. Spring Cloud 的发展历程和趋势是什么？

**发展历程**：
1. **初创期（2015-2017）**：整合 Netflix OSS，奠定微服务基础能力
2. **成熟期（2018-2020）**：完善配置管理、消息总线等组件
3. **变革期（2021 至今）**：Netflix 组件全面停更，Spring Cloud Alibaba 成主流

**技术趋势**：
- **云原生融合**：与 Kubernetes、Service Mesh（Istio）深度集成
- **响应式编程**：全面拥抱 WebFlux 和非阻塞 IO
- **Serverless**：适应函数计算等无服务器架构

📖 [[../../../../27_SpringCloudKnowledge/01_微服务架构基础/01、微服务架构基础概念#四、微服务设计原则]]
