###### 1. Spring 如何实现缓存？
Spring缓存的核心实现基于**AOP（面向切面编程）**​ 和**缓存抽象层**两大机制。其架构设计采用模板方法模式和适配器模式，实现对多种缓存技术的统一支持。
 核心架构组件

|**组件**​|**职责**​|**关键实现类**​|
|---|---|---|
|**缓存操作源**​|解析方法上的缓存注解|`AnnotationCacheOperationSource`|
|**缓存拦截器**​|AOP通知，执行缓存逻辑|`CacheInterceptor`|
|**缓存管理器**​|管理Cache实例的生命周期|`CacheManager`|
|**缓存实例**​|具体缓存操作的执行者|`Cache`|
源码级执行流程
**1. AOP代理创建**
当使用`@EnableCaching`时，Spring通过`CachingConfigurationSelector`导入缓存配置类`ProxyCachingConfiguration`，该配置类会注册三个核心Bean：
```java
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyCachingConfiguration extends AbstractCachingConfiguration {
    
    @Bean(name = CacheManagementConfigUtils.CACHE_ADVISOR_BEAN_NAME)
    public BeanFactoryCacheOperationSourceAdvisor cacheAdvisor() {
        // 创建切面顾问
        BeanFactoryCacheOperationSourceAdvisor advisor = new BeanFactoryCacheOperationSourceAdvisor();
        advisor.setCacheOperationSource(cacheOperationSource());
        advisor.setAdvice(cacheInterceptor());
        return advisor;
    }
    
    @Bean
    public CacheOperationSource cacheOperationSource() {
        // 创建缓存操作源，负责解析注解
        return new AnnotationCacheOperationSource();
    }
    
    @Bean
    public CacheInterceptor cacheInterceptor() {
        // 创建缓存拦截器
        CacheInterceptor interceptor = new CacheInterceptor();
        interceptor.setCacheOperationSource(cacheOperationSource());
        return interceptor;
    }
}
```
**2. 方法拦截执行**
当调用被`@Cacheable`等注解修饰的方法时，`CacheInterceptor`会拦截调用：
```java
public class CacheInterceptor extends CacheAspectSupport implements MethodInterceptor {
    
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();
        CacheOperationInvoker invoker = () -> {
            try {
                return invocation.proceed(); // 执行原始方法
            } catch (Throwable ex) {
                throw new CacheOperationInvoker.ThrowableWrapper(ex);
            }
        };
        
        try {
            // 执行缓存逻辑
            return execute(invoker, invocation.getThis(), method, invocation.getArguments());
        } catch (CacheOperationInvoker.ThrowableWrapper th) {
            throw th.getOriginal();
        }
    }
}
```
**3. 缓存操作执行**
在`CacheAspectSupport.execute()`方法中，Spring会根据注解类型执行相应的缓存操作：
```java
protected Object execute(CacheOperationInvoker invoker, Object target, Method method, Object[] args) {
    // 获取缓存操作配置
    Collection<? extends CacheOperation> operations = getCacheOperationSource()
            .getCacheOperations(method, target.getClass());
    
    if (operations != null) {
        for (CacheOperation operation : operations) {
            if (operation instanceof CacheableOperation) {
                // 处理@Cacheable逻辑
                return executeCacheable((CacheableOperation) operation, invoker, target, method, args);
            } else if (operation instanceof CacheEvictOperation) {
                // 处理@CacheEvict逻辑
                executeCacheEvict((CacheEvictOperation) operation, target, method, args);
                return invoker.invoke();
            } else if (operation instanceof CachePutOperation) {
                // 处理@CachePut逻辑
                return executeCachePut((CachePutOperation) operation, invoker, target, method, args);
            }
        }
    }
    return invoker.invoke();
}
```
这种设计使得业务代码与缓存逻辑完全解耦，开发者只需关注注解配置即可。
###### 2. @Cacheable 注解如何使用？
`@Cacheable`用于声明方法的返回值应该被缓存，在后续调用时直接返回缓存值而不执行方法。
核心属性详解

