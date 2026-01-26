###### 1. 什么是服务网关？
服务网关是分布式系统或微服务架构中的**核心组件**，它作为系统的**统一入口**，所有外部请求都首先经过网关处理。从技术本质看，网关是连接不同网络的**协议转换器**，工作在OSI参考模型的传输层及以上层级，尤其擅长在应用层进行消息转换和路由。
在微服务语境下，服务网关可抽象为 **"路由转发 + 过滤器"**​ 的组合。它不仅实现基本的请求路由，还通过过滤器链机制集成横切关注点（Cross-Cutting Concerns），如认证、限流、监控等。
###### 2. 网关的作用是什么？
网关的核心作用可归纳为以下五点：
1. **统一入口与路由转发**：作为所有客户端请求的唯一入口，根据预设规则将请求精准路由到后端相应的微服务实例。
2. **负载均衡**：在将请求分发到多个服务实例时，采用负载均衡策略（如轮询、最小连接数等），避免单一实例过载。
3. **安全防护**：集成安全功能，如**身份认证**（验证请求者身份）、**授权**（检查访问权限）、**SSL/TLS终止**（解密入口流量，减轻后端压力）、IP黑白名单等，构筑系统安全第一道防线。 
4. **横切功能集中处理**：将微服务架构中多个服务需要的公共功能（如日志记录、请求监控、限流熔断）集中在网关实现，避免代码冗余，提升系统可维护性。
5. **协议转换与适配**：在不同协议之间进行转换（如HTTP/1.1与gRPC互转），简化客户端与后端微服务之间的通信。
###### 3. 什么是微服务网关？为什么需要服务网关？
**微服务网关**是专门为微服务架构设计的业务网关，作为所有微服务的统一流量入口，封装了内部微服务系统的架构，并集中处理非业务功能。
**需要服务网关的主要原因**在于解决微服务架构直接暴露内部服务带来的问题：
- **API聚合与简化**：网关可将多个细粒度微服务的API聚合为粗粒度接口，减少客户端请求次数，简化客户端逻辑。
- **解耦与隐匿内部结构**：客户端仅与网关交互，无需知晓内部微服务的数量、位置和架构细节。内部微服务调整时，只需修改网关配置，客户端无需改动，降低了耦合度。
- **集中治理**：在网关层面统一实施安全策略、流量控制、监控日志收集等，比在每个微服务中单独实现更高效、一致。
###### 4. 网关与过滤器有什么区别？
这是一个关于**整体与部分**的关系问题。
- **网关**：是一个**完整的架构组件**，是一个运行实体（如独立的服务进程）。它定义了请求处理的整体流程和边界，是容器或平台的概念。
- **过滤器**：是网关（或其他组件）内部执行特定功能的**逻辑单元**，是构成网关处理链条的环节。网关的强大功能往往通过一系列有序执行的过滤器来实现。
以Netflix Zuul为例，其核心就是一个动态过滤器链，请求处理过程涉及`pre`（前置）、`route`（路由）、`post`（后置）等不同类型的过滤器协作。
###### 5. 常见的网关解决方案有哪些？
主流网关解决方案可根据其设计侧重分为以下几类：

