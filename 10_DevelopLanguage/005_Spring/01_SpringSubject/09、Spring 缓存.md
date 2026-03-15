###### 1. Spring 如何实现缓存？

Spring 缓存的核心实现基于 **AOP** 和**缓存抽象层**两大机制。思路很简单：用 AOP 拦截被缓存注解标注的方法调用，在方法执行前检查缓存是否命中，命中就直接返回缓存值，不命中才执行真正的方法并把结果存入缓存。这样业务代码和缓存逻辑完全解耦，开发者只需要关注注解配置。

整个机制围绕四个核心组件运转：**缓存操作源**（`AnnotationCacheOperationSource`，解析方法上的缓存注解）、**缓存拦截器**（`CacheInterceptor`，作为 AOP 通知执行缓存逻辑）、**缓存管理器**（`CacheManager`，管理 Cache 实例的生命周期）、**缓存实例**（`Cache`，真正执行 get/put/evict 操作）。

启用缓存只需要加一个注解：

```java
@Configuration
@EnableCaching
public class CacheConfig {
    // 配置 CacheManager Bean 即可
}
```

`@EnableCaching` 会触发导入 `ProxyCachingConfiguration`，注册切面顾问 `BeanFactoryCacheOperationSourceAdvisor`，为所有带缓存注解的方法创建代理。

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/02、Spring缓存机制#缓存机制原理]]

---

###### 2. @Cacheable 注解如何使用？

`@Cacheable` 用于声明方法的返回值应该被缓存，后续相同参数的调用直接返回缓存值，不执行方法本身。

核心属性有这几个：`value/cacheNames`（指定缓存名称）、`key`（自定义缓存键，支持 SpEL）、`condition`（执行前的条件，满足时才缓存）、`unless`（执行后的过滤，结果满足时不缓存）、`sync`（是否同步，防止缓存击穿）。

默认键生成规则是：无参数时用空键，单参数直接用参数值，多参数包装为 `SimpleKey`。自定义键用 SpEL 表达式非常灵活：

```java
@Service
public class UserService {
    
    @Cacheable("users")
    public User getUserById(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    // 自定义键，加条件：id 不为空才缓存，结果为 null 时不缓存
    @Cacheable(value = "users", 
               key = "#user.id + ':' + #user.type",
               condition = "#user.id != null",
               unless = "#result == null")
    public User getUserWithCondition(UserQuery user) {
        return userRepository.findByCondition(user);
    }
    
    // sync = true 防止缓存击穿：高并发时只有一个线程执行方法，其他线程等待
    @Cacheable(value = "hotData", key = "#id", sync = true)
    public HotData getHotData(String id) {
        return dataService.loadHotData(id);
    }
}
```

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/02、Spring缓存机制#Cacheable注解]]

---

###### 3. @CacheEvict 注解如何使用？

`@CacheEvict` 用于清除缓存条目，通常在数据更新或删除后使用，保证缓存和数据库的一致性。

几个关键属性：`allEntries = true` 表示清除整个缓存区的所有数据（比如刷新列表）；`beforeInvocation = true` 表示在方法执行前清除缓存，即使方法抛异常也照样清——默认是方法执行成功后才清。

```java
@Service
public class UserService {
    
    // 更新用户后，清除该用户的缓存
    @CacheEvict(value = "users", key = "#user.id")
    public void updateUser(User user) {
        userRepository.save(user);
    }
    
    // 清除整个 userList 缓存区（全量刷新场景）
    @CacheEvict(value = "userList", allEntries = true)
    public void refreshAllUsers() { }
    
    // 删除前先清缓存，即使删除失败也不会有脏缓存
    @CacheEvict(value = "users", key = "#id", beforeInvocation = true)
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
    
    // 组合清除多个缓存
    @Caching(evict = {
        @CacheEvict(value = "users", key = "#id"),
        @CacheEvict(value = "userStats", allEntries = true)
    })
    public void deleteUserAndCleanStats(Long id) {
        userRepository.deleteById(id);
    }
}
```

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/02、Spring缓存机制#CacheEvict注解]]

---

###### 4. @CachePut 注解如何使用？

`@CachePut` 用于**强制更新缓存**，和 `@Cacheable` 的区别很明确：`@Cacheable` 在缓存命中时不执行方法，`@CachePut` 无论缓存中有没有数据，都会执行方法，然后把结果写入缓存。

用途也很清晰：`@Cacheable` 适合查询，`@CachePut` 适合更新——更新完数据库，同时刷新缓存，保持数据一致。

```java
@Service
public class ProductService {
    
    // 更新产品并刷新缓存
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);  // 返回值会被写入缓存
    }
    
    // 条件更新：只有价格大于100的产品才更新缓存
    @CachePut(value = "products", 
              key = "#product.id",
              condition = "#product.price > 100",
              unless = "#result == null")
    public Product updateExpensiveProduct(Product product) {
        return productRepository.save(product);
    }
    
    // 同时更新两个缓存区
    @Caching(put = {
        @CachePut(value = "products", key = "#product.id"),
        @CachePut(value = "productDetails", key = "#product.sku")
    })
    public Product updateProductWithDetails(Product product) {
        return productRepository.save(product);
    }
}
```

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/02、Spring缓存机制#CachePut注解]]

