###### 1. 什么是 Spring EL（表达式语言）？

Spring 表达式语言（SpEL）是 Spring 提供的一种强大表达式语言，支持在运行时查询和操作对象图。与 OGNL 类似，但 SpEL 与 Spring 生态无缝集成，在注解、XML 配置、AOP 条件等场景都能使用。

SpEL 支持的操作非常丰富：属性访问（`user.name`）、方法调用（`'hello'.toUpperCase()`）、关系/算术/逻辑运算、正则匹配、集合投影和筛选、静态方法调用（`T(Math).random()`）、Bean 引用（`@beanName`）等。

```java
ExpressionParser parser = new SpelExpressionParser();
// 字符串操作
String msg = parser.parseExpression("'Hello' + ' World'").getValue(String.class);

// 通过 EvaluationContext 访问 Bean 属性
StandardEvaluationContext context = new StandardEvaluationContext(user);
String name = parser.parseExpression("name").getValue(context, String.class);

// 调用静态方法
double random = parser.parseExpression("T(java.lang.Math).random() * 100")
    .getValue(Double.class);
```

在注解中使用 SpEL 非常常见，比如 `@Cacheable(key = "#user.id")` 的键表达式、`@Value("#{systemProperties['user.home']}")` 的属性注入、`@PreAuthorize("hasRole('ADMIN')")` 的权限控制——这些背后都是 SpEL 在解析。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#SpEL表达式]]

---

###### 2. 什么是 Spring Validation？

Spring Validation 是 Spring 的数据校验模块，整合了 Bean Validation（JSR-303/JSR-349）标准，让数据校验变成声明式的注解配置，而不需要写一堆 `if (xxx == null)` 的判断代码。

架构上由几个核心组件构成：`Validator` 接口（Spring 自有的校验接口）、`DataBinder`（数据绑定和校验的核心类）、`Errors`/`BindingResult`（收集校验错误信息）以及 JSR-303 注解驱动校验。

在 Spring MVC 中，`@Valid` 或 `@Validated` 注解触发参数校验，`RequestResponseBodyMethodProcessor` 会检查参数注解并调用 `DataBinder.validate()` 执行校验。校验不通过时，`MethodArgumentNotValidException` 会被抛出，可以用 `@ExceptionHandler` 统一处理。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#参数校验]]

---

###### 3. 如何实现参数校验？

声明式注解校验是最佳实践，核心是在 DTO 字段上加约束注解，在 Controller 方法参数上加 `@Valid` 触发校验：

```java
@Data
public class UserDTO {
    @NotNull(message = "用户ID不能为空")
    @Min(value = 1, message = "用户ID必须大于0")
    private Long userId;
    
    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 10, message = "用户名长度必须在2-10之间")
    private String userName;
    
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    private String phone;
}
```

**分组校验**解决创建和更新场景下校验规则不同的问题，比如创建时 `userId` 不需要传，更新时必须有：

```java
public class UserDTO {
    @NotNull(groups = Update.class)    // 只在更新时校验
    private Long userId;
    
    @NotBlank(groups = {Create.class, Update.class})  // 创建和更新都要校验
    private String userName;
    
    public interface Create {}
    public interface Update {}
}

@PostMapping("/create")
public Result create(@Validated(UserDTO.Create.class) @RequestBody UserDTO user) { ... }

@PostMapping("/update")
public Result update(@Validated(UserDTO.Update.class) @RequestBody UserDTO user) { ... }
```

**嵌套校验**：对象内部有嵌套对象时，用 `@Valid` 触发级联校验：

```java
@Data
public class OrderDTO {
    @Valid
    @NotNull
    private UserDTO user;     // 会触发 UserDTO 内部的校验
    
    @Valid
    @NotEmpty
    private List<OrderItemDTO> items;
}
```

方法参数级别校验（如路径变量、查询参数），在 Controller 类上加 `@Validated`，直接在参数上加约束注解即可。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#参数校验]]

