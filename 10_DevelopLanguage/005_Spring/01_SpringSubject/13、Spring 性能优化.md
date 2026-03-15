###### 1. 如何优化 Spring 应用的启动速度？

Spring 应用启动速度优化主要从几个方向入手，组合使用效果最好。

**懒加载（首选）**：开启全局懒加载，大部分 Bean 推迟到首次使用时才创建，启动时的 Bean 实例化数量大幅减少：

```yaml
spring:
  main:
    lazy-initialization: true
```

对于必须立即启动的 Bean（定时任务、消息监听器），用 `@Lazy(false)` 覆盖全局设置，强制启动时初始化。

**精准组件扫描**：缩小扫描范围，减少 Spring 需要处理的类数量：

```java
@SpringBootApplication(scanBasePackages = "com.example.core")
@ComponentScan(excludeFilters = @Filter(pattern = "com.external.*"))
```

**排除不必要的自动配置**：项目里没用到数据库，就没必要让 `DataSourceAutoConfiguration` 去跑一遍：

```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
```

**JVM 参数调整**（开发/测试环境）：

```bash
java -XX:TieredStopAtLevel=1 -Xverify:none -jar app.jar
```

`-XX:TieredStopAtLevel=1` 限制 JIT 编译层级，牺牲运行期性能换取更快的启动速度，不推荐生产环境。

**分析启动耗时**：开启 Spring Boot Actuator 的 startup 端点，访问 `/actuator/startup` 查看每个步骤的耗时，精准定位瓶颈：

```yaml
management:
  endpoint:
    startup:
      enabled: true
  endpoints:
    web:
      exposure:
        include: startup
```

通过以上措施组合，典型 Spring Boot 应用启动时间可以优化 30%–50%。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/03、Spring性能优化#启动速度优化]]

---

###### 2. 如何减少 Bean 的创建开销？

Bean 创建的主要开销来自实例化、依赖解析和初始化三个阶段，分别都有优化空间。

**构造器注入优于字段注入**：构造器注入在 Bean 创建时一次性完成依赖解析，且字段用 `final` 修饰保证不可变，线程安全：

```java
@Service
public class OrderService {
    private final UserRepository userRepo;
    
    @Autowired
    public OrderService(UserRepository userRepo) {
        this.userRepo = userRepo;
    }
}
```

**避免循环依赖**：循环依赖会导致 Spring 容器多次处理 Bean（三级缓存中反复查找），最好从设计上消除它，拆分职责或引入中间服务。

**`@Conditional` 条件化装配**：根据条件决定是否创建 Bean，不需要的功能就不初始化：

```java
@Bean
@ConditionalOnProperty(name = "feature.sms.enabled", havingValue = "true")
public SmsService smsService() {
    return new SmsService();
}
```

**避免在 `@Configuration` 类中执行繁重逻辑**：`@Configuration` 类在容器启动时就会被处理，重逻辑（网络请求、文件加载）应该放在 `@PostConstruct` 里，或者用懒加载推迟到实际使用时。

**原型 Bean 谨慎使用**：`@Scope("prototype")` 的 Bean 每次 `getBean()` 都会创建新实例，高频场景下开销很大。如果对象创建成本高，考虑对象池方案。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/03、Spring性能优化#Bean创建优化]]

---

###### 3. 懒加载如何提升性能？

懒加载通过**延迟资源初始化到实际使用时**来提升性能，主要体现在两方面：

**减少启动时间**：默认情况下，Spring 容器启动时会预实例化所有单例 Bean。如果你有 200 个 Bean，启动时就要创建 200 个对象，初始化它们的依赖、执行 `@PostConstruct` 方法，这些都需要时间。懒加载把这些工作推迟到首次请求时，启动时只做最少的初始化。

**减少内存峰值**：懒加载下，没有被用到的 Bean 压根不会创建，也不会占用内存，特别适合内存受限的容器化部署场景。

实现原理上，Spring 通过 `LazyInitTargetSource` 为懒加载的 Bean 创建代理对象（占位符），真正的 Bean 在首次方法调用时才通过 `getTarget()` 实例化。

```java
@Component
@Lazy
public class HeavyAnalyticsService {
    // 构造时需要加载大量数据，只有真正需要分析功能时才初始化
    public HeavyAnalyticsService() {
        log.info("Analytics service initializing - loading data...");
    }
}
```

**注意事项**：懒加载的 Bean 在首次访问时会有额外的初始化延迟（可能触发链式依赖的初始化），对于关键路径上的服务，这个延迟不可接受。解决方法是应用启动后通过预热（Warm-up）机制提前触发这些 Bean 的初始化。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/03、Spring性能优化#懒加载机制]]

---

###### 4. 如何优化 AOP 性能？

AOP 的性能损耗主要来自两块：**代理创建开销**（启动时）和**方法拦截开销**（运行时）。优化策略如下：