|**网关名称**​|**技术栈/语言**​|**核心特点**​|**典型适用场景**​|
|---|---|---|---|
|**Nginx**​|C|高性能**流量网关**，专注于负载均衡、反向代理、静态内容缓存。|作为入口层的流量分发和负载均衡器。|
|**Spring Cloud Gateway**​|Java (Reactive)|Spring官方出品，基于**WebFlux响应式编程模型**（非阻塞IO），与Spring Cloud生态无缝集成，性能优异。|Java技术栈的微服务项目，特别是Spring Cloud体系。|
|**Netflix Zuul**​|Java|早期微服务网关经典方案，Zuul 2.x支持异步非阻塞模型。与Netflix OSS组件集成度高。|传统Spring Cloud项目，已有Zuul技术积累的团队。|
|**Kong**​|Lua/OpenResty|基于Nginx，**插件生态丰富**，API管理能力强，社区成熟。|需要丰富插件支持、多语言异构架构的大规模分布式系统。|
|**Apache APISIX**​|Lua/Go|动态、实时、高性能的API网关，支持热加载和多种协议。|云原生环境，对性能和控制力要求极高的场景。|
###### 6. 什么是 Zuul？
Zuul是Netflix开源的一款提供**动态路由、监控、弹性和安全**等边缘服务的**微服务网关**组件。它在微服务架构中扮演**流量门卫**的角色，是所有请求进入系统后的第一站。
Zuul的核心工作原理是**基于过滤器链**。一个HTTP请求在Zuul中的生命周期会经历一系列具有不同职责的过滤器处理。
###### 7. Zuul 的工作原理是什么？
Zuul 1.x 的工作流程围绕**Servlet和过滤器链**展开：
1. **请求接收**：外部请求首先到达Zuul Server，由`ZuulServlet`接收。
2. **过滤器链执行**：`ZuulServlet`将请求委托给`ZuulRunner`，进而通过`FilterProcessor`来执行一系列过滤器。Zuul会从`FilterLoader`加载过滤器（支持Groovy热加载）。
3. **过滤器类型与顺序**：
    - **`pre`Filters（前置路由过滤器）**：在路由到具体微服务之前执行，用于实现**身份验证**、**路由决策**、**请求日志记录**等。
    - **`route`Filters（路由过滤器）**：负责将请求**转发**到具体的微服务。Zuul默认使用Apache HttpClient或Ribbon进行转发。
    - **`post`Filters（后置路由过滤器）**：在请求已被路由到微服务并收到响应后执行，用于添加标准HTTP Header、收集统计信息、将响应流发送回客户端等。
    - **`error`Filters（错误过滤器）**：在上述任何阶段发生错误时触发，用于进行错误处理和定制错误响应。
**Zuul 2.x**​ 进行了架构升级，采用了**异步非阻塞模型**（基于Netty），显著提升了吞吐量和并发处理能力。
###### 8. Zuul 的过滤器类型有哪些？
如上所述，Zuul的过滤器主要有四种类型，其执行顺序和作用概括如下表：

|过滤器类型|执行时机|核心职责|
|---|---|---|
|**`pre`**​|路由之前|认证、安全、路由决策、日志|
|**`route`**​|路由过程中|请求转发（如HTTP/Ribbon）|
|**`post`**​|路由之后（收到响应）|响应头处理、指标收集、流式响应|
|**`error`**​|发生错误时|错误处理、统一异常响应|
###### 9. 什么是 Spring Cloud Gateway？
Spring Cloud Gateway是Spring官方基于**Spring 5、Project Reactor和Spring Boot 2.x**构建的**API网关**。它旨在为微服务架构提供一种简单、有效且统一的API路由管理方式，其核心目标是替代Netflix Zuul。
它的设计哲学是使用**函数式编程模型**和**响应式栈**（Reactive Stack），构建在非阻塞的Netty网络服务器之上，从而具备处理高并发流量的能力。
###### 10. Gateway 与 Zuul 的区别是什么？

|特性维度|**Spring Cloud Gateway**​|**Netflix Zuul (1.x)**​|
|---|---|---|
|**编程模型/性能**​|**响应式模型**（WebFlux），**非阻塞IO**，资源消耗少，**性能更高**，尤其适合长连接（如WebSocket）。|**阻塞IO模型**（Servlet），每个请求占用一个线程，高并发下线程资源消耗大，性能有瓶颈。|
|**技术生态**​|**Spring"亲生子"**，与Spring Cloud生态（如LoadBalancer、CircuitBreaker）无缝深度集成。|Netflix OSS组件，与Eureka/Ribbon/Hystrix等集成，但Netflix相关组件已逐步进入维护模式。|
|**功能特性**​|提供**强大的断言（Predicate）和过滤器（Filter）机制**，配置更灵活，功能更现代（如支持熔断、限流、灰度发布）。|过滤器功能稳定，但扩展性和现代流量治理功能相对较弱。|
**结论**：对于新项目，尤其是基于Spring Boot 2.x+的微服务系统，**Spring Cloud Gateway是更推荐的选择**。
###### 11. Gateway 的核心概念有哪些（路由、断言、过滤器）？
Spring Cloud Gateway的架构基于三个核心概念，理解它们是掌握其用法的关键：
1. **路由**：这是网关的基本构建块。一个**RouteDefinition**主要包含：
    - **ID**：路由的唯一标识符。
    - **目标URI**：请求最终要被转发到的地址。
    - **断言集合**（Predicates）：判断条件，决定何时匹配该路由。
    - **过滤器集合**（Filters）：在发送下游请求前或后对请求/响应进行修改的逻辑。
