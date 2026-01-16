###### 1. 如何自定义 BeanPostProcessor？
BeanPostProcessor是Spring框架的核心扩展点，允许在Bean初始化前后插入自定义逻辑。其执行时机在Bean实例化、依赖注入完成后，初始化方法调用前后。
**实现原理与源码机制：**
Spring在`AbstractAutowireCapableBeanFactory.initializeBean()`方法中调用BeanPostProcessor：
```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
    // 1. 执行Aware接口方法
    invokeAwareMethods(beanName, bean);
    
    // 2. BeanPostProcessor前置处理
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }
    
    // 3. 执行初始化方法(@PostConstruct/InitializingBean)
    try {
        invokeInitMethods(beanName, wrappedBean, mbd);
    } catch (Throwable ex) { ... }
    
    // 4. BeanPostProcessor后置处理
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```
**实战示例：方法级性能监控处理器**
```java
@Component
public class ProfilingBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // 仅处理@Service注解的Bean
        if (bean.getClass().isAnnotationPresent(Service.class)) {
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                bean.getClass().getInterfaces(),
                (proxy, method, args) -> {
                    long start = System.currentTimeMillis();
                    Object result = method.invoke(bean, args);
                    long duration = System.currentTimeMillis() - start;
                    
                    if (duration > 100) { // 记录慢方法
                        Logger.warn("Slow method: {}.{} took {}ms", 
                            beanName, method.getName(), duration);
                    }
                    return result;
                });
        }
        return bean;
    }
}
```
**高级应用：自定义注解处理器**
```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Encrypted {
    String algorithm() default "AES";
}

@Component
public class EncryptionBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        Field[] fields = bean.getClass().getDeclaredFields();
        for (Field field : fields) {
            if (field.isAnnotationPresent(Encrypted.class)) {
                field.setAccessible(true);
                try {
                    String original = (String) field.get(bean);
                    Encrypted annotation = field.getAnnotation(Encrypted.class);
                    String encrypted = encrypt(original, annotation.algorithm());
                    field.set(bean, encrypted);
                } catch (Exception e) {
                    throw new RuntimeException("Encryption failed", e);
                }
            }
        }
        return bean;
    }
}
```
###### 2. 如何自定义 BeanFactoryPostProcessor？
BeanFactoryPostProcessor在Bean定义加载后、实例化前执行，用于修改Bean的配置元数据。
**源码执行时机：**
在`AbstractApplicationContext.refresh()`的`invokeBeanFactoryPostProcessors()`方法中调用：
```java
public void refresh() throws BeansException {
    synchronized (this.startupShutdownMonitor) {
        // 1. 准备上下文
        prepareRefresh();
        
        // 2. 获取BeanFactory并加载Bean定义
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        
        // 3. 执行BeanFactoryPostProcessor
        invokeBeanFactoryPostProcessors(beanFactory);
        
        // 4. 后续的Bean实例化等流程
        // ...
    }
}
```
**实战示例：动态属性修改**
```java
@Component
public class PropertyOverrideProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) 
            throws BeansException {
        String env = System.getProperty("spring.profiles.active", "dev");
        
        if ("prod".equals(env)) {
            BeanDefinition definition = beanFactory.getBeanDefinition("dataSource");
            MutablePropertyValues properties = definition.getPropertyValues();
            
            // 生产环境使用不同的配置
            properties.add("url", "jdbc:mysql://prod-server:3306/app");
            properties.add("maxPoolSize", 50);
        }
    }
}
```
**高级应用：条件化Bean注册**
```java
@Component
public class ConditionalBeanProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) 
            throws BeansException {
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
        
        // 根据条件动态注册Bean
        if (isCacheEnabled()) {
            GenericBeanDefinition cacheBean = new GenericBeanDefinition();
            cacheBean.setBeanClass(RedisCacheManager.class);
            registry.registerBeanDefinition("cacheManager", cacheBean);
        } else {
            // 移除不需要的Bean定义
            if (registry.containsBeanDefinition("cacheManager")) {
                registry.removeBeanDefinition("cacheManager");
            }
        }
    }
    
    private boolean isCacheEnabled() {
        return Boolean.parseBoolean(System.getProperty("app.cache.enabled", "true"));
    }
}
```
###### 3. 如何实现自定义注解？
自定义注解需要结合BeanPostProcessor或AOP来实现功能。以下是完整的开发流程：
**1. 定义注解**
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int value() default 100; // 每秒最大请求数
    String key() default ""; // 限流键
    long timeout() default 1000; // 超时时间(毫秒)
}

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {
    String lockName();
    long expireTime() default 30000; // 锁过期时间
}
```
**2. 实现注解处理器**
```java
@Component
public class RateLimitBeanPostProcessor implements BeanPostProcessor {
    
