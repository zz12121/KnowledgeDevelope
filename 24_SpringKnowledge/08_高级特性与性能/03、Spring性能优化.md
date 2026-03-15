# Spring 性能优化

Spring 应用的性能优化需要从多个维度入手：启动速度、Bean 管理、AOP 效率、数据库访问、连接池配置。每个环节都有明确的优化手段，不是玄学。

---

## 一、启动速度优化

Spring Boot 应用启动慢是很多人遇到的问题，主要原因是启动时要扫描大量类、初始化大量 Bean。

### 1. 启用全局懒加载

最简单粗暴的方式：不用的 Bean 先别初始化，等第一次使用时再创建：

```yaml
spring:
  main:
    lazy-initialization: true
```

对于必须在启动时就初始化的 Bean（比如消息监听器、定时任务、数据库连接池），用 `@Lazy(false)` 强制覆盖：

```java
@Component
@Lazy(false)  // 强制启动时初始化
public class MessageQueueListener {
    // 监听器必须启动时就注册
}
```

懒加载开启后，首次请求的响应时间会变长，生产环境通常配合服务预热来解决。

---

### 2. 精准组件扫描

避免大范围扫描，把扫描路径缩小到核心包：

```java
@SpringBootApplication(scanBasePackages = "com.example.core")
@ComponentScan(excludeFilters = {
    @Filter(type = FilterType.REGEX, pattern = "com.example.test.*"),
    @Filter(type = FilterType.ANNOTATION, classes = TestComponent.class)
})
public class Application { }
```

---

### 3. 排除不需要的自动配置

Spring Boot 会自动配置很多东西，用不上的配置也会消耗时间：

```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,       // 不需要数据库
    WebMvcAutoConfiguration.class,           // 非 Web 应用
    HibernateJpaAutoConfiguration.class
})
public class Application { }
```

---

### 4. JVM 层优化（开发环境）

```bash
# 禁用 JIT 分层编译，降低启动时的编译开销（仅开发环境）
java -XX:TieredStopAtLevel=1 -jar app.jar
```

注意：这个参数会降低运行时性能，只适合开发/测试环境用于快速迭代。

---

### 5. 启动分析

用 Spring Boot Actuator 的 startup 端点找出启动慢的 Bean：

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

启动后访问 `/actuator/startup`，可以看到每个 Bean 的初始化时间，针对性优化。

---

## 二、Bean 创建优化

### 1. 优先使用构造器注入

构造器注入在 Bean 创建时一次性完成所有依赖解析，而且天然支持不可变设计：

```java
@Service
public class OrderService {
    private final UserRepository userRepo;    // final 保证不可变
    private final ProductService productSvc;
    
    // Spring 4.3+ 单构造器可以省略 @Autowired
    public OrderService(UserRepository userRepo, ProductService productSvc) {
        this.userRepo = userRepo;
        this.productSvc = productSvc;
    }
}
```

---

### 2. 合理使用 @Lazy

对于重量级的 Bean（大型缓存、很少用到的功能模块），加 `@Lazy` 延迟初始化，降低启动时的内存峰值：

```java
@Autowired
@Lazy
private HeavyReportService reportService; // 只有第一次用到时才初始化
```

---

### 3. 用 @Conditional 避免不必要的 Bean

根据条件决定是否创建 Bean，不需要的功能就不要初始化：

```java
@Bean
@ConditionalOnProperty(name = "feature.payment.enabled", havingValue = "true")
@ConditionalOnClass(PaymentClient.class)
public PaymentService paymentService() {
    return new PaymentService();
}
```

---

### 4. 避免循环依赖

循环依赖会导致容器多次尝试实例化 Bean，增加创建开销。更根本的问题是它往往意味着设计上的职责划分有问题，重构是根本解法。

---

## 三、AOP 性能优化

### 1. 精准切点表达式

切点越宽泛，Spring 需要检查的方法越多，匹配开销越大：

```java
@Aspect
@Component
public class LoggingAspect {
    
    // 不推荐：太宽泛
    // @Pointcut("execution(* com.example..*(..))")
    
    // 推荐：精确到 service 层的 public 方法
    @Pointcut("execution(public * com.example.service.*Service.*(..))")
    public void serviceMethods() {}
    
    @Around("serviceMethods()")
    public Object logMethod(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            long duration = System.currentTimeMillis() - start;
            if (duration > 200) {
                log.warn("慢方法: {} 耗时 {}ms", pjp.getSignature(), duration);
            }
        }
    }
}
```