2. **断言**：这是Java 8中的**`Predicate`**接口的实现。它用于检查HTTP请求的各项属性（如Header、Method、Path、Query参数等），判断当前请求是否与该路由匹配。例如，`Path=/api/users/**`就是一个路径断言。
3. **过滤器**：这是Spring框架中**`GatewayFilter`**接口的实例。它们可以在将请求发送到下游之前（"pre"）或之后（"post"）拦截并修改请求和响应。过滤器用于实现横切关注点，如添加请求头、权限校验、限流等。
###### 12. 什么是 Predict（断言）？
在Spring Cloud Gateway中，Predicate是一个**路由匹配的核心抽象**，源自Java 8的函数式接口`java.util.function.Predicate`。它接受一个`ServerWebExchange`输入参数，返回布尔值，决定请求是否与当前路由规则匹配。
**源码层面的实现机制**：
```java
// 核心接口定义
public interface Predicate<T> {
    boolean test(T t);
}

// Gateway中的具体应用
public interface RoutePredicateFactory extends ShortcutConfigurable {
    Predicate<ServerWebExchange> apply(Consumer<T> consumer);
}
```
**设计模式分析**：Gateway采用**工厂方法模式**，每个Predicate对应一个`XxxRoutePredicateFactory`。当配置如`Path=/api/**`时，Gateway会通过`PathRoutePredicateFactory`创建对应的Predicate实例。
**执行流程**：在`RoutePredicateHandlerMapping`中，Gateway遍历所有路由的Predicate集合，只有全部返回true的请求才会进入该路由的过滤器链。
###### 13. Gateway 的断言类型有哪些？
Spring Cloud Gateway提供了丰富的内置断言工厂，以下是核心分类及应用场景：
**时间相关断言**
- **AfterRoutePredicateFactory**：匹配指定时间之后的请求
- **BeforeRoutePredicateFactory**：匹配指定时间之前的请求
- **BetweenRoutePredicateFactory**：匹配时间区间内的请求
```yaml
predicates:
  - After=2024-01-01T00:00:00+08:00[Asia/Shanghai]
  - Between=2024-01-01T00:00:00+08:00,2024-12-31T23:59:59+08:00
```
**请求特征断言**
- **PathRoutePredicateFactory**：基于URL路径匹配（最常用）
- **MethodRoutePredicateFactory**：按HTTP方法匹配
- **QueryRoutePredicateFactory**：按查询参数匹配
- **HeaderRoutePredicateFactory**：按请求头匹配
- **CookieRoutePredicateFactory**：按Cookie值匹配
```yaml
predicates:
  - Path=/api/v1/users/**
  - Method=GET,POST
  - Query=version,v[0-9]+
  - Header=X-Request-ID,\d+
  - Cookie=sessionId,.+
```
**网络相关断言**
- **RemoteAddrRoutePredicateFactory**：按客户端IP匹配
- **HostRoutePredicateFactory**：按Host头匹配域名
```yaml
predicates:
  - RemoteAddr=192.168.1.1/24
  - Host=**.example.com
```
**高级断言**
- **WeightRoutePredicateFactory**：权重路由，用于灰度发布
- **CloudFoundryRouteServiceRoutePredicateFactory**：云环境专用
**组合断言**：支持AND逻辑组合，提供精细化的路由控制：
```yaml
predicates:
  - Path=/api/**
  - Method=GET
  - Header=Authorization,Bearer.*
  - After=2024-01-01T00:00:00+08:00
```
###### 14. Gateway 的过滤器有哪些类型？
Gateway过滤器分为**全局过滤器**和**网关过滤器**两大类，从作用范围和功能上形成完整体系。
**网关过滤器**
作用于特定路由，通过配置在routes下实现：
**请求/响应修改类**：
- `AddRequestHeaderFilter`/`AddResponseHeaderFilter`：添加头信息
- `RemoveRequestHeaderFilter`/`RemoveResponseHeaderFilter`：移除头信息
- `RewritePathFilter`：路径重写
- `PrefixPathFilter`：添加路径前缀
**参数处理类**：
- `AddRequestParameterFilter`：添加请求参数
- `RewriteRequestParameterFilter`：重写请求参数
**特殊功能类**：
- `StripPrefixFilter`：去除路径前缀
- `RetryFilter`：请求重试
- `RequestRateLimiterFilter`：限流控制
**全局过滤器**
全局生效，实现`GlobalFilter`接口，核心包括：
- `LoadBalancerClientFilter`：服务发现与负载均衡
- `RouteToRequestUrlFilter`：路由URL重写
- `NettyRoutingFilter`：Netty客户端路由转发
- `NettyWriteResponseFilter`：响应写回
- `GatewayMetricsFilter`：指标收集
###### 15. Gateway 的全局过滤器与局部过滤器的区别？

