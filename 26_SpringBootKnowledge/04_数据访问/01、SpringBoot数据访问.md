# SpringBoot 数据访问

## 一、MyBatis 集成

Spring Boot 通过 `mybatis-spring-boot-starter` 提供开箱即用的 MyBatis 集成，核心是自动配置机制。

**自动配置原理：**

`MybatisAutoConfiguration` 通过 `@ConditionalOnClass` 检测类路径下存在 `SqlSessionFactory` 和 `SqlSessionFactoryBean` 时自动生效，在 `DataSourceAutoConfiguration` 之后初始化，自动注册 `SqlSessionFactory` 和 `SqlSessionTemplate`。

**集成步骤：**

1. 引入依赖 `mybatis-spring-boot-starter`
2. 配置数据源和 MyBatis 参数（mapper-locations、type-aliases-package、驼峰命名映射等）
3. Mapper 扫描：启动类加 `@MapperScan("com.example.mapper")` 或各 Mapper 接口单独加 `@Mapper`

**关键配置项：**

```yaml
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.example.entity
  configuration:
    map-underscore-to-camel-case: true  # 驼峰命名映射
    cache-enabled: true                  # 开启二级缓存
```

---

## 二、JPA 集成

Spring Data JPA 通过 Repository 抽象层极大简化数据访问代码，底层基于 Hibernate 实现。

**自动配置核心：**

`JpaRepositoriesAutoConfiguration` 检测到 `EntityManagerFactory` 和 `PlatformTransactionManager` 时自动激活，完成实体管理器工厂和事务管理器的配置。

**核心特性：**

- **方法名推导查询**：`findByUsernameContaining(String keyword)` 自动生成 `LIKE` 查询
- **分页支持**：`findByAgeGreaterThan(int age, Pageable pageable)` 内置分页
- **自定义 JPQL**：`@Query("SELECT u FROM User u WHERE u.email LIKE %:email%")`
- **实体关联**：`@OneToMany`、`@ManyToOne`、`@ManyToMany`，配合 `cascade` 和 `fetch` 策略

**JPA 配置：**

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update   # 开发环境，生产建议 validate
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect
```

---

## 三、MyBatis vs JPA 对比

| 维度 | MyBatis | Spring Data JPA |
|------|---------|-----------------|
| 设计理念 | SQL 映射，SQL 可控性优先 | ORM，面向对象优先 |
| SQL 控制 | 完全掌控，手动编写 | 自动生成，@Query 可扩展 |
| 开发效率 | CRUD 需手动编码，复杂 SQL 高效 | 简单 CRUD 零编码，派生查询高效 |
| 性能调优 | 直接优化 SQL，精准控制 | 需理解 Hibernate 缓存、懒加载 |
| 适用场景 | 复杂报表、高性能、遗留系统 | 快速原型、DDD、标准 CRUD |

**源码架构差异：**

- **MyBatis**：`SqlSessionTemplate` 管理会话，`MapperProxy` 实现接口动态代理
- **JPA**：基于 `EntityManager` 持久化上下文，`SimpleJpaRepository` 提供默认实现

**选择建议**：需要精细控制 SQL、处理复杂查询 → MyBatis；追求开发效率、对象模型复杂 → JPA。

---

## 四、多数据源配置

多数据源需要手动定义多个 DataSource Bean，并明确指定各自的 EntityManagerFactory 和 TransactionManager。

**配置要点：**

1. 主数据源加 `@Primary` 注解，作为默认数据源
2. 每个数据源对应独立的 `@EnableJpaRepositories` 配置，指定 basePackages、entityManagerFactoryRef、transactionManagerRef
3. 配置文件按前缀区分：`spring.datasource.primary.*` / `spring.datasource.secondary.*`

**注意事项：**

- 多数据源环境下 Spring Boot 自动配置会失效，需要完全手动配置
- `@Primary` 标注主数据源，避免注入歧义
- 每个数据源需要独立的 `PlatformTransactionManager`，不能共用

---

## 五、数据库连接池

### HikariCP（默认）

Spring Boot 2.x 及以上默认使用 HikariCP，因其高性能和轻量级特性。

**自动配置：** `DataSourceConfiguration.Hikari` 在检测到 `HikariDataSource.class` 且未指定其他类型时自动生效。

**性能优势：**

- **无锁设计**：`ConcurrentBag` 结合 ThreadLocal 和 CopyOnWriteArrayList，减少锁竞争
- **字节码优化**：Javassist 字节码增强关键路径，减少动态分派开销
- **智能清理**：自动回收泄漏连接，`leak-detection-threshold` 可配置检测阈值

**推荐配置：**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20         # CPU 核心数 × 2 + 1
      minimum-idle: 5
      connection-timeout: 30000     # 获取连接超时 30s
      idle-timeout: 600000          # 空闲超时 10min
      max-lifetime: 1800000         # 最大生命周期 30min
      leak-detection-threshold: 60000  # 泄漏检测 60s
```

