###### 1. 什么是 Spring EL（表达式语言）？
Spring表达式语言（SpEL）是一种强大的表达式语言，支持在运行时查询和操作对象图。与OGNL和MVEL类似，但SpEL与Spring生态系统无缝集成，提供了更丰富的功能特性。
**核心特性：**
- **Bean属性访问**：支持通过表达式直接访问Bean属性和方法
- **方法调用**：可以在表达式中调用对象方法
- **运算符支持**：关系、算术、逻辑、正则匹配等丰富运算符
- **集合操作**：支持集合的投影、筛选等复杂操作
- **模板表达式**：支持可复用的表达式模板
**源码中的表达式解析机制：**
Spring通过`ExpressionParser`接口解析表达式，`EvaluationContext`提供表达式执行的上下文环境。
```java
// SpEL核心API使用示例
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("'Hello World'.concat('!')"); 
String message = (String) exp.getValue(); // 返回"Hello World!"

// 方法调用示例
parser.parseExpression("'spring'.toUpperCase()").getValue(String.class); // 返回"SPRING"
```
**在Bean定义中的高级应用：**
```xml
<!-- XML配置中使用SpEL -->
<bean id="world" class="java.lang.String">
    <constructor-arg value="#{'World!'}"/>
</bean>

<bean id="hello1" class="java.lang.String">
    <constructor-arg value="#{'Hello ' + @world}"/>
</bean>
```
```java
// 注解方式使用SpEL
@Component
public class SpELBean {
    @Value("#{systemProperties['user.home']}")
    private String userHome;
    
    @Value("#{T(java.lang.Math).random() * 100.0}")
    private double randomNumber;
}
```
###### 2. 什么是 Spring Validation？
Spring Validation是Spring框架的数据校验模块，它整合了Bean Validation（JSR-303/JSR-349）标准，并提供了Spring特有的增强功能。
**架构组成：**
- **Validator接口**：Spring自有的校验接口，定义`supports()`和`validate()`方法
- **DataBinder**：数据绑定和校验的核心类
- **Errors接口**：收集和存储校验错误信息
- **注解驱动校验**：基于JSR-303标准的注解校验支持
**源码层面的校验流程：**
在Spring MVC中，`RequestResponseBodyMethodProcessor`负责处理`@RequestBody`参数校验：
```java
public class RequestResponseBodyMethodProcessor {
    protected void validateIfApplicable(WebDataBinder binder, MethodParameter parameter) {
        // 判断是否需要执行校验
        for (Annotation ann : parameter.getParameterAnnotations()) {
            Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);
            if (validatedAnn != null || ann.annotationType().getSimpleName().startsWith("Valid")) {
                Object hints = (validatedAnn != null ? validatedAnn.value() : AnnotationUtils.getValue(ann));
                Object[] validationHints = (hints instanceof Object[] ? (Object[]) hints : new Object[] {hints});
                binder.validate(validationHints); // 执行校验
                break;
            }
        }
    }
}
```
###### 3. 如何实现参数校验？
**声明式校验（注解驱动）是最佳实践**，Spring提供了完整的注解校验支持。
**基础注解校验：**
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
**分组校验实现复杂场景：**
```java
public class UserDTO {
    @NotNull(groups = Update.class)
    private Long userId;
    
    @NotBlank(groups = {Create.class, Update.class})
    private String userName;
    
    // 分组接口定义
    public interface Create {}
    public interface Update {}
}

// 在Controller中使用分组校验
@PostMapping("/create")
public Result createUser(@Validated(UserDTO.Create.class) @RequestBody UserDTO user) {
    return Result.ok();
}

@PostMapping("/update")  
public Result updateUser(@Validated(UserDTO.Update.class) @RequestBody UserDTO user) {
    return Result.ok();
}
```
**方法参数级校验：**
```java
@RestController
@Validated // 类级别开启方法参数校验
public class UserController {
    
    @GetMapping("/{userId}")
    public UserDTO getUser(
            @PathVariable @Min(1) Long userId,
            @RequestParam @NotBlank String type) {
        return userService.getUser(userId, type);
    }
}
```
**嵌套校验支持：**
```java
@Data
public class OrderDTO {
    @Valid
    @NotNull
    private UserDTO user;
    
    @Valid
    @NotEmpty(message = "订单项不能为空")
    private List<OrderItemDTO> items;
}

@Data
public class OrderItemDTO {
    @NotBlank
    private String productName;
    
    @Min(1)
    private Integer quantity;
}
```
###### 4. 什么是国际化（i18n）？Spring 如何支持国际化？
国际化（i18n）是指使应用程序能够适应不同语言和地区的过程，而无需修改代码逻辑。Spring通过`MessageSource`接口提供了完善的国际化支持。
**Spring国际化核心组件：**
- **MessageSource**：国际化消息加载的核心接口
- **LocaleResolver**：区域信息解析器，决定当前使用的Locale
- **LocaleContextHolder**：通过ThreadLocal存储当前Locale信息
**源码层面的Locale解析机制：**
```java
public class SessionLocaleResolver implements LocaleResolver {
    public static final String LOCALE_SESSION_ATTRIBUTE_NAME = 
        SessionLocaleResolver.class.getName() + ".LOCALE";
    
    public Locale resolveLocale(HttpServletRequest request) {
        // 1. 检查session中是否设置了locale
        HttpSession session = request.getSession(false);
        if (session != null) {
            Locale locale = (Locale) session.getAttribute(LOCALE_SESSION_ATTRIBUTE_NAME);
            if (locale != null) return locale;
        }
        
        // 2. 检查默认locale设置
        if (this.defaultLocale != null) return this.defaultLocale;
        
        // 3. 返回请求的Accept-Language
        return request.getLocale();
    }
}
```
**国际化实战配置：**
```java
@Configuration
public class I18nConfig {
    
    @Bean
    public MessageSource messageSource() {
        ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
        messageSource.setBasenames("i18n/messages", "i18n/errors"); // 资源文件基名
        messageSource.setDefaultEncoding("UTF-8");
        messageSource.setUseCodeAsDefaultMessage(true); // 找不到资源时返回编码
        return messageSource;
    }
    
    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver = new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.CHINA); // 设置默认locale
        return resolver;
    }
}
```
**在业务代码中使用国际化：**
```java
@Service
public class UserService {
    
    @Autowired
    private MessageSource messageSource;
    
    public void createUser(UserDTO user) {
        // 获取当前Locale
        Locale locale = LocaleContextHolder.getLocale();
        
        // 使用国际化消息
        String successMsg = messageSource.getMessage(
            "user.create.success", 
            new Object[]{user.getUserName()}, 
            "默认成功消息", 
            locale
        );
        
        String errorMsg = messageSource.getMessage(
            "user.create.error", 
            null, 
            "创建用户失败", 
            locale
        );
    }
}
```
###### 5. 什么是资源加载？Spring 如何加载资源？
Spring的资源加载机制提供了统一的资源访问抽象，屏蔽了底层资源（类路径、文件系统、URL等）的差异。
**Resource接口体系：**
- **UrlResource**：访问网络或文件系统资源
- **ClassPathResource**：访问类路径下的资源
- **FileSystemResource**：访问文件系统资源
- **ServletContextResource**：Web应用上下文资源访问
**源码中的资源加载策略：**
Spring通过`ResourceLoader`和`ResourcePatternResolver`实现资源加载：
```java
public interface ResourceLoader {
    Resource getResource(String location);
    ClassLoader getClassLoader();
}

public interface ResourcePatternResolver extends ResourceLoader {
    Resource[] getResources(String locationPattern) throws IOException;
}
```
**资源加载实战应用：**
```java
@Service
public class ResourceService {
    
    @Autowired
    private ResourceLoader resourceLoader;
    
    public void loadResources() throws IOException {
        // 加载类路径资源
        Resource classpathResource = resourceLoader.getResource("classpath:config/app.properties");
        
        // 加载文件系统资源
        Resource fileResource = resourceLoader.getResource("file:/etc/app/config.properties");
        
        // 加载URL资源
        Resource urlResource = resourceLoader.getResource("https://example.com/config.properties");
        
        // 模式匹配加载多个资源
        ResourcePatternResolver patternResolver = new PathMatchingResourcePatternResolver();
        Resource[] resources = patternResolver.getResources("classpath*:config/*.properties");
        
        // 读取资源内容
        if (classpathResource.exists()) {
            InputStream is = classpathResource.getInputStream();
            String content = StreamUtils.copyToString(is, StandardCharsets.UTF_8);
        }
    }
}
```
###### 6. Environment 接口的作用是什么？
Environment接口是Spring框架的环境抽象，它整合了properties文件和系统环境变量、JVM系统属性等多方面的配置属性，为应用提供统一的配置访问接口。
**Environment接口的核心能力：**
- **属性解析**：提供层次化的属性访问（系统环境变量 > JVM系统属性 > 应用配置文件）
- **Profile管理**：支持基于Profile的环境配置隔离
- **类型安全的配置访问**：支持`@ConfigurationProperties`绑定
**源码层面的属性解析顺序：**
```java
public abstract class AbstractEnvironment implements ConfigurableEnvironment {
    private final MutablePropertySources propertySources = new MutablePropertySources();
    
    public AbstractEnvironment() {
        // 按优先级添加PropertySource
        propertySources.addLast(new PropertiesPropertySource("systemProperties", 
            getSystemProperties()));
        propertySources.addLast(new SystemEnvironmentPropertySource("systemEnvironment", 
            getSystemEnvironment()));
    }
    
    public String getProperty(String key) {
        // 按优先级解析属性
        for (PropertySource<?> propertySource : this.propertySources) {
            Object value = propertySource.getProperty(key);
            if (value != null) return String.valueOf(value);
        }
        return null;
    }
}
```
**Environment实战应用：**
```java
@Service
public class DatabaseService {
    
    @Autowired
    private Environment env;
    
    @PostConstruct
    public void init() {
        // 获取数据库配置
        String url = env.getProperty("database.url");
        String username = env.getProperty("database.username", "sa"); // 默认值
        int port = env.getProperty("database.port", Integer.class, 3306);
        
        // 检查Profile激活状态
        if (env.acceptsProfiles("dev")) {
            // 开发环境特定逻辑
        }
        
        // 检查属性是否存在
        if (env.containsProperty("redis.enabled")) {
            boolean redisEnabled = env.getProperty("redis.enabled", Boolean.class);
        }
    }
}
```
###### 7. PropertyResolver 接口的作用是什么？
PropertyResolver是Environment的父接口，提供了属性解析的基础能力，支持属性占位符解析和类型转换。
**PropertyResolver核心方法：**
- `getProperty(String key)`：获取属性值
- `getProperty(String key, String defaultValue)`：带默认值的属性获取
- `resolvePlaceholders(String text)`：解析占位符
**属性占位符解析机制：**
```java
public class PropertyPlaceholderHelper {
    public String replacePlaceholders(String value, PlaceholderResolver placeholderResolver) {
        // 解析${...}格式的占位符
        return parseStringValue(value, placeholderResolver, null);
    }
}
```
**在Bean定义中使用属性占位符：**
```xml
<bean id="dataSource" class="com.zaxxer.hikari.HikariDataSource">
    <property name="jdbcUrl" value="${database.url}"/>
    <property name="username" value="${database.username}"/>
    <property name="password" value="${database.password}"/>
</bean>
```
```java
@Configuration
@PropertySource("classpath:database.properties")
public class DatabaseConfig {
    
    @Value("${database.url}")
    private String url;
    
    @Bean
    public DataSource dataSource(Environment env) {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(env.getProperty("database.url"));
        return dataSource;
    }
}
```
###### 8. 什么是 Type Conversion？
Spring类型转换是框架中处理对象类型转换的基础设施，它将外部输入（如HTTP参数、配置文件属性）转换为内部Java对象所需的类型。
**类型转换核心API：**
- **Converter<S, T>**：通用类型转换接口
- **ConversionService**：类型转换服务入口
- **GenericConverter**：支持复杂类型转换
**自定义类型转换器：**
```java
@Component
public class StringToUserConverter implements Converter<String, User> {
    @Override
    public User convert(String source) {
        String[] parts = source.split(",");
        User user = new User();
        user.setId(Long.valueOf(parts[0]));
        user.setName(parts[1]);
        return user;
    }
}

@Configuration
public class ConversionConfig {
    
    @Bean
    public ConversionService conversionService() {
        DefaultConversionService service = new DefaultConversionService();
        service.addConverter(new StringToUserConverter());
        return service;
    }
}
```
###### 9. 什么是 Formatter？
Formatter是类型转换的特殊形式，专为文本格式化场景设计，特别适合Web层的数据格式化需求。
**Formatter接口设计：**
```java
public interface Formatter<T> extends Printer<T>, Parser<T> {
    // Printer: 对象格式化为文本
    // Parser: 文本解析为对象
}
```
**自定义日期格式化器：**
```java
@Component
public class DateFormatter implements Formatter<Date> {
    private final SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
    
    @Override
    public Date parse(String text, Locale locale) throws ParseException {
        return dateFormat.parse(text);
    }
    
    @Override
    public String print(Date object, Locale locale) {
        return dateFormat.format(object);
    }
}
```
**在Spring MVC中注册Formatter：**
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Autowired
    private DateFormatter dateFormatter;
    
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addFormatter(dateFormatter);
        registry.addFormatterForFieldType(LocalDate.class, new ISOLocalDateFormatter());
    }
}
```
Spring通过这些核心机制提供了强大的数据表达、校验、转换和国际化支持，这些功能共同构成了Spring应用开发的基础设施。