    private final RateLimiter rateLimiter = RateLimiter.create(1000);
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        Class<?> clazz = bean.getClass();
        Method[] methods = clazz.getDeclaredMethods();
        
        for (Method method : methods) {
            if (method.isAnnotationPresent(RateLimit.class)) {
                return createRateLimitProxy(bean, method);
            }
        }
        return bean;
    }
    
    private Object createRateLimitProxy(Object target, Method annotatedMethod) {
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            (proxy, method, args) -> {
                if (method.getName().equals(annotatedMethod.getName())) {
                    RateLimit limit = annotatedMethod.getAnnotation(RateLimit.class);
                    
                    // 获取令牌，如果无法获取则阻塞
                    if (!rateLimiter.tryAcquire(limit.timeout(), TimeUnit.MILLISECONDS)) {
                        throw new RateLimitExceededException("Rate limit exceeded");
                    }
                }
                return method.invoke(target, args);
            });
    }
}
```
**3. 使用注解**
```java
@Service
public class OrderService {
    
    @RateLimit(value = 50, key = "createOrder", timeout = 500)
    public Order createOrder(OrderRequest request) {
        // 业务逻辑
        return orderRepository.save(request.toOrder());
    }
}
```
###### 4. 如何扩展 Spring 容器？
Spring提供了多种扩展点来定制容器行为：
**1. ApplicationContextInitializer**
```java
public class CustomContextInitializer implements 
        ApplicationContextInitializer<ConfigurableApplicationContext> {
    
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        // 在容器刷新前执行自定义配置
        ConfigurableEnvironment env = applicationContext.getEnvironment();
        
        // 添加自定义PropertySource
        Map<String, Object> customProperties = loadCustomProperties();
        env.getPropertySources().addFirst(
            new MapPropertySource("customProperties", customProperties));
        
        // 动态注册Bean
        if (env.acceptsProfiles("cloud")) {
            registerCloudBeans(applicationContext);
        }
    }
}

// 注册方式：META-INF/spring.factories
// org.springframework.context.ApplicationContextInitializer=com.example.CustomContextInitializer
```
**2. BeanDefinitionRegistryPostProcessor（更强大的BeanFactoryPostProcessor）**
```java
@Component
public class DynamicBeanRegistry implements BeanDefinitionRegistryPostProcessor {
    
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) 
            throws BeansException {
        // 可以注册更多的Bean定义
        String[] beanNames = registry.getBeanDefinitionNames();
        
        // 动态扫描并注册组件
        ClassPathScanningCandidateComponentProvider scanner = 
            new ClassPathScanningCandidateComponentProvider(false);
        scanner.addIncludeFilter(new AnnotationTypeFilter(MyComponent.class));
        
        Set<BeanDefinition> candidates = scanner.findCandidateComponents("com.example");
        for (BeanDefinition candidate : candidates) {
            String beanName = generateBeanName(candidate);
            registry.registerBeanDefinition(beanName, candidate);
        }
    }
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) 
            throws BeansException {
        // 可以修改已有的Bean定义
    }
}
```
###### 5. 什么是 SPI 机制？Spring 如何使用 SPI？
**SPI（Service Provider Interface）是Java提供的服务发现机制，允许第三方实现可插拔的组件。Spring大量使用SPI机制来实现自动配置和扩展。
Spring Boot中的SPI应用：**
**1. 自动配置SPI**
```java
// META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration,\
com.example.OtherAutoConfiguration