|**属性**​|**作用**​|**示例**​|
|---|---|---|
|`value/cacheNames`|指定缓存名称|`@Cacheable("users")`|
|`key`|自定义缓存键|`key = "#userId"`|
|`condition`|执行前条件判断|`condition = "#userId > 1000"`|
|`unless`|执行后结果过滤|`unless = "#result == null"`|
|`sync`|是否同步缓存|`sync = true`（防缓存击穿）|
源码中的键生成机制
Spring使用`SimpleKeyGenerator`作为默认的键生成器：
```java
public class SimpleKeyGenerator implements KeyGenerator {
    @Override
    public Object generate(Object target, Method method, Object... params) {
        if (params.length == 0) {
            return SimpleKey.EMPTY; // 空参数返回固定键
        }
        if (params.length == 1) {
            Object param = params[0];
            if (param != null && !param.getClass().isArray()) {
                return param; // 单参数直接作为键
            }
        }
        return new SimpleKey(params); // 多参数包装为SimpleKey
    }
}
```
实战使用示例
```java
@Service
public class UserService {
    
    // 基本用法：缓存到"users"缓存区
    @Cacheable("users")
    public User getUserById(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    // 自定义键和条件
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
        // 高并发时只有一个线程会执行方法，其他线程等待
        return dataService.loadHotData(id);
    }
    
    // 使用SpEL表达式复杂键
    @Cacheable(value = "orders", 
               key = "T(java.util.UUID).randomUUID().toString() + ':' + #order.status")
    public Order findOrderByStatus(Order order) {
        return orderRepository.findByStatus(order.getStatus());
    }
}
```
条件缓存的高级用法
```java
@Cacheable(value = "products", 
           condition = "#category != null && #category.length() > 3",
           unless = "#result == null || #result.stock == 0")
public Product getProductByCategory(String category) {
    // 只有当category非空且长度大于3时才缓存
    // 且结果不为空且库存不为0时才实际缓存
    return productRepository.findByCategory(category);
}
```
这种灵活的键和条件机制使得缓存策略可以精确控制到方法级别。
###### 3. @CacheEvict 注解如何使用？
`@CacheEvict`用于清除缓存条目，通常在数据更新或删除操作时使用。
核心属性说明

|**属性**​|**作用**​|**默认值**​|
|---|---|---|
|`value/cacheNames`|指定要清除的缓存名称|必填|
|`key`|指定要清除的缓存键|方法参数推导|
|`allEntries`|是否清除整个缓存区|`false`|
|`beforeInvocation`|在方法执行前还是后清除|`false`（方法执行后）|
源码中的清除逻辑
在`CacheAspectSupport.executeCacheEvict()`方法中：
```java
protected void executeCacheEvict(CacheEvictOperation operation, Object target, 
                                Method method, Object[] args) {
    
    Collection<? extends Cache> caches = getCaches(operation, target, method, args);
    
    for (Cache cache : caches) {
        // 根据allEntries决定清除范围
        if (operation.isCacheWide()) {
            cache.clear(); // 清除整个缓存区
        } else {
            // 根据key清除特定条目
            Object key = generateKey(operation, target, method, args);
            cache.evict(key);
        }
    }
}
```
实战使用示例
```java
@Service
public class UserService {
    
    // 基本用法：根据id清除特定用户缓存
    @CacheEvict(value = "users", key = "#user.id")
    public void updateUser(User user) {
        userRepository.save(user);
    }
    
    // 清除整个缓存区（如用户列表缓存）
    @CacheEvict(value = "userList", allEntries = true)
    public void refreshAllUsers() {
        // 方法执行后清除userList缓存区的所有数据
    }
    
    // 方法执行前清除缓存（避免并发问题）
    @CacheEvict(value = "users", key = "#id", beforeInvocation = true)
    public void deleteUser(Long id) {
        // 在方法执行前就清除缓存，即使方法抛出异常也会清除
        userRepository.deleteById(id);
    }
    
    // 组合清除多个缓存
    @Caching(evict = {
        @CacheEvict(value = "users", key = "#id"),
        @CacheEvict(value = "userStats", allEntries = true)
    })
    public void deleteUserAndCleanStats(Long id) {
        userRepository.deleteById(id);
        statsService.cleanUserStats();
    }
}
```
批量清除策略
```java
@Service
public class BatchCacheService {
    
    // 定时任务清除过期缓存
    @Scheduled(fixedRate = 3600000) // 每小时执行一次
    @CacheEvict(value = {"cache1", "cache2"}, allEntries = true)
    public void clearExpiredCaches() {
        log.info("定期清理缓存完成");
    }
    
    // 基于条件的批量清除
    @CacheEvict(value = "products", 
                condition = "#category != null",
                key = "#category + ':*'") // 支持通配符清除
    public void evictProductsByCategory(String category) {
        // 清除特定分类下的所有产品缓存
    }
}
```
`@CacheEvict`确保了数据更新时缓存的一致性，是保持缓存有效性的关键注解。
###### 4. @CachePut 注解如何使用？
`@CachePut`用于强制更新缓存，无论缓存中是否存在相应数据，都会执行方法并将结果存入缓存。
与@Cacheable的区别

