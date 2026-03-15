###### 1. 如何自定义 BeanPostProcessor？

`BeanPostProcessor` 是 Spring 最重要的扩展点之一，允许在 Bean 初始化的前后插入自定义逻辑。执行时机：Bean 实例化、依赖注入完成之后，`@PostConstruct` 等初始化方法调用的前后。

接口定义非常简洁：

```java
public interface BeanPostProcessor {
    // 在初始化方法执行前调用
    Object postProcessBeforeInitialization(Object bean, String beanName);
    // 在初始化方法执行后调用
    Object postProcessAfterInitialization(Object bean, String beanName);
}
```

**注意**：这两个方法的返回值就是最终放入容器的 Bean，你可以返回原始对象，也可以返回一个包装后的代理对象。Spring AOP 就是通过 `AbstractAutoProxyCreator`（一个 `BeanPostProcessor`）在 `postProcessAfterInitialization` 阶段把原始 Bean 替换成代理对象的。

实战示例——方法级性能监控：

```java
@Component
public class ProfilingBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean.getClass().isAnnotationPresent(Service.class)) {
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                bean.getClass().getInterfaces(),
                (proxy, method, args) -> {
                    long start = System.currentTimeMillis();
                    Object result = method.invoke(bean, args);
                    long duration = System.currentTimeMillis() - start;
                    if (duration > 100) {
                        log.warn("慢方法: {}.{} 耗时 {}ms", beanName, method.getName(), duration);
                    }
                    return result;
                });
        }
        return bean;
    }
}
```

自定义注解处理也是 `BeanPostProcessor` 的高频应用场景，比如扫描字段上的 `@Encrypted` 注解，在 `postProcessBeforeInitialization` 阶段对字段值加密。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#自定义BeanPostProcessor]]

---

###### 2. 如何自定义 BeanFactoryPostProcessor？

`BeanFactoryPostProcessor` 在 Bean 定义加载完成后、Bean 实例化之前执行，用于修改 Bean 的配置元数据（`BeanDefinition`）。这个执行时机比 `BeanPostProcessor` 更早。

执行时机在 `AbstractApplicationContext.refresh()` 的 `invokeBeanFactoryPostProcessors()` 阶段，早于任何业务 Bean 的创建。

典型应用：动态修改 Bean 的属性配置（比如根据环境切换数据源参数）、条件化注册 Bean：

```java
@Component
public class PropertyOverrideProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        String env = System.getProperty("spring.profiles.active", "dev");
        if ("prod".equals(env)) {
            BeanDefinition definition = beanFactory.getBeanDefinition("dataSource");
            MutablePropertyValues props = definition.getPropertyValues();
            props.add("url", "jdbc:mysql://prod-server:3306/app");
            props.add("maxPoolSize", 50);
        }
    }
}
```

更强大的版本是 `BeanDefinitionRegistryPostProcessor`（继承自 `BeanFactoryPostProcessor`），额外提供 `postProcessBeanDefinitionRegistry()` 方法，可以动态注册或删除 Bean 定义——`ConfigurationClassPostProcessor` 就是这个接口的核心实现，负责处理 `@Configuration`/`@ComponentScan`/`@Import` 等注解。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#自定义BeanFactoryPostProcessor]]

---

###### 3. 如何实现自定义注解？

自定义注解通常结合 **AOP** 或 **BeanPostProcessor** 来实现功能。以下是一个完整的限流注解实现流程。

**第一步：定义注解**

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int value() default 100;        // 每秒最大请求数
    String key() default "";        // 限流键
    long timeout() default 1000;    // 超时时间（毫秒）
}
```

**第二步：用 AOP 实现注解逻辑（推荐）**

```java
@Aspect
@Component
public class RateLimitAspect {
    
    private final Map<String, RateLimiter> rateLimiters = new ConcurrentHashMap<>();
    
    @Around("@annotation(rateLimit)")
    public Object around(ProceedingJoinPoint pjp, RateLimit rateLimit) throws Throwable {
        String key = rateLimit.key().isEmpty() ? 
            pjp.getSignature().toShortString() : rateLimit.key();
        
        RateLimiter limiter = rateLimiters.computeIfAbsent(key, 
            k -> RateLimiter.create(rateLimit.value()));
        
        if (!limiter.tryAcquire(rateLimit.timeout(), TimeUnit.MILLISECONDS)) {
            throw new RateLimitExceededException("请求过于频繁，请稍后重试");
        }
        return pjp.proceed();
    }
}
```

**第三步：使用**

```java
@Service
public class OrderService {
    