### 其他连接池

- **Druid**（阿里）：监控能力强，SQL 统计、慢查询分析、防 SQL 注入，适合生产环境监控
- **DBCP2**：Apache 出品，功能全面但性能不如 HikariCP

---

## 六、读写分离

读写分离通过继承 `AbstractRoutingDataSource` 实现，根据操作类型动态路由到主从数据库。

**实现核心：**

```java
// 1. 线程上下文持有路由键
public class DatabaseContextHolder {
    private static final ThreadLocal<DatabaseType> context = new ThreadLocal<>();
    // MASTER / SLAVE
}

// 2. 动态路由数据源
public class ReadWriteRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return DatabaseContextHolder.getDatabaseType();
    }
}

// 3. AOP 切面自动切换
@Around("@annotation(readOnly)")
public Object routeDataSource(ProceedingJoinPoint point, ReadOnly readOnly) {
    try {
        DatabaseContextHolder.setDatabaseType(DatabaseType.SLAVE);
        return point.proceed();
    } finally {
        DatabaseContextHolder.clear();  // 必须清理，防止线程复用污染
    }
}
```

**注意事项：**

- `finally` 块必须清理 ThreadLocal，否则线程池复用会导致路由错误
- 事务内不应切换数据源，`@Transactional` 已绑定连接
- 从库延迟问题：写后立即读可能读到旧数据，此时强制走主库

---

## 七、Redis 集成与缓存

### Redis 集成

Spring Boot 通过 `spring-boot-starter-data-redis` 提供自动配置，默认使用 Lettuce 客户端（响应式、线程安全）。

`RedisAutoConfiguration` 自动注册 `RedisTemplate<Object, Object>` 和 `StringRedisTemplate`，通常需要自定义序列化方式（默认 JDK 序列化不可读，推荐 Jackson2JsonRedisSerializer）。

### Spring Cache 缓存注解

| 注解 | 执行时机 | 使用场景 |
|------|----------|----------|
| `@Cacheable` | 方法执行前检查缓存，命中直接返回 | 查询操作 |
| `@CachePut` | 方法执行后更新缓存，始终执行方法 | 增改操作 |
| `@CacheEvict` | 方法执行后清理缓存 | 删除操作 |
| `@Caching` | 组合多个缓存操作 | 复杂场景 |

**底层机制：** `CacheInterceptor` 拦截注解方法，调用 `CacheAspectSupport` 中的对应方法实现缓存逻辑。

**高级用法：**

```java
// 条件缓存
@Cacheable(value = "users", key = "#id", condition = "#id > 100")

// 清理整个缓存空间
@CacheEvict(value = "users", allEntries = true)

// 组合注解：先清列表缓存，再更新单个缓存
@Caching(
    evict = {@CacheEvict(value = "user-list", allEntries = true)},
    put  = {@CachePut(value = "users", key = "#user.id")}
)
```