**精准切点表达式**：切点匹配范围越窄，运行时匹配计算越快。宽泛的切点（如 `execution(* com.example..*(..))` 匹配所有方法）会让 Spring 在每次方法调用时都要计算是否匹配：

```java
@Aspect
@Component
public class OptimizedLoggingAspect {
    // 不推荐：execution(* com.example..*(..))
    // 推荐：精确到包和返回类型
    @Pointcut("execution(public String com.example.service.*Service.*(..))")
    public void serviceMethods() {}
    
    @Around("serviceMethods()")
    public Object logMethod(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            long duration = System.currentTimeMillis() - start;
            if (duration > 100) {
                log.warn("慢方法: {} 执行 {}ms", pjp.getSignature(), duration);
            }
        }
    }
}
```

**合理选择通知类型**：`@Around` 功能最强但开销最大（需要包装整个方法执行），如果只需要在方法前记录日志，用 `@Before` 就够了，不要一律上 `@Around`。

**切面内避免繁重操作**：切面代码在每次方法调用时都会执行，IO 操作（数据库查询、HTTP 请求）放在切面里会严重拖慢整个应用。

**条件化切面**：在切面里加轻量的条件判断，避免不必要的处理：

```java
@Around("serviceMethods()")
public Object conditionalAdvice(ProceedingJoinPoint pjp) throws Throwable {
    if (!log.isDebugEnabled()) {  // 不开调试日志时直接跳过
        return pjp.proceed();
    }
    // 详细的调试日志逻辑
}
```

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/03、Spring性能优化#AOP性能优化]]

---

###### 5. 如何优化数据库访问性能？

数据库访问往往是 Spring 应用的性能瓶颈，从几个维度综合优化效果最好。

**连接池配置优化**：连接数太少导致等待，太多给数据库造成压力：

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 3000
      max-lifetime: 300000
      idle-timeout: 300000
```

连接池大小的经验公式：`连接数 = (CPU核数 × 2) + 磁盘数`。IO 密集型查询可以适当放大，但超过数据库最大连接数限制就没意义了。

**解决 N+1 问题**：这是 ORM 框架最常见的性能陷阱。循环内查询关联对象，1 次查询主数据 + N 次查询关联数据，随着数据量增长性能急剧下降：

```java
// 不推荐：N+1 查询
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    order.getCustomer();  // 每个订单都触发一次查询
}

// 推荐：JOIN FETCH 一次获取
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.id IN :ids")
List<Order> findOrdersWithCustomers(@Param("ids") List<Long> ids);
```

**批量操作优化**：循环单条 insert/update 性能很差，改用批量操作：

```java
@Transactional
public void batchInsert(List<Entity> entities) {
    for (int i = 0; i < entities.size(); i++) {
        em.persist(entities.get(i));
        if (i % 50 == 0) {
            em.flush();
            em.clear();  // 清理一级缓存，避免 OOM
        }
    }
}
```

**查询只取必要字段**：用投影（`interface` 或 `DTO` 方式）替代直接查实体，减少数据库传输量和对象映射开销，特别是对有大字段（TEXT/BLOB）的表。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/03、Spring性能优化#数据库访问优化]]

---

###### 6. 连接池如何配置？

HikariCP 是 Spring Boot 默认连接池，也是目前性能最好的 Java 连接池。核心参数说明：

```yaml
spring:
  datasource:
    hikari:
      # 池大小
      maximum-pool-size: 20        # 最大连接数，通常 = CPU核数*2 + 磁盘数
      minimum-idle: 5              # 最小空闲连接
      
      # 超时控制
      connection-timeout: 30000    # 获取连接的最大等待时间（30秒），超时抛异常
      validation-timeout: 5000     # 验证连接有效性的超时
      max-lifetime: 300000         # 连接最大存活时间（5分钟），避免数据库侧超时关闭
      idle-timeout: 300000         # 空闲连接超时，超时回收
      
      # 健康与监控
      keepalive-time: 30000        # 保持连接存活的心跳间隔
      leak-detection-threshold: 60000  # 连接泄漏检测阈值（超过1分钟未归还就告警）
```

几个容易踩坑的参数：

**`max-lifetime`** 必须小于数据库/防火墙的连接空闲超时配置，否则数据库侧主动关闭连接，HikariCP 还以为连接有效，拿到死连接会报错。

**`connection-timeout`** 是获取连接的等待时间，不是 SQL 执行超时。设太小（比如 1000ms）会导致流量高峰期频繁报"连接获取超时"。

**`minimum-idle`** 在高负载场景下可以设置为等于 `maximum-pool-size`，避免连接池伸缩带来的延迟；低负载场景设小一点，减少对数据库的长连接压力。

通过 Actuator 监控连接池状态，动态调整配置：`/actuator/metrics/hikaricp.connections` 可以看到当前活跃连接数、等待数等指标，是调优的重要依据。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/03、Spring性能优化#连接池配置调优]]