|**特性**​|**@CachePut**​|**@Cacheable**​|
|---|---|---|
|**方法执行**​|总是执行|缓存命中时不执行|
|**使用场景**​|数据更新操作|数据查询操作|
|**缓存时机**​|方法执行后更新|方法执行前检查|
源码中的更新逻辑
```java
protected Object executeCachePut(CachePutOperation operation, CacheOperationInvoker invoker,
                                Object target, Method method, Object[] args) {
    
    // 1. 总是执行原始方法
    Object result = invoker.invoke();
    
    // 2. 检查unless条件
    if (isConditionPassing(operation, result, target, method, args)) {
        // 3. 获取缓存并更新
        Collection<? extends Cache> caches = getCaches(operation, target, method, args);
        Object key = generateKey(operation, target, method, args);
        
        for (Cache cache : caches) {
            cache.put(key, result); // 强制更新缓存
        }
    }
    
    return result;
}
```
实战使用示例
```java
@Service
public class ProductService {
    
    // 基本用法：更新产品信息并刷新缓存
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        Product updated = productRepository.save(product);
        return updated; // 返回值会被缓存
    }
    
    // 条件更新：只有满足条件时才更新缓存
    @CachePut(value = "products", 
              key = "#product.id",
              condition = "#product.price > 100",
              unless = "#result == null")
    public Product updateExpensiveProduct(Product product) {
        // 只有价格大于100的产品才更新缓存
        // 且结果不为null时才实际缓存
        return productRepository.save(product);
    }
    
    // 组合使用：先更新数据库，再更新缓存
    @Caching(
        put = {
            @CachePut(value = "products", key = "#product.id"),
            @CachePut(value = "productDetails", key = "#product.sku")
        }
    )
    public Product updateProductWithDetails(Product product) {
        Product updated = productRepository.save(product);
        // 同时更新两个缓存区的数据
        return updated;
    }
}
```
特殊场景应用
```java
@Service
public class InventoryService {
    
    // 计算密集型任务结果缓存
    @CachePut(value = "calculations", key = "#inputData.hashCode()")
    public ComplexResult calculateComplexData(InputData inputData) {
        // 复杂的计算逻辑，结果需要缓存供后续使用
        return complexCalculator.process(inputData);
    }
    
    // 缓存预热：系统启动时加载数据
    @PostConstruct
    @CachePut(value = "configs", key = "'systemConfig'")
    public SystemConfig preloadSystemConfig() {
        log.info("系统启动时预加载配置信息到缓存");
        return configService.loadSystemConfig();
    }
}
```
`@CachePut`确保了缓存数据与数据源的一致性，特别适合写多读少的场景。
###### 5. @Caching 注解如何使用？
`@Caching`用于在同一个方法上组合多个缓存操作，解决复杂业务场景下的缓存需求。
组合注解的源码结构
```java
public @interface Caching {
    Cacheable[] cacheable() default {};  // 可缓存操作
    CachePut[] put() default {};        // 更新操作  
    CacheEvict[] evict() default {};    // 清除操作
}
```
复杂业务场景实战
```java
@Service
public class OrderService {
    
    // 场景1：下单后更新多个缓存
    @Caching(
        put = {
            @CachePut(value = "orders", key = "#order.id"),           // 缓存订单详情
            @CachePut(value = "userOrders", key = "#order.userId")    // 缓存用户订单列表
        },
        evict = {
            @CacheEvict(value = "orderStats", allEntries = true)      // 清除统计信息缓存
        }
    )
    public Order createOrder(Order order) {
        Order saved = orderRepository.save(order);
        // 创建订单后，需要更新多个相关缓存
        return saved;
    }
    
    // 场景2：复杂的缓存策略组合
    @Caching(
        cacheable = {
            @Cacheable(value = "products", key = "#productId", 
                      condition = "#productId != null")
        },
        evict = {
            @CacheEvict(value = "productList", allEntries = true,
                       condition = "#updateInventory == true")
        }
    )
    public Product getProductWithInventory(String productId, boolean updateInventory) {
        Product product = productRepository.findById(productId);
        if (updateInventory) {
            inventoryService.updateInventory(productId);
        }
        return product;
    }
    
    // 场景3：用户信息全方位管理
    @Caching(
        evict = {
            @CacheEvict(value = "users", key = "#user.id"),           // 清除用户基本信息
            @CacheEvict(value = "userProfiles", key = "#user.id"),    // 清除用户档案
            @CacheEvict(value = "userPermissions", key = "#user.id"), // 清除权限信息
            @CacheEvict(value = "userSessions", allEntries = true)    // 清除所有会话
        }
    )
    public void deactivateUser(User user) {
        user.setActive(false);
        userRepository.save(user);
        // 用户停用时需要清理所有相关缓存
    }
}
```
条件组合的高级用法
```java
@Service
public class AdvancedCacheService {
    
    // 基于条件的复杂缓存组合
    @Caching(
        cacheable = {
            @Cacheable(value = "primaryCache", key = "#id", 
                      condition = "#id < 1000")
        },
        put = {
            @CachePut(value = "secondaryCache", key = "#id",
                     condition = "#result != null && #result.status == 'ACTIVE'")
        },
        evict = {
            @CacheEvict(value = "legacyCache", key = "#id",
                       condition = "#id > 5000")
        }
    )
    public DataEntity getDataWithComplexCaching(Long id) {
        // 根据不同的条件执行不同的缓存策略
        return dataRepository.findById(id);
    }
}
```
`@Caching`注解提供了极大的灵活性，能够应对企业级应用中的复杂缓存需求。
###### 6. 如何配置缓存管理器？
缓存管理器是Spring缓存的核心配置组件，负责创建和管理Cache实例。
内置缓存管理器实现