    @RateLimit(value = 50, key = "createOrder", timeout = 500)
    public Order createOrder(OrderRequest request) {
        return orderRepository.save(request.toOrder());
    }
}
```

AOP 方式比 `BeanPostProcessor` 代理方式更推荐，代码更清晰，切点表达式更灵活，也更容易和事务等其他切面协作。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#自定义注解与AOP]]

---

###### 4. 如何扩展 Spring 容器？

Spring 提供了多个容器扩展点，可以在不同阶段介入容器的初始化流程。

**`ApplicationContextInitializer`**：在容器 `refresh()` 之前执行，可以添加自定义 `PropertySource`、激活特定 Profile：

```java
public class CustomContextInitializer implements 
        ApplicationContextInitializer<ConfigurableApplicationContext> {
    
    @Override
    public void initialize(ConfigurableApplicationContext ctx) {
        ConfigurableEnvironment env = ctx.getEnvironment();
        Map<String, Object> props = loadCustomProperties();
        env.getPropertySources().addFirst(new MapPropertySource("custom", props));
    }
}
// 注册：META-INF/spring.factories
// org.springframework.context.ApplicationContextInitializer=com.example.CustomContextInitializer
```

**`BeanDefinitionRegistryPostProcessor`**：在 Bean 定义加载阶段介入，可以动态扫描并注册额外的 Bean 定义：

```java
@Component
public class DynamicBeanRegistry implements BeanDefinitionRegistryPostProcessor {
    
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        ClassPathScanningCandidateComponentProvider scanner =
            new ClassPathScanningCandidateComponentProvider(false);
        scanner.addIncludeFilter(new AnnotationTypeFilter(MyComponent.class));
        
        for (BeanDefinition candidate : scanner.findCandidateComponents("com.example")) {
            registry.registerBeanDefinition(generateBeanName(candidate), candidate);
        }
    }
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {}
}
```

扩展点选择原则：需要修改环境配置用 `ApplicationContextInitializer`；需要修改 Bean 定义用 `BeanFactoryPostProcessor` 或 `BeanDefinitionRegistryPostProcessor`；需要修改 Bean 实例用 `BeanPostProcessor`。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#Spring容器扩展点]]

---

###### 5. 什么是 SPI 机制？Spring 如何使用 SPI？

SPI（Service Provider Interface）是 Java 提供的服务发现机制，允许第三方实现可插拔的组件扩展。核心思想是：定义接口和协议，实现由第三方提供，运行时通过 `ServiceLoader` 动态加载。

Spring Boot 大量使用 SPI 机制，核心是 `META-INF/spring.factories` 文件。这个文件里声明自动配置类、初始化器、监听器等，Spring Boot 启动时会读取所有 jar 包里的这个文件，自动注册：

```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration,\
com.example.OtherAutoConfiguration
```

自动配置类结合条件注解，实现按需装配：

```java
@Configuration
@ConditionalOnClass(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class MyAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

Spring Boot 3.x 之后，`spring.factories` 被 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 替代，格式更简洁，一行一个配置类。

SPI 机制的价值是实现了**框架的可插拔性**——引入 starter 依赖，自动获得配置，不需要修改任何代码，这正是 Spring Boot "约定优于配置"理念的核心实现手段。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#SPI机制]]

---

###### 6. 如何在 Spring 中集成第三方框架？

集成第三方框架的标准做法是**开发 Spring Boot Starter**，让用户只需引入一个依赖就能自动获得所有功能。

结构上分为三部分：**`AutoConfiguration` 类**（条件化创建 Bean）、**`Properties` 类**（绑定 `application.yml` 中的配置项）、**`spring.factories` 文件**（注册自动配置类）。

```java
// 配置属性类
@ConfigurationProperties(prefix = "myframework")
public class MyFrameworkProperties {
    private String endpoint = "http://localhost:8080";
    private int timeout = 5000;
    private int retryCount = 3;
    // getter 和 setter...
}

// 自动配置类
@Configuration
@ConditionalOnClass(MyFrameworkClient.class)  // 类路径存在才生效
@EnableConfigurationProperties(MyFrameworkProperties.class)
public class MyFrameworkAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean  // 用户没有自定义时才创建
    public MyFrameworkTemplate myFrameworkTemplate(MyFrameworkProperties properties) {
        MyFrameworkClient client = new MyFrameworkClient();
        client.setEndpoint(properties.getEndpoint());
        client.setTimeout(properties.getTimeout());
        return new MyFrameworkTemplate(client);
    }
}
```

```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.autoconfigure.MyFrameworkAutoConfiguration
```

集成之后，用户只需在 `application.yml` 里配置 `myframework.endpoint = ...` 就能使用，整个集成过程对用户透明。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#第三方框架集成]]