|**特性**​|**全局过滤器**​|**网关过滤器**​|
|---|---|---|
|**作用范围**​|所有路由生效|仅对配置的特定路由生效|
|**配置方式**​|代码实现`GlobalFilter`接口|在routes配置中通过filters属性定义|
|**执行顺序**​|通过`Ordered`接口或`@Order`注解控制|在配置文件中按顺序执行|
|**功能定位**​|核心基础设施功能（负载均衡、路由转发等）|业务相关功能（头信息处理、路径修改等）|
|**数量限制**​|数量相对固定，由Gateway框架定义|可根据业务需要灵活配置多个|
**源码设计差异**：全局过滤器通过`GlobalFilter`接口实现，由Gateway自动加载；网关过滤器通过`GatewayFilter`接口实现，需显式配置。
###### 16. Gateway 的过滤器执行顺序是什么？
Gateway采用**责任链模式**执行过滤器，顺序由`Order`值决定，值越小优先级越高。
 **核心过滤器执行顺序**：
1. **负载均衡过滤**：`ReactiveLoadBalancerClientFilter`(Order: 10100)
2. **路由转换过滤**：`RouteToRequestUrlFilter`(Order: 10000)
3. **转发路由过滤**：`ForwardRoutingFilter`(Order: 0)
4. **Netty路由过滤**：`NettyRoutingFilter`(Order: -1)
5. **响应写回过滤**：`NettyWriteResponseFilter`(Order: -1)
**自定义过滤器顺序控制**：
```java
@Component
public class CustomGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 前置处理
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            // 后置处理
        }));
    }
    
    @Override
    public int getOrder() {
        return -1; // 高优先级
    }
}
```
**执行流程**：前置处理按Order升序执行，后置处理按降序执行，形成完整的过滤器链。
###### 17. 如何在 Spring Cloud Gateway 中实现限流？
Gateway基于**令牌桶算法**实现限流，主要集成Redis和RateLimiter组件。
 **基于Redis的限流配置**：
```yaml
spring:
  redis:
    host: localhost
    port: 6379
  cloud:
    gateway:
      routes:
        - id: user_service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10    # 每秒令牌数
                redis-rate-limiter.burstCapacity: 20    # 令牌桶容量
                key-resolver: "#{@userKeyResolver}"      # 限流键解析器
```
**自定义限流键解析器**：
```java
@Bean
KeyResolver userKeyResolver() {
    return exchange -> Mono.just(
        exchange.getRequest().getQueryParams().getFirst("userId")
    );
}
```
**源码层面的限流实现**：
```java
public class RedisRateLimiter implements RateLimiter {
    public Mono<Response> isAllowed(String routeId, String id) {
        // 基于Lua脚本实现原子操作
        return redisTemplate.execute(script, keys, args);
    }
}
```
**算法原理**：令牌桶以固定速率生成令牌，请求获取令牌后才能通过，突发流量可消耗桶内积累的令牌。
###### 18. Gateway 如何实现动态路由？
动态路由允许**运行时修改路由配置**而不重启服务，主要通过以下方式实现：
**基于Nacos配置中心**：
```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
      routes:
        - id: dynamic_route
          uri: lb://user-service
          predicates:
            - Path=/api/**
```
 **编程式动态路由**：