|**管理器**​|**用途**​|**特点**​|
|---|---|---|
|`ConcurrentMapCacheManager`|内存缓存|基于ConcurrentHashMap，适用于测试|
|`EhCacheCacheManager`|Ehcache集成|功能丰富，支持磁盘持久化|
|`RedisCacheManager`|Redis集成|分布式缓存，支持集群|
|`CaffeineCacheManager`|Caffeine集成|高性能，推荐的生产环境选择|
源码中的管理器架构
Spring通过`CacheManager`接口抽象缓存管理：
```java
public interface CacheManager {
    Cache getCache(String name);          // 根据名称获取缓存
    Collection<String> getCacheNames();   // 获取所有缓存名称
}
```
具体配置示例
**1. Caffeine缓存配置（推荐）**
```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(caffeineCacheBuilder());
        cacheManager.setCacheNames(Arrays.asList("users", "products", "orders"));
        return cacheManager;
    }
    
    private Caffeine<Object, Object> caffeineCacheBuilder() {
        return Caffeine.newBuilder()
                .initialCapacity(100)              // 初始容量
                .maximumSize(1000)                 // 最大容量
                .expireAfterAccess(10, TimeUnit.MINUTES)  // 访问后过期时间
                .expireAfterWrite(1, TimeUnit.HOURS)     // 写入后过期时间
                .recordStats();                    // 开启统计
    }
}
```
**2. Redis缓存配置**
```yaml
# application.yml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 3600000        # 缓存过期时间（毫秒）
      key-prefix: "app:"           # 键前缀
      use-key-prefix: true         # 是否使用前缀
      cache-null-values: true      # 是否缓存null值
  redis:
    host: localhost
    port: 6379
    password: yourpassword
```
```java
@Configuration
@EnableCaching
public class RedisCacheConfig {
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1))     // 默认过期时间
                .disableCachingNullValues()        // 不缓存null值
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .withCacheConfiguration("users", 
                    config.entryTtl(Duration.ofMinutes(30))) // 特定缓存配置
                .transactionAware()                // 支持事务
                .build();
    }
}
```
**3. 多级缓存配置**
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
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1));
        
        return RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .build();
    }
    
    @Bean("compositeCacheManager")
    public CacheManager compositeCacheManager(
            @Qualifier("localCacheManager") CacheManager local,
            @Qualifier("redisCacheManager") CacheManager redis) {
        
        CompositeCacheManager composite = new CompositeCacheManager();
        composite.setCacheManagers(Arrays.asList(local, redis));
        composite.setFallbackToNoOpCache(false); // 找不到缓存时抛出异常
        return composite;
    }
}
```
自定义缓存管理器
```java
@Component
public class CustomCacheManager extends AbstractCacheManager {
    