**缓存配置：**

```java
@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
    RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(30))
        .disableCachingNullValues();  // 防止缓存穿透
    return RedisCacheManager.builder(factory).cacheDefaults(config).build();
}
```

---

## 八、事务管理

### @Transactional 核心属性

| 属性 | 说明 |
|------|------|
| `propagation` | 传播行为，默认 REQUIRED |
| `isolation` | 隔离级别，默认 DEFAULT（跟随数据库） |
| `rollbackFor` | 触发回滚的异常类型，默认 RuntimeException |
| `readOnly` | 只读事务，数据库可优化执行计划 |
| `timeout` | 事务超时秒数 |

### 七种传播行为

| 传播行为 | 含义 |
|----------|------|
| `REQUIRED`（默认）| 有事务则加入，无则新建 |
| `SUPPORTS` | 有事务则加入，无则非事务执行 |
| `MANDATORY` | 有事务则加入，无则抛异常 |
| `REQUIRES_NEW` | 无论如何新建事务，挂起当前事务 |
| `NOT_SUPPORTED` | 非事务执行，挂起当前事务 |
| `NEVER` | 非事务执行，有事务则抛异常 |
| `NESTED` | 嵌套事务，外层回滚则内层也回滚 |

### 事务失效场景

1. **同类内部调用**：`this.method()` 绕过代理，AOP 不生效 → 注入自身或抽取到另一个 Bean
2. **非 public 方法**：Spring AOP 只代理 public 方法
3. **异常被捕获**：方法内 catch 后未重新抛出，事务感知不到异常
4. **异常类型不匹配**：默认只回滚 `RuntimeException`，受检异常需手动指定 `rollbackFor`
5. **数据库不支持事务**：如 MySQL MyISAM 引擎

### 分布式事务

| 方案 | 特点 |
|------|------|
| **Seata AT 模式** | 自动补偿，无业务侵入，适合强一致性场景 |
| **TCC** | 手动编写 try/confirm/cancel，灵活但侵入性强 |
| **Saga 模式** | 长事务补偿，适合微服务链路 |
| **消息事务** | 最终一致性，基于可靠消息（RocketMQ 事务消息） |

---

## 九、数据库迁移

### Flyway

- 通过版本化 SQL 脚本管理数据库变更
- 脚本命名规范：`V{version}__{description}.sql`（如 `V1__Create_user_table.sql`）
- 启动时自动检测并执行未执行的脚本，版本号严格递增

### Liquibase

- 支持 XML/YAML/JSON/SQL 格式的变更集
- 支持回滚（rollback）功能，Flyway 不支持
- ChangeSet 由 author + id 唯一标识

**选择建议：** 需要回滚能力选 Liquibase；纯 SQL 脚本管理、简单直观选 Flyway。

---

## 十、查询性能优化

1. **索引优化**：为查询条件、排序字段、关联字段建立合适索引，避免全表扫描
2. **分页优化**：深分页使用游标分页替代 LIMIT OFFSET；JPA 使用 `Slice` 替代 `Page` 避免 count 查询
3. **N+1 问题**：JPA 懒加载触发 N+1，使用 `@EntityGraph` 或 `JOIN FETCH` 一次性加载
4. **批量操作**：JPA `saveAll()` + `@Modifying @Query` 批量更新；MyBatis `foreach` 批量插入
5. **投影查询**：只查询需要的字段，避免 `SELECT *`；JPA 使用 DTO 投影或接口投影
6. **连接池调优**：合理配置 HikariCP 的 maximum-pool-size，过大反而增加调度开销
7. **读写分离**：读操作路由到从库，降低主库压力
8. **慢查询监控**：开启 `show-sql: true` + Druid 慢查询统计，定期分析执行计划

---

## 关联面试题

- 📝 [[10_Developlanguage/005_Spring/03_SpringBootSubject/05、数据访问与持久化]]