```java
@Component
public class DynamicRouteService {
    
    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;
    
    public void addRoute(String id, String path, String uri) {
        RouteDefinition definition = new RouteDefinition();
        definition.setId(id);
        
        // 配置Predicate
        PredicateDefinition predicate = new PredicateDefinition();
        predicate.setName("Path");
        predicate.addArg("pattern", path);
        definition.getPredicates().add(predicate);
        
        // 设置目标URI
        definition.setUri(URI.create(uri));
        
        // 保存路由定义
        routeDefinitionWriter.save(Mono.just(definition)).subscribe();
    }
}
```
**监听配置变更**：
```java
@Component
public class RouteRefreshListener {
    @EventListener
    public void refreshRoutes(RefreshRoutesEvent event) {
        // 触发路由刷新
        routeLocator.refresh();
    }
}
```
**生产实践**：结合配置中心如Nacos，实现路由规则的热更新，支持灰度发布、AB测试等场景。
###### 19. Gateway 如何实现灰度发布？
灰度发布通过**权重路由和元数据过滤**实现流量的可控分发。
**权重路由配置**：
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: gray_user_service_v1
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
            - Weight=user-service, 90
          metadata:
            version: v1.0
            
        - id: gray_user_service_v2  
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
            - Weight=user-service, 10
          metadata:
            version: v2.0
```
**基于Header的灰度路由**：
```java
@Component
public class GrayFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        Request request = exchange.getRequest();
        
        // 检查灰度标识
        if (request.getHeaders().containsKey("X-Gray-Version")) {
            String version = request.getHeaders().getFirst("X-Gray-Version");
            // 将版本信息传递给负载均衡器
            exchange.getAttributes().put(GRAY_VERSION, version);
        }
        
        return chain.filter(exchange);
    }
}
```
 **自定义负载均衡策略**：
```java
@Bean
public ServiceInstanceListSupplier grayServiceInstanceSupplier() {
    return new GrayServiceInstanceListSupplier();
}
```
**实现原理**：通过定制`ReactiveLoadBalancer`，根据灰度规则选择对应版本的服务实例。
###### 20. Gateway 如何实现跨域配置？
Gateway提供灵活的CORS配置，支持全局和路由级别的跨域控制。
**全局CORS配置**：
```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowed-origins: "https://example.com"
            allowed-methods: 
              - GET
              - POST
              - PUT
            allowed-headers: "*"
            allow-credentials: true
            max-age: 3600
```
**基于Java Config的精细控制**：
```java
@Configuration
public class CorsConfiguration {
    @Bean
    public CorsWebFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(true);
        config.addAllowedOrigin("https://example.com");
        config.addAllowedMethod("*");
        config.addAllowedHeader("*");
        config.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        
        return new CorsWebFilter(source);
    }
}
```
 **路由级别CORS**：
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: api_route
          uri: lb://api-service
          predicates:
            - Path=/api/**
          metadata:
            cors:
              allowed-origins: "https://api.example.com"
              allowed-methods: "GET,POST"
```
**预检请求处理**：Gateway自动处理OPTIONS预检请求，无需业务方干预。
###### 21. Gateway 如何实现统一鉴权？
统一鉴权通过**全局过滤器**实现，集中处理身份验证和授权逻辑。
 **JWT鉴权过滤器**：
```java
@Component
public class JwtAuthenticationFilter implements GlobalFilter, Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = extractToken(exchange.getRequest());
        
        if (StringUtils.isEmpty(token)) {
            return unauthorized(exchange, "Missing token");
        }
        
        return validateToken(token)
            .flatMap(valid -> {
                if (valid) {
                    return chain.filter(exchange);
                } else {
                    return unauthorized(exchange, "Invalid token");
                }
            });
    }
    
    private Mono<Boolean> validateToken(String token) {
        // JWT验证逻辑
        return Mono.just(true);
    }
}
```
 **基于角色的访问控制**：
```java
@Component
public class RoleAuthorizationFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();
        String method = exchange.getRequest().getMethodValue();
        
        // 从JWT中提取角色信息
        Set<String> roles = extractRoles(exchange);
        
        if (!hasPermission(path, method, roles)) {
            return forbidden(exchange, "Insufficient permissions");
        }
        
        return chain.filter(exchange);
    }
}
```
**白名单配置**：
```yaml
gateway:
  security:
    whitelist:
      - /api/public/**
      - /api/auth/login
      - /actuator/health
```
**最佳实践**：结合OAuth2、JWT等标准协议，实现安全的统一认证体系。
###### 22. Gateway 如何集成 Sentinel 实现限流降级？
Spring Cloud Alibaba Sentinel提供完善的Gateway集成方案。
 **依赖配置**：