    private final Map<String, Cache> caches = new ConcurrentHashMap<>();
    
    @Override
    protected Collection<? extends Cache> loadCaches() {
        // 加载初始缓存配置
        return Arrays.asList(
            createCache("users"),
            createCache("products")
        );
    }
    
    @Override
    protected Cache getMissingCache(String name) {
        // 动态创建缓存
        return createCache(name);
    }
    
    private Cache createCache(String name) {
        return new ConcurrentMapCache(name, 
            CacheBuilder.newBuilder()
                .expireAfterWrite(10, TimeUnit.MINUTES)
                .maximumSize(1000)
                .build().asMap(), false);
    }
}
```
合理的缓存管理器配置是保证缓存性能和稳定性的基础。
###### 7. Spring 支持哪些缓存实现？
Spring通过统一的缓存抽象层支持多种缓存实现，开发者可以根据需求灵活选择。
主要缓存实现对比

|**缓存实现**​|**特点**​|**适用场景**​|**性能**​|
|---|---|---|---|
|**Caffeine**​|高性能，丰富的过期策略|生产环境首选|⭐⭐⭐⭐⭐|
|**Redis**​|分布式，持久化|集群环境，数据共享|⭐⭐⭐⭐|
|**Ehcache**​|功能全面，支持磁盘存储|单机复杂场景|⭐⭐⭐|
|**ConcurrentMap**​|简单内存缓存|测试，开发环境|⭐⭐|
具体集成配置
**1. Caffeine（推荐生产环境）**
```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>3.1.8</version>
</dependency>
```
```java
@Configuration
@EnableCaching
public class CaffeineConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
                .initialCapacity(100)
                .maximumSize(10000)
                .expireAfterWrite(10, TimeUnit.MINUTES)
                .recordStats());
        return manager;
    }
}
```
**2. Redis（分布式环境）**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
```java
@Configuration
@EnableCaching
public class RedisConfig {
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.create(factory);
    }
}
```
**3. 多缓存实现组合**
```java
@Configuration
@EnableCaching
public class MultiCacheConfig {
    
