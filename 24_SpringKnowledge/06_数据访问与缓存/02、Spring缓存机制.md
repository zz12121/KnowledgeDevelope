# Spring 缓存机制

Spring 缓存抽象层是框架中非常实用的一套机制，核心思路是：不管底层你用的是 Caffeine、Redis 还是 Ehcache，上层的使用方式都是统一的注解风格。这是典型的适配器设计思路，极大地降低了缓存切换的成本。

---

## 一、Spring 缓存核心原理

Spring 缓存基于 **AOP + 缓存抽象层** 两大支柱实现。整体架构用了模板方法模式和适配器模式。

启用 `@EnableCaching` 后，Spring 通过 `CachingConfigurationSelector` 导入 `ProxyCachingConfiguration`，这个配置类注册了三个核心 Bean：

- **`AnnotationCacheOperationSource`**：解析方法上的缓存注解（@Cacheable、@CachePut、@CacheEvict）
- **`CacheInterceptor`**：AOP 拦截器，真正执行缓存逻辑的地方
- **`BeanFactoryCacheOperationSourceAdvisor`**：切面顾问，把拦截器和操作源组合起来

当你调用一个被缓存注解修饰的方法时，`CacheInterceptor` 会拦截这次调用，根据注解类型决定是查缓存、写缓存还是删缓存，整个过程对业务代码完全透明。

---

## 二、四个核心缓存注解

### @Cacheable —— 查询缓存

`@Cacheable` 的逻辑很简单：先查缓存，命中就直接返回，不命中才执行方法，然后把结果放进缓存。

**核心属性：**

- **`value/cacheNames`**：缓存区域名称，比如 `"users"`、`"products"`
- **`key`**：缓存键，支持 SpEL 表达式，比如 `key = "#userId"` 或 `key = "#user.id + ':' + #user.type"`
- **`condition`**：方法执行前的条件判断，满足条件才启用缓存，比如 `condition = "#userId > 1000"`
- **`unless`**：方法执行后的结果过滤，满足条件就不缓存，比如 `unless = "#result == null"`
- **`sync`**：是否同步加载。设为 `true` 时，高并发场景下只有一个线程执行方法，其他线程等待，有效防止缓存击穿

```java
@Service
public class UserService {
    
    // 基础用法
    @Cacheable("users")
    public User getUserById(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    // 自定义键 + 条件控制
    @Cacheable(value = "users",
               key = "#user.id + ':' + #user.type",
               condition = "#user.id != null",
               unless = "#result == null")
    public User getUserWithCondition(UserQuery user) {
        return userRepository.findByCondition(user);
    }
    
    // 防缓存击穿：同步加载
    @Cacheable(value = "hotData", key = "#id", sync = true)
    public HotData getHotData(String id) {
        return dataService.loadHotData(id);
    }
}
```

Spring 默认使用 `SimpleKeyGenerator` 生成缓存键：没有参数时用 `SimpleKey.EMPTY`，一个参数时直接用参数本身，多个参数时包装成 `SimpleKey` 对象。

---

### @CacheEvict —— 删除缓存

数据更新或删除时，需要同步清除缓存，用 `@CacheEvict`。

**核心属性：**

- **`allEntries`**：是否清除整个缓存区域，默认 `false`（只清指定 key）
- **`beforeInvocation`**：在方法执行前还是后清除缓存。默认 `false`（方法执行后清），改为 `true` 后，即使方法抛异常也会清缓存，适合对数据一致性要求高的场景

```java
@Service
public class UserService {
    
    // 清除特定用户缓存
    @CacheEvict(value = "users", key = "#user.id")
    public void updateUser(User user) {
        userRepository.save(user);
    }
    
    // 清除整个缓存区域
    @CacheEvict(value = "userList", allEntries = true)
    public void refreshAllUsers() { }
    
    // 先清后执行（保证强一致性）
    @CacheEvict(value = "users", key = "#id", beforeInvocation = true)
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
}
```

---

### @CachePut —— 强制更新缓存

`@CachePut` 和 `@Cacheable` 的区别在于：它**无论如何都会执行方法**，然后把结果更新到缓存里。适合写操作场景，保证缓存和数据库同步。

```java
@Service
public class ProductService {
    
    // 更新产品信息，同时刷新缓存
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }
    
    // 缓存预热：启动时加载
    @PostConstruct
    @CachePut(value = "configs", key = "'systemConfig'")
    public SystemConfig preloadSystemConfig() {
        return configService.loadSystemConfig();
    }
}
```

---

### @Caching —— 组合多个缓存操作

一个方法上需要同时做多种缓存操作时，用 `@Caching` 来组合：