---

###### 5. @Caching 注解如何使用？

`@Caching` 用于在同一个方法上组合多个缓存操作，结构很清晰：

```java
public @interface Caching {
    Cacheable[] cacheable() default {};
    CachePut[] put() default {};
    CacheEvict[] evict() default {};
}
```

适合业务操作比较复杂、需要同时操作多个缓存的场景。比如创建订单后，要更新订单详情缓存、更新用户订单列表缓存、同时清除订单统计缓存：

```java
@Service
public class OrderService {
    
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
    
    // 用户停用：清理所有相关缓存
    @Caching(evict = {
        @CacheEvict(value = "users", key = "#user.id"),
        @CacheEvict(value = "userProfiles", key = "#user.id"),
        @CacheEvict(value = "userPermissions", key = "#user.id"),
        @CacheEvict(value = "userSessions", allEntries = true)
    })
    public void deactivateUser(User user) {
        user.setActive(false);
        userRepository.save(user);
    }
}
```

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/02、Spring缓存机制#Caching组合注解]]

---

###### 6. 如何配置缓存管理器？

缓存管理器是 Spring 缓存的核心配置组件，通过 `CacheManager` 接口抽象管理所有 `Cache` 实例。

**Caffeine（推荐，生产环境首选）**：高性能本地缓存，支持 LRU/LFU 淘汰策略、基于写入时间和访问时间的过期、最大容量控制：

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
                .maximumSize(1000)
                .expireAfterWrite(1, TimeUnit.HOURS)
                .expireAfterAccess(10, TimeUnit.MINUTES)
                .recordStats());
        manager.setCacheNames(Arrays.asList("users", "products", "orders"));
        return manager;
    }
}
```

**Redis（分布式环境）**：多实例共享缓存，可设置不同缓存区的过期时间：

```java
@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
    RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
    
    return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withCacheConfiguration("users", config.entryTtl(Duration.ofMinutes(30)))
            .build();
}
```

**多级缓存**：Caffeine 做一级（本地热点）+ Redis 做二级（分布式共享），用 `CompositeCacheManager` 组合，先查本地，没有再查 Redis：

```java
@Bean
public CacheManager compositeCacheManager(
        CacheManager localCacheManager, CacheManager redisCacheManager) {
    CompositeCacheManager composite = new CompositeCacheManager();
    composite.setCacheManagers(Arrays.asList(localCacheManager, redisCacheManager));
    return composite;
}
```

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/02、Spring缓存机制#CacheManager配置]]

---

###### 7. Spring 支持哪些缓存实现？

Spring 通过统一的 `CacheManager` 抽象层支持多种缓存技术，切换时业务代码不需要改动。

主流选择有四种：**Caffeine** 性能最高，W-TinyLFU 算法接近最优淘汰效果，是生产环境本地缓存首选；**Redis** 支持分布式、持久化，适合多实例共享数据的场景；**Ehcache** 功能全面，支持磁盘持久化和集群，适合复杂单机场景；**ConcurrentMap** 就是简单的 `ConcurrentHashMap`，适合开发测试用，不推荐上生产（无淘汰策略、无过期时间）。

实际项目中常见的搭配是：热点数据用 Caffeine 本地缓存，共享数据用 Redis，用 `@Cacheable(cacheManager = "localCacheManager")` 指定具体的缓存管理器来分别控制。

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/02、Spring缓存机制#缓存实现选型]]

---

###### 8. 缓存的过期策略如何设置？

过期策略直接影响缓存命中率和数据新鲜度，需要根据数据特性来配置。

**Caffeine** 支持三种过期维度：基于写入时间（`expireAfterWrite`）、基于访问时间（`expireAfterAccess`）、基于容量（`maximumSize`）。还可以自定义权重淘汰，按对象大小来控制缓存占用。

**Redis** 通过 `entryTtl` 设置每个缓存区的 TTL，可以针对不同缓存区分别配置：

```java
@Bean
public RedisCacheManagerBuilderCustomizer redisCacheManagerBuilderCustomizer() {
    return builder -> builder
            .withCacheConfiguration("shortTermCache",
                RedisCacheConfiguration.defaultCacheConfig()
                        .entryTtl(Duration.ofMinutes(10)))
            .withCacheConfiguration("longTermCache",
                RedisCacheConfiguration.defaultCacheConfig()
                        .entryTtl(Duration.ofDays(1)))
            .withCacheConfiguration("eternalCache",
                RedisCacheConfiguration.defaultCacheConfig()
                        .entryTtl(Duration.ZERO));  // 不过期
}
```

过期时间的选择原则：数据变化频繁（比如库存、价格）设置短 TTL 或用 `@CacheEvict` 主动失效；数据相对稳定（比如商品分类、系统配置）可以设置较长 TTL；基础字典数据可以设置很长 TTL 甚至不过期，靠管理后台手动刷新。

另外注意**缓存雪崩**的防范：不要给所有缓存设置相同的过期时间，加个随机偏移量（如 TTL + random(0, 60s)），避免大批缓存同时失效打爆数据库。

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/02、Spring缓存机制#过期策略配置]]