    // 本地热点数据缓存
    @Bean("localCacheManager")
    public CacheManager localCacheManager() {
        return new CaffeineCacheManager();
    }
    
    // 分布式共享缓存
    @Bean("sharedCacheManager") 
    public CacheManager sharedCacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.create(factory);
    }
    
    // 组合缓存管理器
    @Bean
    @Primary
    public CacheManager compositeCacheManager(
            @Qualifier("localCacheManager") CacheManager local,
            @Qualifier("sharedCacheManager") CacheManager shared) {
        
        List<CacheManager> managers = Arrays.asList(local, shared);
        CompositeCacheManager composite = new CompositeCacheManager();
        composite.setCacheManagers(managers);
        composite.setFallbackToNoOpCache(true);
        return composite;
    }
}
```
缓存实现选择策略
```java
@Service
public class StrategicCacheService {
    
    // 热点数据使用本地缓存
    @Cacheable(value = "hotData", cacheManager = "localCacheManager")
    public HotData getHotData(String id) {
        return dataService.loadHotData(id);
    }
    
    // 共享数据使用分布式缓存
    @Cacheable(value = "sharedData", cacheManager = "sharedCacheManager")
    public SharedData getSharedData(String key) {
        return sharedService.loadSharedData(key);
    }
    
    // 重要数据使用多级缓存
    @Caching(cacheable = {
        @Cacheable(value = "importantData", cacheManager = "localCacheManager"),
        @Cacheable(value = "importantData", cacheManager = "sharedCacheManager")
    })
    public ImportantData getImportantData(String id) {
        return importantService.loadImportantData(id);
    }
}
```
选择合适的缓存实现需要综合考虑数据特性、性能要求和系统架构。
###### 8. 缓存的过期策略如何设置？
合理的过期策略是保证缓存有效性和性能的关键，不同的缓存实现支持不同的过期策略。
过期策略类型

|**策略类型**​|**说明**​|**适用场景**​|
|---|---|---|
|**TTL（Time To Live）**​|固定时间后过期|常规数据缓存|
|**TTI（Time To Idle）**​|空闲时间后过期|访问频率不高的数据|
|**基于大小**​|达到最大容量时淘汰|内存敏感场景|
|**显式失效**​|手动清除过期数据|数据更新频繁场景|
具体配置示例
**1. Caffeine过期策略**
```java
@Configuration
public class CaffeineExpiryConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
                // 基于写入时间的过期
                .expireAfterWrite(30, TimeUnit.MINUTES)
                // 基于访问时间的过期  
                .expireAfterAccess(1, TimeUnit.HOURS)
                // 基于大小的淘汰
                .maximumSize(10000)
                // 基于权重的淘汰
                .maximumWeight(100000)
                .weigher((String key, Object value) -> {
                    // 自定义权重计算逻辑
                    return value.toString().length();
                }));
        return manager;
    }
}
```
**2. Redis过期策略**
```yaml
# application.yml
spring:
  cache:
    redis:
      time-to-live: 3600000        # 全局默认TTL（1小时）
      key-prefix: "cache:"
      use-key-prefix: true
      cache-null-values: true
  redis:
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0
        max-wait: -1ms
```
```java
@Configuration
@EnableCaching
public class RedisExpiryConfig {
    