---

###### 4. 什么是国际化（i18n）？Spring 如何支持？

国际化（i18n）是让应用能够根据用户的语言和地区显示对应文本，而不需要修改代码逻辑。Spring 通过 `MessageSource` 接口提供了完善的国际化支持。

核心组件三件套：**`MessageSource`**（加载多语言消息资源）、**`LocaleResolver`**（解析当前请求的 Locale，支持从 Session、Cookie、HTTP Header 等来源获取）、**`LocaleContextHolder`**（ThreadLocal 存储，让业务代码随时获取当前 Locale）。

```java
@Configuration
public class I18nConfig {
    
    @Bean
    public MessageSource messageSource() {
        ResourceBundleMessageSource ms = new ResourceBundleMessageSource();
        ms.setBasenames("i18n/messages");  // 对应 messages_zh_CN.properties、messages_en_US.properties
        ms.setDefaultEncoding("UTF-8");
        return ms;
    }
    
    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver = new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.CHINA);
        return resolver;
    }
}
```

资源文件 `messages_zh_CN.properties`：
```properties
user.create.success=用户 {0} 创建成功
user.create.error=创建用户失败
```

业务代码里用 `messageSource.getMessage("user.create.success", new Object[]{userName}, locale)` 获取对应语言的文本。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#国际化支持]]

---

###### 5. 什么是资源加载？Spring 如何加载资源？

Spring 的资源加载机制提供了统一的资源访问抽象，通过 `Resource` 接口屏蔽了不同来源（类路径、文件系统、网络）的差异，让代码不需要关心资源在哪里。

`Resource` 接口有几个常用实现：`ClassPathResource`（类路径下的资源，前缀 `classpath:`）、`FileSystemResource`（文件系统，前缀 `file:`）、`UrlResource`（网络资源，`http:`/`ftp:` 等前缀）、`ServletContextResource`（Web 应用上下文资源）。

通过 `ResourceLoader` 按前缀自动选择实现，通过 `ResourcePatternResolver`（`PathMatchingResourcePatternResolver`）支持 Ant 风格的通配符：

```java
@Service
public class ResourceService {
    @Autowired
    private ResourceLoader resourceLoader;
    
    public void loadResources() throws IOException {
        // 加载类路径下的配置文件
        Resource config = resourceLoader.getResource("classpath:config/app.properties");
        
        // 通配符加载多个资源
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        Resource[] xmlFiles = resolver.getResources("classpath*:mappers/**/*.xml");
        
        if (config.exists()) {
            String content = StreamUtils.copyToString(
                config.getInputStream(), StandardCharsets.UTF_8);
        }
    }
}
```

`classpath:` 只搜索当前 classpath，`classpath*:` 搜索所有 classpath（包括 jar 包内的），这个区别在开发 starter 时很重要。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#资源加载机制]]

---

###### 6. Environment 接口的作用是什么？

`Environment` 接口是 Spring 的**环境抽象**，整合了来自多个来源的配置属性，为应用提供统一的配置访问入口，同时支持 Profile 管理。

属性来源的优先级从高到低：JVM 系统属性（`-Dkey=value`）→ 系统环境变量（OS 环境变量）→ 应用配置文件（`application.yml`）→ `@PropertySource` 加载的文件。高优先级的来源会覆盖低优先级，这也是为什么可以通过 JVM 参数覆盖配置文件里的值。

```java
@Service
public class AppService {
    @Autowired
    private Environment env;
    
    @PostConstruct
    public void init() {
        String url = env.getProperty("database.url");
        // 带默认值
        int port = env.getProperty("database.port", Integer.class, 3306);
        // 检查属性是否存在
        if (env.containsProperty("redis.enabled")) {
            boolean enabled = env.getProperty("redis.enabled", Boolean.class);
        }
        // Profile 检查
        if (env.acceptsProfiles(Profiles.of("dev"))) {
            // 开发环境特定逻辑
        }
    }
}
```