// 自动配置类
@Configuration
@ConditionalOnClass(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class MyAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
    
    @Bean
    @ConditionalOnBean(DataSource.class)
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```
**2. 自定义SPI扩展**
```java
// 定义SPI接口
public interface CacheProvider {
    String getName();
    boolean isAvailable();
    CacheManager getCacheManager();
}

// META-INF/services/com.example.CacheProvider
// com.example.RedisCacheProvider
// com.example.CaffeineCacheProvider

// Spring集成
@Component
public class CacheProviderManager {
    
    @Autowired
    private ApplicationContext context;
    
    public CacheManager getCacheManager() {
        ServiceLoader<CacheProvider> loader = ServiceLoader.load(CacheProvider.class);
        
        for (CacheProvider provider : loader) {
            // 依赖Spring容器进行依赖注入
            context.getAutowireCapableBeanFactory().autowireBean(provider);
            
            if (provider.isAvailable()) {
                return provider.getCacheManager();
            }
        }
        return new SimpleCacheManager(); // 默认实现
    }
}
```
###### 6. 如何在 Spring 中集成第三方框架？
集成第三方框架通常采用**自动配置模式**，以下是完整的最佳实践：
**1. 创建starter模块结构**
```
myframework-spring-boot-starter/
├── src/main/java/
│   ├── com/example/autoconfigure/
│   │   ├── MyFrameworkProperties.java      # 配置属性
│   │   ├── MyFrameworkAutoConfiguration.java # 自动配置
│   │   └── MyFrameworkHealthIndicator.java   # 健康检查
│   └── com/example/MyFrameworkTemplate.java  # 核心模板类
├── src/main/resources/
│   └── META-INF/
│       ├── spring.factories                 # 自动配置注册
│       └── additional-spring-configuration-metadata.json # 配置元数据
└── pom.xml
```
**2. 实现自动配置类**
```java
@Configuration
@ConditionalOnClass(MyFrameworkClient.class)
@EnableConfigurationProperties(MyFrameworkProperties.class)
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class MyFrameworkAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public MyFrameworkTemplate myFrameworkTemplate(MyFrameworkProperties properties) {
        MyFrameworkClient client = new MyFrameworkClient();
        client.setEndpoint(properties.getEndpoint());
        client.setTimeout(properties.getTimeout());
        client.setRetryCount(properties.getRetryCount());
        return new MyFrameworkTemplate(client);
    }
    
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnEnabledHealthCheck("myframework")
    public MyFrameworkHealthIndicator myFrameworkHealthIndicator(
            MyFrameworkTemplate template) {
        return new MyFrameworkHealthIndicator(template);
    }
    
    @Bean
    public MyFrameworkEndpoint myFrameworkEndpoint(MyFrameworkTemplate template) {
        return new MyFrameworkEndpoint(template);
    }
}
```
**3. 配置属性类**
```java
@ConfigurationProperties(prefix = "myframework")
public class MyFrameworkProperties {
    private String endpoint = "http://localhost:8080";
    private int timeout = 5000;
    private int retryCount = 3;
    private Pool pool = new Pool();
    
    // getter和setter...
    
    public static class Pool {
        private int maxSize = 10;
        private int minSize = 2;
        // getter和setter...
    }
}
```
**4. 注册自动配置**
```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.autoconfigure.MyFrameworkAutoConfiguration

org.springframework.boot.autoconfigure.condition.ConditionalOnClass=\
com.example.MyFrameworkClient
```
**5. 使用方配置**
```yaml
# application.yml
myframework:
  endpoint: http://myframework-server:8080
  timeout: 10000
  retry-count: 5
  pool:
    max-size: 20
    min-size: 5
```
通过以上方式，第三方框架可以无缝集成到Spring生态中，用户只需添加starter依赖即可自动获得所有功能。