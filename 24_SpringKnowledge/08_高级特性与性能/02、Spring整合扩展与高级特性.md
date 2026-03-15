# Spring 整合、扩展与高级特性

这篇文档覆盖 Spring 的几个高级主题：如何扩展容器、SPI 机制、SpEL 表达式、参数校验、国际化、资源加载以及 Environment 体系。这些内容在日常开发和框架设计中都很常用。

---

## 一、扩展 Spring 容器

Spring 提供了丰富的扩展点，从 Bean 定义阶段到初始化阶段都可以插入自定义逻辑。

### 自定义 BeanPostProcessor

`BeanPostProcessor` 在每个 Bean 初始化前后执行，是最常用的扩展点。可以用来实现自定义注解处理、AOP 代理、性能监控等：

```java
@Component
public class ProfilingBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // 只处理 @Service 注解的 Bean
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

执行时序在 `AbstractAutowireCapableBeanFactory.initializeBean()` 里：依赖注入完成 → Aware 接口 → `postProcessBeforeInitialization` → `@PostConstruct` / `InitializingBean` → `postProcessAfterInitialization`。

---

### 自定义 BeanFactoryPostProcessor

`BeanFactoryPostProcessor` 在所有 Bean 定义加载完成、Bean 实例化之前执行，用于修改 Bean 的配置元数据（BeanDefinition）。

```java
@Component
public class PropertyOverrideProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        String env = System.getProperty("spring.profiles.active", "dev");
        
        if ("prod".equals(env)) {
            BeanDefinition definition = beanFactory.getBeanDefinition("dataSource");
            MutablePropertyValues properties = definition.getPropertyValues();
            properties.add("url", "jdbc:mysql://prod-server:3306/app");
            properties.add("maximumPoolSize", 50);
        }
    }
}
```

更强大的是 `BeanDefinitionRegistryPostProcessor`，它在 `BeanFactoryPostProcessor` 之前执行，可以动态注册/删除 Bean 定义。MyBatis 的 `MapperScannerConfigurer` 就是这个接口的实现。

---

### ApplicationContextInitializer

在容器 `refresh()` 之前执行，可以向环境中添加自定义 PropertySource 或做其他准备工作：

```java
public class CustomContextInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        ConfigurableEnvironment env = applicationContext.getEnvironment();
        
        // 添加自定义配置源
        Map<String, Object> customProps = loadCustomProperties();
        env.getPropertySources().addFirst(new MapPropertySource("customProperties", customProps));
    }
}
```

注册方式：在 `META-INF/spring.factories` 里配置，或在 Spring Boot 的 `SpringApplication.addInitializers()` 方法里注册。

---

### 自定义注解

完整的自定义注解开发流程：定义注解 → 实现处理器（BeanPostProcessor 或 AOP）→ 使用注解。

```java
// 1. 定义限流注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int value() default 100;  // 每秒最大请求数
    long timeout() default 1000;
}

// 2. 用 AOP 实现处理逻辑
@Aspect
@Component
public class RateLimitAspect {
    
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    @Around("@annotation(rateLimit)")
    public Object around(ProceedingJoinPoint pjp, RateLimit rateLimit) throws Throwable {
        String key = pjp.getSignature().toShortString();
        RateLimiter limiter = limiters.computeIfAbsent(key, 
            k -> RateLimiter.create(rateLimit.value()));
        
        if (!limiter.tryAcquire(rateLimit.timeout(), TimeUnit.MILLISECONDS)) {
            throw new RateLimitExceededException("请求频率超限");
        }
        return pjp.proceed();
    }
}

// 3. 使用
@Service
public class OrderService {
    @RateLimit(value = 50)
    public Order createOrder(OrderRequest request) { ... }
}
```

---

## 二、SPI 机制

SPI（Service Provider Interface）是 Java 提供的可插拔扩展机制，Spring Boot 大量使用它来实现自动配置。

**Spring Boot 的自动配置 SPI** 就是 `META-INF/spring.factories` 文件（Spring Boot 3.x 之后改为 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`）：

```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    com.example.MyAutoConfiguration,\
    com.example.OtherAutoConfiguration
```

自定义 Starter 的核心就是利用这个 SPI 机制，把自己的 `AutoConfiguration` 类注册进去，Spring Boot 启动时会自动发现并加载。

---

## 三、SpEL 表达式

Spring Expression Language（SpEL）是 Spring 提供的表达式语言，支持在运行时查询和操作对象图，几乎随处可见（@Value、@Cacheable 的 key、@ConditionalOnExpression 等）。

**基本用法：**

```java
ExpressionParser parser = new SpelExpressionParser();

// 字符串操作
String result = parser.parseExpression("'Hello World'.toUpperCase()").getValue(String.class);

// 访问 Bean 属性（通过 EvaluationContext）
User user = new User("张三", 25);
EvaluationContext context = new StandardEvaluationContext(user);
String name = parser.parseExpression("name").getValue(context, String.class); // 返回 "张三"

// 调用静态方法
double random = parser.parseExpression("T(java.lang.Math).random() * 100.0")
    .getValue(Double.class);

// 条件表达式
Boolean result = parser.parseExpression("age > 18 ? '成年' : '未成年'")
    .getValue(context, Boolean.class);
```

**在注解里的常见用法：**

```java
// @Value 里注入系统属性
@Value("#{systemProperties['user.home']}")
private String userHome;

// 注入其他 Bean 的属性
@Value("#{appConfig.maxRetry}")
private int maxRetry;

// @Cacheable 里自定义 key
@Cacheable(value = "users", key = "#user.id + ':' + #user.type")
public User getUser(UserQuery user) { ... }

// @ConditionalOnExpression
@ConditionalOnExpression("${feature.enabled:false} and ${module.active:true}")
public FeatureService featureService() { ... }
```