```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
</dependency>
```
 **Sentinel配置**：
```yaml
spring:
  cloud:
    sentinel:
      filter:
        enabled: false  # 关闭默认Filter
      gateway:
        enabled: true
      transport:
        dashboard: localhost:8080
```
 **自定义流控规则**：
```java
@Configuration
public class SentinelConfig {
    
    @PostConstruct
    public void initRules() {
        // API分组限流
        Set<ApiDefinition> definitions = new HashSet<>();
        ApiDefinition api = new ApiDefinition("user_api")
            .setPredicateItems(new HashSet<>() {{
                add(new ApiPathPredicateItem().setPattern("/api/users/**"));
            }});
        definitions.add(api);
        
        // 流控规则
        List<GatewayFlowRule> rules = new ArrayList<>();
        rules.add(new GatewayFlowRule("user_api")
            .setCount(100)        // 阈值
            .setIntervalSec(1)    // 统计窗口
            .setBurst(50));       // 突发容量
        
        GatewayRuleManager.loadRules(rules);
    }
}
```
**自定义降级逻辑**：
```java
@Component
public class SentinelFallbackHandler {
    
    public Mono<Response> handleFallback(ServerWebExchange exchange, Throwable ex) {
        // 自定义降级响应
        return Mono.just(Response.fallback("Service temporary unavailable"));
    }
}
```
**核心功能**：Sentinel提供API级别的流控、熔断降级、系统自适应保护等能力。
###### 23. Gateway 的性能优化方法有哪些？
**底层通信优化**
- **使用Netty原生传输**：开启Native传输提升性能
- **连接池优化**：配置合适的连接池参数
```yaml
spring:
  cloud:
    gateway:
      httpclient:
        pool:
          max-connections: 1000
          acquire-timeout: 20000
```
**JVM优化**
- **内存设置**：合理分配堆内存和直接内存
- **GC调优**：使用G1垃圾收集器，减少GC停顿
**路由配置优化**
- **减少过滤器链长度**：移除不必要的过滤器
- **合理使用缓存**：对静态资源启用缓存
- **异步处理**：避免阻塞操作，使用响应式编程
**监控与调优**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: metrics,gateway
  endpoint:
    metrics:
      enabled: true
    gateway:
      enabled: true
```
###### 24. 如何设计一套 API 接口？
**RESTful设计原则**
- **资源导向**：使用名词而非动词，如`/users`而非`/getUsers`
- **HTTP方法语义化**：GET(查询)、POST(创建)、PUT(更新)、DELETE(删除)
- **状态码标准化**：200(成功)、201(创建)、400(参数错误)、404(不存在)
**版本管理策略**
```yaml
# URI路径版本控制
spring:
  cloud:
    gateway:
      routes:
        - id: api_v1
          uri: lb://api-service
          predicates:
            - Path=/api/v1/**
        - id: api_v2  
          uri: lb://api-service
          predicates:
            - Path=/api/v2/**
```
 **统一响应格式**
```java
public class ApiResponse<T> {
    private int code;
    private String message;
    private T data;
    private long timestamp;
    
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(200, "success", data, System.currentTimeMillis());
    }
}
```
**API文档化**
- 集成Swagger/OpenAPI3.0
- 提供完整的接口文档和示例
###### 25. API 网关的安全设计有哪些方面？
 **多层次安全防护**
1. **网络层安全**
    - TLS/SSL终端加密
    - IP白名单限制
    - DDoS防护
2. **身份认证**
    - OAuth2.0/OpenID Connect
    - JWT令牌验证
    - API密钥认证
3. **授权控制**
    - RBAC基于角色的访问控制
    - ABAC基于属性的访问控制
    - 细粒度权限管理
4. **安全审计**
    - 完整请求日志记录
    - 异常行为监控告警
    - 安全事件追踪
 **具体安全配置**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: secure_api
          uri: lb://secure-service
          predicates:
            - Path=/api/secure/**
          filters:
            - AuthenticationFilter
            - AuthorizationFilter
            - RateLimitFilter
            - AuditLogFilter
```