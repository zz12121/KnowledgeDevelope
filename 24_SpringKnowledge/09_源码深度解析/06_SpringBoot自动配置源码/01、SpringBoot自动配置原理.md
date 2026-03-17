# Spring Boot 自动配置源码解析

> **核心类**：`SpringApplication`、`AutoConfigurationImportSelector`  
> **核心包**：`org.springframework.boot.autoconfigure`

---

## 一、@SpringBootApplication 注解解析

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration          // ← 相当于 @Configuration
@EnableAutoConfiguration          // ← ★ 开启自动配置
@ComponentScan(excludeFilters = {  // ← 自动扫描包
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)
})
public @interface SpringBootApplication {
    // exclude：排除指定自动配置类
    @AliasFor(annotation = EnableAutoConfiguration.class)
    Class<?>[] exclude() default {};

    // excludeName：按名称排除
    @AliasFor(annotation = EnableAutoConfiguration.class)
    String[] excludeName() default {};

    // scanBasePackages：指定扫描包
    @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
    String[] scanBasePackages() default {};
}
```

---

## 二、@EnableAutoConfiguration —— 自动配置入口

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage           // ← 注册自动配置包
@Import(AutoConfigurationImportSelector.class)  // ← ★ 核心：导入自动配置选择器
public @interface EnableAutoConfiguration {
    Class<?>[] exclude() default {};
    String[] excludeName() default {};
}
```

---

## 三、AutoConfigurationImportSelector —— 加载自动配置类

```java
public class AutoConfigurationImportSelector
        implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
        BeanFactoryAware, EnvironmentAware, Ordered {

    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        }
        // ★ 获取自动配置 Entry
        AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
        return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
    }

    protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        }
        AnnotationAttributes attributes = getAttributes(annotationMetadata);

        // ★ 从 spring.factories 或 AutoConfiguration.imports 加载所有候选配置类
        List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);

        // 去重
        configurations = removeDuplicates(configurations);

        // 处理 exclude（排除指定配置类）
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);

        // ★ 使用 AutoConfigurationImportFilter 过滤（@ConditionalOnClass 等）
        configurations = getConfigurationClassFilter().filter(configurations);

        // 发布自动配置导入事件
        fireAutoConfigurationImportEvents(configurations, exclusions);

        return new AutoConfigurationEntry(configurations, exclusions);
    }

    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        // Spring Boot 2.7+ 使用 ImportCandidates
        List<String> configurations = ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader())
                .getCandidates();
        // Spring Boot 2.7 之前从 META-INF/spring.factories 加载
        // configurations = SpringFactoriesLoader.loadFactoryNames(
        //     EnableAutoConfiguration.class, getBeanClassLoader());
        return configurations;
    }
}
```

---

## 四、自动配置加载机制（spring.factories → AutoConfiguration.imports）

### Spring Boot 2.7 之前：META-INF/spring.factories

```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
  org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,\
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
  ...（共 100+ 个）
```

### Spring Boot 2.7+：META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports

```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
...
```

---

## 五、@Conditional 条件注解源码

```java
// @ConditionalOnClass —— 类路径存在指定类时才生效
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)  // ← 实际判断逻辑
public @interface ConditionalOnClass {
    Class<?>[] value() default {};
    String[] name() default {};
}

// OnClassCondition.java（简化）
class OnClassCondition extends FilteringSpringBootCondition {
    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
        ClassLoader classLoader = context.getClassLoader();
        ConditionMessage matchMessage = ConditionMessage.empty();

        // 获取注解中配置的类名
        List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
        if (onClasses != null) {
            // 检查类是否存在
            List<String> missing = filter(onClasses, ClassNameFilter.MISSING, classLoader);
            if (!missing.isEmpty()) {
                return ConditionOutcome.noMatch(
                        ConditionMessage.forCondition(ConditionalOnClass.class)
                                .didNotFind("required class", "required classes")
                                .items(Style.QUOTE, missing));
            }
        }
        return ConditionOutcome.match(matchMessage);
    }
}
```

### 常用条件注解汇总

```java
@ConditionalOnClass(RedisTemplate.class)      // 类路径存在 RedisTemplate
@ConditionalOnMissingClass("org.redis.Client") // 类路径不存在指定类
@ConditionalOnBean(DataSource.class)           // 容器中存在 DataSource Bean
@ConditionalOnMissingBean(DataSource.class)    // 容器中不存在 DataSource Bean（最常用！）
@ConditionalOnProperty(prefix="spring.redis", name="host") // 配置文件存在指定属性
@ConditionalOnWebApplication                   // 是 Web 应用
@ConditionalOnNotWebApplication                // 不是 Web 应用
@ConditionalOnExpression("${myapp.feature.enabled:true}") // SpEL 表达式为 true
@ConditionalOnResource(resources="classpath:redis.conf")  // 存在指定资源文件
```

---

## 六、典型自动配置类解析 —— DataSourceAutoConfiguration