---

## 四、参数校验

Spring 集成了 Bean Validation（JSR-303/380）标准，声明式注解校验是首选方式。

**基本校验注解：**

```java
@Data
public class UserDTO {
    @NotNull(message = "用户 ID 不能为空")
    @Min(value = 1, message = "用户 ID 必须大于 0")
    private Long userId;
    
    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 20, message = "用户名长度在 2-20 之间")
    private String userName;
    
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    private String phone;
}
```

**分组校验**（新增和修改用不同规则）：

```java
public class UserDTO {
    @NotNull(groups = Update.class)  // 修改时必填
    private Long userId;
    
    @NotBlank(groups = {Create.class, Update.class})  // 新增和修改都必填
    private String userName;
    
    public interface Create {}
    public interface Update {}
}

@PostMapping("/create")
public Result createUser(@Validated(UserDTO.Create.class) @RequestBody UserDTO user) { ... }

@PostMapping("/update")
public Result updateUser(@Validated(UserDTO.Update.class) @RequestBody UserDTO user) { ... }
```

**嵌套校验**（级联校验内部对象）：

```java
@Data
public class OrderDTO {
    @Valid
    @NotNull
    private UserDTO user;  // 会触发 UserDTO 内部的校验
    
    @Valid
    @NotEmpty(message = "订单项不能为空")
    private List<OrderItemDTO> items;
}
```

---

## 五、国际化（i18n）

Spring 通过 `MessageSource` 接口提供国际化支持，核心是根据当前 `Locale` 加载对应的消息文件。

```java
@Configuration
public class I18nConfig {
    
    @Bean
    public MessageSource messageSource() {
        ResourceBundleMessageSource source = new ResourceBundleMessageSource();
        source.setBasenames("i18n/messages", "i18n/errors");
        source.setDefaultEncoding("UTF-8");
        return source;
    }
    
    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver = new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.SIMPLIFIED_CHINESE);
        return resolver;
    }
}
```

消息文件：`messages_zh_CN.properties`（中文）、`messages_en.properties`（英文）。

在业务代码里使用：

```java
@Autowired
private MessageSource messageSource;

public String getWelcomeMessage() {
    Locale locale = LocaleContextHolder.getLocale();
    return messageSource.getMessage("welcome.message", new Object[]{"张三"}, locale);
}
```

---

## 六、Environment 与配置属性体系

`Environment` 是 Spring 的环境抽象，统一管理所有配置属性，并支持 Profile 管理。

**属性优先级（从高到低）**：命令行参数 > 系统环境变量 > JVM 系统属性 > 应用配置文件 > 默认值

```java
@Service
public class ConfigService {
    
    @Autowired
    private Environment env;
    
    public void showConfig() {
        // 获取属性，支持默认值和类型转换
        String url = env.getProperty("database.url");
        int port = env.getProperty("server.port", Integer.class, 8080);
        
        // 检查 Profile
        if (env.acceptsProfiles("dev")) {
            // 开发环境特有逻辑
        }
        
        // 检查属性是否存在
        boolean redisEnabled = env.containsProperty("redis.enabled")
            && env.getProperty("redis.enabled", Boolean.class, false);
    }
}
```

`PropertyResolver` 是 `Environment` 的父接口，专门负责属性解析和占位符替换（`${...}` 格式）。

---

## 七、资源加载

Spring 通过 `Resource` 接口统一抽象了各类资源的访问方式，屏蔽了底层差异：

```java
@Service
public class ResourceService {
    
    @Autowired
    private ResourceLoader resourceLoader;
    
    public void loadResources() throws IOException {
        // 类路径资源
        Resource classpath = resourceLoader.getResource("classpath:config/app.properties");
        
        // 文件系统资源
        Resource file = resourceLoader.getResource("file:/etc/app/config.properties");
        
        // 网络资源
        Resource url = resourceLoader.getResource("https://config-server/app.properties");
        
        // 通配符批量加载
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        Resource[] resources = resolver.getResources("classpath*:config/*.xml");
        
        if (classpath.exists()) {
            String content = StreamUtils.copyToString(
                classpath.getInputStream(), StandardCharsets.UTF_8);
        }
    }
}
```

---

## 八、类型转换体系

Spring 提供了完整的类型转换基础设施，用于处理 HTTP 参数、配置属性等外部输入与 Java 类型之间的转换。

**自定义 Converter（通用转换）**：

```java
@Component
public class StringToUserConverter implements Converter<String, User> {
    @Override
    public User convert(String source) {
        String[] parts = source.split(",");
        return new User(Long.valueOf(parts[0]), parts[1]);
    }
}
```

**自定义 Formatter（Web 层格式化）**：

```java
@Component
public class DateFormatter implements Formatter<LocalDate> {
    
    @Override
    public LocalDate parse(String text, Locale locale) {
        return LocalDate.parse(text, DateTimeFormatter.ISO_LOCAL_DATE);
    }
    
    @Override
    public String print(LocalDate date, Locale locale) {
        return date.format(DateTimeFormatter.ISO_LOCAL_DATE);
    }
}
```

注册到 Spring MVC：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addFormatter(new DateFormatter());
    }
}
```

---

## 相关面试题 →

- [[../../10_Developlanguage/005_Spring/01_SpringSubject/12、Spring 整合与扩展]]
- [[../../10_Developlanguage/005_Spring/01_SpringSubject/14、Spring 其他高级特性]]