---

### 2. 选择合适的通知类型

不同通知类型性能开销不同：

- `@Around`：功能最全，但开销最大（需要控制流程时才用）
- `@Before` / `@After`：简单日志、审计场景首选
- `@AfterReturning` / `@AfterThrowing`：针对性处理，开销最小

---

### 3. 通知里加条件判断

避免在每次方法调用时都执行昂贵的操作：

```java
@Before("serviceMethods()")
public void logBefore(JoinPoint jp) {
    if (log.isDebugEnabled()) {  // 条件检查，非 debug 级别时直接跳过
        log.debug("调用方法: {}", jp.getSignature().getName());
    }
}
```

---

### 4. 代理方式选择

```java
// 强制使用 CGLIB，对于没有接口的类能减少代理创建时的判断开销
@SpringBootApplication
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class Application { }
```

---

## 四、数据库访问优化

### 1. 连接池精细调优

HikariCP 的推荐配置（最常用的连接池）：

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20         # 最大连接数：(核心数 × 2) + 磁盘数，常规配置 10-20
      minimum-idle: 5               # 最小空闲连接
      connection-timeout: 30000     # 获取连接超时 30s
      idle-timeout: 300000          # 空闲连接超时 5min
      max-lifetime: 1800000         # 连接最大生命周期 30min（要比数据库 wait_timeout 小）
      keepalive-time: 30000         # 心跳检测间隔 30s
      leak-detection-threshold: 60000  # 连接泄漏检测阈值 1min
```

**连接池大小口诀**：连接数不是越多越好，公式是 `connections = CPU 核数 × 2 + 磁盘数`，对于纯内存数据库约 `CPU 核数 × 2 + 1`。连接太多反而会因为上下文切换拖慢性能。

---

### 2. N+1 查询问题

最常见的数据库性能杀手。比如查询 100 个订单，每个订单再查一次用户信息，结果变成 101 次查询：

```java
// 问题代码：N+1 查询
List<Order> orders = orderRepo.findAll();
for (Order order : orders) {
    System.out.println(order.getCustomer().getName()); // 每次都触发一次 SQL
}

// 解决：JOIN FETCH 一次查回来
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :status")
List<Order> findOrdersWithCustomer(@Param("status") String status);
```

---

### 3. 批量操作优化

避免在循环里一条一条 insert/update：

```java
@Transactional
public void batchInsert(List<Entity> entities) {
    for (int i = 0; i < entities.size(); i++) {
        em.persist(entities.get(i));
        if (i % 50 == 0) {    // 每 50 条 flush 一次
            em.flush();
            em.clear();        // 清理一级缓存，防止内存溢出
        }
    }
}
```

JdbcTemplate 的批量操作更高效：

```java
jdbcTemplate.batchUpdate(
    "INSERT INTO users (name, email) VALUES (?, ?)",
    users,
    100,  // 批次大小
    (ps, user) -> {
        ps.setString(1, user.getName());
        ps.setString(2, user.getEmail());
    }
);
```

---

### 4. 合理使用缓存

频繁查询且不常变化的数据，用 Spring Cache 缓存起来，减少数据库压力：

```java
// 用户基础信息变化不频繁，缓存 30 分钟
@Cacheable(value = "users", key = "#id")
public User getUserById(Long id) {
    return userRepository.findById(id).orElse(null);
}
```

---

## 五、性能监控与分析

性能优化要有数据支撑，不要凭感觉优化。常用工具：

**Spring Boot Actuator**：暴露 `/actuator/metrics`、`/actuator/health`、`/actuator/httptrace` 等端点，监控 JVM 内存、GC、HTTP 请求延迟、连接池状态等。

**Micrometer + Prometheus**：将指标推送到 Prometheus，再用 Grafana 做可视化。

**Arthas**（阿里巴巴）：生产环境动态诊断工具，可以在不重启的情况下查看方法执行时间、调用链路等，非常实用。

---

## 相关面试题 →

- [[../../10_Developlanguage/005_Spring/01_SpringSubject/13、Spring 性能优化]]