    @Bean
    public RedisCacheManagerBuilderCustomizer redisCacheManagerBuilderCustomizer() {
        return builder -> builder
                .withCacheConfiguration("shortTermCache",
                    RedisCacheConfiguration.defaultCacheConfig()
                            .entryTtl(Duration.ofMinutes(10))) // 短期缓存
                .withCacheConfiguration("longTermCache",
                    RedisCacheConfiguration.defaultCacheConfig()  
                            .entryTtl(Duration.ofDays(1)))      // 长期缓存
                .withCacheConfiguration("eternalCache",
                    RedisCacheConfiguration.defaultCacheConfig()
                            .entryTtl(Duration.ZERO))           // 永不过期
                .enableStatistics();                            // 开启统计
    }
}
```
**3. 多级过期策略**
```java
@Service
public class MultiLevelExpiryService {
    
    // 短期热点数据：5分钟过期
    @Cacheable(value = "hotspot", key = "#id")
    @CacheConfig(cacheNames = "hotspot")
    public Data getHotspotData(String id) {
        return dataService.loadHotspot(id);
    }
    
    // 常规业务数据：1小时过期  
    @Cacheable(value = "business", key = "#id")
    @CacheConfig(cacheNames = "business")
    public Data getBusinessData(String id) {
        return dataService.loadBusiness(id);
    }
    
    // 基础数据：1天过期，可设置较长时间
    @Cacheable(value = "basic", key = "#id")  
    @CacheConfig(cacheNames = "basic")
    public Data getBasicData(String id) {
        return dataService.loadBasic(id);
    }
}
```
**4. 动态过期策略**
```java
@Service
public class DynamicExpiryService {
    
    @Autowired
    private CacheManager cacheManager;
    
    // 根据业务特性动态设置过期时间
    @Cacheable(value = "dynamic", key = "#data.id")
    public Data getDataWithDynamicExpiry(Data data) {
        // 根据数据特性决定缓存时间
        Duration ttl = calculateTTL(data);
        setCustomTTL("dynamic", data.getId(), ttl);
        return dataService.loadData(data.getId());
    }
    
    private Duration calculateTTL(Data data) {
        if (data.isHighFrequency()) {
            return Duration.ofMinutes(5);    // 高频数据短期缓存
        } else if (data.isLargeSize()) {
            return Duration.ofHours(1);      // 大文件适中时间
        } else {
            return Duration.ofDays(1);       // 常规数据长期缓存
        }
    }
    
    private void setCustomTTL(String cacheName, Object key, Duration ttl) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache instanceof RedisCache) {
            // Redis特定设置逻辑
            RedisCache redisCache = (RedisCache) cache;
            // 自定义Redis过期设置
        }
    }
}
```
**5. 过期策略监控**
```java
@Component
public class CacheExpiryMonitor {
    
    @Autowired
    private CacheManager cacheManager;
    
    @Scheduled(fixedRate = 300000) // 每5分钟执行一次
    public void monitorCacheExpiry() {
        cacheManager.getCacheNames().forEach(cacheName -> {
            Cache cache = cacheManager.getCache(cacheName);
            if (cache instanceof RedisCache) {
                monitorRedisCache((RedisCache) cache);
            } else if (cache instanceof CaffeineCache) {
                monitorCaffeineCache((CaffeineCache) cache);
            }
        });
    }
    
    private void monitorRedisCache(RedisCache cache) {
        // 监控Redis缓存命中率、过期情况等
        log.info("Redis缓存 {} 统计信息: {}", cache.getName(), cache.getNativeCache());
    }
    
    private void monitorCaffeineCache(CaffeineCache cache) {
        // 监控Caffeine缓存统计信息
        com.github.benmanes.caffeine.cache.stats.CacheStats stats = 
            ((com.github.benmanes.caffeine.cache.Cache) cache.getNativeCache()).stats();
        log.info("Caffeine缓存 {} 命中率: {}", cache.getName(), stats.hitRate());
    }
}
```
合理的过期策略需要根据业务数据特性、访问模式和系统资源来综合制定。