```java
@Service
public class OrderService {
    
    // 创建订单后：更新两个缓存 + 清除统计缓存
    @Caching(
        put = {
            @CachePut(value = "orders", key = "#order.id"),
            @CachePut(value = "userOrders", key = "#order.userId")
        },
        evict = {
            @CacheEvict(value = "orderStats", allEntries = true)
        }
    )
    public Order createOrder(Order order) {
        return orderRepository.save(order);
    }
}
```

---

## 三、缓存管理器配置

`CacheManager` 是 Spring 缓存的配置核心，负责创建和管理 `Cache` 实例。Spring 提供了多种内置实现：

| 管理器 | 用途 | 适用场景 |
|--------|------|----------|
| `ConcurrentMapCacheManager` | 内存缓存 | 测试/开发环境 |
| `CaffeineCacheManager` | Caffeine 集成 | 生产环境首选 |
| `RedisCacheManager` | Redis 集成 | 分布式/集群环境 |
| `EhCacheCacheManager` | Ehcache 集成 | 需要磁盘持久化的场景 |

**Caffeine 配置（生产环境推荐）：**

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
                .initialCapacity(100)
                .maximumSize(10000)
                .expireAfterWrite(30, TimeUnit.MINUTES)
                .expireAfterAccess(1, TimeUnit.HOURS)
                .recordStats()); // 开启命中率统计
        return manager;
    }
}
```

**Redis 缓存配置：**

```java
@Configuration
@EnableCaching
public class RedisCacheConfig {
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1))
                .disableCachingNullValues()
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .withCacheConfiguration("users", config.entryTtl(Duration.ofMinutes(30)))
                .transactionAware() // 支持事务
                .build();
    }
}
```

---

## 四、过期策略设计

合理的过期策略是缓存稳定性的基础，常见策略有：

**基于时间：**
- **TTL（写入后过期）**：Caffeine 的 `expireAfterWrite`，Redis 的 `entryTtl`
- **TTI（访问后过期）**：Caffeine 的 `expireAfterAccess`，冷数据会自动回收

**基于容量：**
- Caffeine 的 `maximumSize`：达到上限时按 LRU 或 W-TinyLFU 策略淘汰

**针对不同数据设置不同过期时间，是比较常见的实践：**

```java
@Bean
public RedisCacheManagerBuilderCustomizer redisCacheManagerBuilderCustomizer() {
    return builder -> builder
            .withCacheConfiguration("hotspot",
                RedisCacheConfiguration.defaultCacheConfig()
                        .entryTtl(Duration.ofMinutes(5)))      // 热点数据短期
            .withCacheConfiguration("business",
                RedisCacheConfiguration.defaultCacheConfig()
                        .entryTtl(Duration.ofHours(1)))        // 业务数据中期
            .withCacheConfiguration("basicData",
                RedisCacheConfiguration.defaultCacheConfig()
                        .entryTtl(Duration.ofDays(1)))         // 基础数据长期
            .enableStatistics();
}
```

---

## 五、多级缓存架构

高并发场景下，通常会把本地缓存（Caffeine）和分布式缓存（Redis）组合使用：

- **一级缓存（本地）**：延迟极低，但只对单机有效，容量有限
- **二级缓存（分布式）**：集群共享，容量大，但有网络开销

```java
@Configuration
@EnableCaching
public class MultiLevelCacheConfig {
    
    @Primary
    @Bean("localCacheManager")
    public CacheManager localCacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
                .maximumSize(1000)
                .expireAfterWrite(5, TimeUnit.MINUTES));
        return manager;
    }
    
    @Bean("redisCacheManager")
    public CacheManager redisCacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory)
                .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                        .entryTtl(Duration.ofHours(1)))
                .build();
    }
}

// 业务代码中按需选择缓存层
@Service
public class StrategicCacheService {
    
    @Cacheable(value = "hotData", cacheManager = "localCacheManager")
    public HotData getHotData(String id) {
        return dataService.loadHotData(id);
    }
    
    @Cacheable(value = "sharedData", cacheManager = "redisCacheManager")
    public SharedData getSharedData(String key) {
        return sharedService.loadSharedData(key);
    }
}
```

---

## 六、注意事项

**自调用问题**：`@Cacheable` 基于 AOP 代理，同一个类内部的方法调用不经过代理，注解不生效。解决方式是把缓存方法放在独立的 Bean 里，或通过 `ApplicationContext` 取代理对象来调用。

**缓存穿透防护**：使用 `unless = "#result == null"` 之前要想清楚，如果允许缓存空值来防穿透，就不能加这个条件；`@Cacheable` 的 `sync = true` 可以防击穿，但同一时刻只有一个线程去查数据库，其他等待。

**事务一致性**：`RedisCacheManager.transactionAware()` 开启后，缓存写操作会跟随事务提交才真正执行，事务回滚时缓存操作也会回滚。

---

## 相关面试题 →

- [[../../10_Developlanguage/005_Spring/01_SpringSubject/09、Spring 缓存]]