```java
@AutoConfiguration(before = SqlInitializationAutoConfiguration.class)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })  // 有 DataSource 类
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")     // 没有 r2dbc
@EnableConfigurationProperties(DataSourceProperties.class)              // 绑定配置
@Import(DataSourcePoolMetadataProvidersConfiguration.class)
@Import({ DataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {

    // 嵌入式数据库配置（H2/HSQL/Derby）
    @Configuration(proxyBeanMethods = false)
    @Conditional(EmbeddedDatabaseCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import(EmbeddedDataSourceConfiguration.class)
    protected static class EmbeddedDatabaseConfiguration {}

    // 连接池配置（HikariCP/Tomcat/DBCP2/c3p0）
    @Configuration(proxyBeanMethods = false)
    @Conditional(PooledDataSourceCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import({ DataSourceConfiguration.Hikari.class,         // 优先 HikariCP
              DataSourceConfiguration.Tomcat.class,
              DataSourceConfiguration.Dbcp2.class,
              DataSourceConfiguration.OracleUcp.class,
              DataSourceConfiguration.Generic.class,
              DataSourceJmxConfiguration.class })
    protected static class PooledDataSourceConfiguration {}
}

// DataSourceProperties.java —— 配置属性绑定
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties implements BeanFactoryAware, InitializingBean {
    private String url;
    private String username;
    private String password;
    private String driverClassName;
    // ... getter/setter
}
```

---

## 七、SpringApplication.run() —— 启动流程源码

```java
// SpringApplication.java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return new SpringApplication(primarySource).run(args);
}

public ConfigurableApplicationContext run(String... args) {
    Startup startup = Startup.create();

    // ① 从 spring.factories 加载 SpringApplicationRunListener（如 EventPublishingRunListener）
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting(bootstrapContext, this.mainApplicationClass);

    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

        // ② 准备 Environment（加载配置文件、系统环境变量等）
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);

        // ③ 打印 Banner（就是那个 Spring 的 ASCII 图）
        Banner printedBanner = printBanner(environment);

        // ④ 创建 ApplicationContext
        //    Web 环境 → AnnotationConfigServletWebServerApplicationContext
        //    非 Web → AnnotationConfigApplicationContext
        context = createApplicationContext();
        context.setApplicationStartup(this.applicationStartup);

        // ⑤ 准备 Context（注册主启动类 BeanDefinition 等）
        prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);

        // ⑥ ★ 刷新 Context（调用 refresh()，触发所有 Bean 初始化）
        refreshContext(context);

        // ⑦ 刷新后处理（空实现，子类扩展）
        afterRefresh(context, applicationArguments);

        startup.started();
        listeners.started(context, startup.timeTakenToStarted());

        // ⑧ 执行所有 ApplicationRunner 和 CommandLineRunner
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        listeners.failed(context, ex);
        throw new IllegalStateException(ex);
    }

    listeners.ready(context, startup.ready());
    return context;
}
```

---

## 八、自定义 Starter 的实现原理

```
自定义 Starter 的最佳实践：

my-spring-boot-starter/
├── my-spring-boot-autoconfigure/         ← 自动配置模块
│   ├── src/main/java/
│   │   └── com/example/
│   │       ├── MyAutoConfiguration.java  ← 自动配置类
│   │       └── MyProperties.java         ← 配置属性
│   └── src/main/resources/
│       └── META-INF/spring/
│           └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
│               内容：com.example.MyAutoConfiguration
└── my-spring-boot-starter/               ← 仅作为依赖聚合，无代码
    └── pom.xml（依赖 autoconfigure 和目标库）
```

```java
// MyAutoConfiguration.java
@AutoConfiguration
@ConditionalOnClass(MyService.class)                           // 目标类存在才生效
@EnableConfigurationProperties(MyProperties.class)             // 绑定配置
@ConditionalOnProperty(prefix = "my", name = "enabled",
        havingValue = "true", matchIfMissing = true)           // 配置开关
public class MyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                                  // 用户未自定义时才创建
    public MyService myService(MyProperties properties) {
        return new MyService(properties.getUrl(), properties.getTimeout());
    }
}

// MyProperties.java
@ConfigurationProperties(prefix = "my")
public class MyProperties {
    private String url = "http://localhost:8080";
    private int timeout = 3000;
    // getter/setter
}
```

---

## 九、常见面试问题

| 问题 | 答案要点 |
|------|---------|
| Spring Boot 自动配置原理？ | @EnableAutoConfiguration → AutoConfigurationImportSelector → 加载 AutoConfiguration.imports → @Conditional 过滤 |
| spring.factories 和 AutoConfiguration.imports 的区别？ | Spring Boot 2.7+ 新增 imports 文件，2.x 用 spring.factories，3.x 完全切换 |
| @ConditionalOnMissingBean 的典型用途？ | 自动配置提供默认 Bean，用户自定义后自动退出 |
| 如何排除某个自动配置？ | `@SpringBootApplication(exclude = xxx)` 或 `spring.autoconfigure.exclude=xxx` |
| Spring Boot 的启动流程？ | 创建 SpringApplication → prepareEnvironment → createApplicationContext → refresh → callRunners |

---

## 十、相关源码文件

- [[../01_IoC容器源码/02、ApplicationContext启动流程（refresh方法）]]
- [[../05_SpringMVC源码/01、DispatcherServlet请求处理源码]]
- [[02、SpringMVC自动配置]]