Profile 管理是 `Environment` 的重要功能：`@Profile("dev")` 注解的 Bean 只在 dev 环境下创建，通过 `spring.profiles.active=dev` 激活。多 Profile 支持 `spring.profiles.active=dev,local`，用逗号分隔。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#Environment体系]]

---

###### 7. PropertyResolver 接口的作用是什么？

`PropertyResolver` 是 `Environment` 的父接口，提供属性解析的基础能力：获取属性值（带/不带默认值）、类型安全的获取、属性占位符解析。

核心功能是 **`${...}` 占位符解析**，在 XML、`@Value`、配置文件中大量使用。`PropertyPlaceholderHelper` 负责解析占位符，找到对应的 `PropertySource` 取值并替换：

```java
@Configuration
@PropertySource("classpath:database.properties")
public class DatabaseConfig {
    
    @Value("${database.url}")
    private String url;
    
    @Value("${database.port:3306}")  // 带默认值的占位符
    private int port;
    
    @Bean
    public DataSource dataSource(Environment env) {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(env.getProperty("database.url"));
        return ds;
    }
}
```

`resolvePlaceholders()` 方法可以解析字符串里的 `${...}` 占位符，在处理配置值时很有用。`resolveRequiredPlaceholders()` 更严格，找不到对应属性时直接抛异常，而不是保留原始占位符字符串。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#Environment体系]]

---

###### 8. 什么是 Type Conversion？

Spring 类型转换是框架的基础设施，把外部输入（HTTP 请求参数、配置文件属性、数据库返回值）转换为 Java 对象所需的类型。核心 API 是 `Converter<S, T>` 接口和 `ConversionService`。

自定义转换器很简单，实现 `Converter<S, T>` 并注册到 `ConversionService`：

```java
@Component
public class StringToUserConverter implements Converter<String, User> {
    @Override
    public User convert(String source) {
        String[] parts = source.split(",");
        return new User(Long.valueOf(parts[0]), parts[1]);
    }
}

@Configuration
public class ConversionConfig {
    @Bean
    public ConversionService conversionService(StringToUserConverter converter) {
        DefaultConversionService service = new DefaultConversionService();
        service.addConverter(converter);
        return service;
    }
}
```

Spring 内置了大量默认转换器，处理基本类型之间的转换、字符串到枚举、字符串到各种数字类型等，大多数场景不需要自定义。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#类型转换与Formatter]]

---

###### 9. 什么是 Formatter？

`Formatter` 是类型转换的特殊形式，专为**文本格式化**场景设计，适合 Web 层的数据格式化需求（表单提交、JSON 序列化中的格式转换）。

`Formatter<T>` 接口继承了 `Printer<T>`（对象格式化为文本）和 `Parser<T>`（文本解析为对象），处理的是特定格式的字符串与 Java 对象之间的双向转换：

```java
@Component
public class DateFormatter implements Formatter<Date> {
    private final DateTimeFormatter formatter = 
        DateTimeFormatter.ofPattern("yyyy-MM-dd");
    
    @Override
    public Date parse(String text, Locale locale) throws ParseException {
        return new SimpleDateFormat("yyyy-MM-dd").parse(text);
    }
    
    @Override
    public String print(Date object, Locale locale) {
        return new SimpleDateFormat("yyyy-MM-dd").format(object);
    }
}
```

在 Spring MVC 中注册 `Formatter`：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addFormatter(new DateFormatter());
        // 也可以针对特定类型注册
        registry.addFormatterForFieldType(LocalDate.class, new ISOLocalDateFormatter());
    }
}
```

`Converter` 和 `Formatter` 的区别：`Converter` 是通用的类型转换，不感知 `Locale`；`Formatter` 针对文本格式，感知 `Locale`，适合做国际化场景下的格式转换（比如不同地区的数字格式、日期格式）。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/02、Spring整合扩展与高级特性#类型转换与Formatter]]
