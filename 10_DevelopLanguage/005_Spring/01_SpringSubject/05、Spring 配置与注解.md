###### 1. Spring 有哪些配置方式？

Spring 提供三种配置方式，体现了框架十几年的演进历史：

**XML 配置（传统方式）**：Spring 最早的配置方式，用 XML 文件定义 Bean 和依赖关系。优点是配置和代码分离；缺点是没有类型检查，重命名类后 XML 不会自动更新，维护成本高。现在新项目基本不用了。

```xml
<bean id="userService" class="com.example.UserService">
    <property name="userDao" ref="userDao"/>
</bean>
```

**注解配置（现代首选）**：Spring 2.5+ 引入，用 `@Component`/`@Service`/`@Autowired` 等注解实现自动扫描和注入，代码简洁，是目前主流方式。

```java
@Service
public class UserService {
    @Autowired
    private UserDao userDao;
}
```

**Java Config（显式控制）**：Spring 3.0+ 引入，用 `@Configuration` + `@Bean` 完全替代 XML，有类型检查，IDE 重构友好，适合第三方库集成或需要精细控制 Bean 创建过程的场景。

```java
@Configuration
public class AppConfig {
    @Bean
    public UserService userService() {
        return new UserService(userDao());
    }
}
```

**实际项目通常混合使用**：业务代码用注解，第三方库集成和复杂依赖用 Java Config，遗留系统可能还保留部分 XML。

📖 [[../../../24_SpringKnowledge/05_配置与注解/01、配置方式与核心注解#一、三种配置方式对比]]

---

###### 2. XML 配置和 Java 配置的区别是什么？

**类型安全**：XML 配置是字符串，写错类名运行时才报错；Java Config 是 Java 代码，编译期就能检查类型错误，更安全。

**重构支持**：XML 里的类名是字符串，重命名类后 XML 不会自动更新；Java Config 里的类引用支持 IDE 自动重构。

**条件化配置**：Java Config 有强大的 `@Conditional` 系列注解，可以根据类路径、配置项、Bean 是否存在等条件动态决定是否创建 Bean；XML 的条件化能力很弱。

**动态性**：XML 修改不用重新编译，可以在不打包的情况下修改配置（某些场景下有价值）；Java Config 修改需要重新编译。

**源码差异**：XML 配置通过 `BeanDefinitionReader` 解析 XML 生成 `BeanDefinition`；Java Config 通过 `ConfigurationClassPostProcessor` 解析 `@Configuration` 类，并用 CGLIB 代理增强配置类，确保 `@Bean` 方法的单例语义。

📖 [[../../../24_SpringKnowledge/05_配置与注解/01、配置方式与核心注解#一、三种配置方式对比]]

---

###### 3. @Configuration 注解的作用是什么？

`@Configuration` 标记一个类是 Spring 配置类，相当于 XML 里的 `<beans>` 根元素。

但它和普通 `@Component` 有个关键区别：**`@Configuration` 类会被 CGLIB 代理增强**。

这个增强有什么用？来看这段代码：

```java
@Configuration
public class AppConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // 调用了 serviceB() 方法
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}
```

`serviceA()` 里调用了 `serviceB()`，如果没有 CGLIB 增强，这里会直接执行方法，每次都 `new ServiceB()`，导致 ServiceB 不是单例。有了 CGLIB 增强后，`serviceB()` 调用会被拦截，转而去容器里找已有的 ServiceB 实例，保证了单例语义。

如果你用的是 `@Component`（没有 CGLIB 增强），就叫做"Lite 模式"，`@Bean` 方法之间互相调用不会保证单例，要注意。

📖 [[../../../24_SpringKnowledge/05_配置与注解/01、配置方式与核心注解#二、@Configuration 的 CGLIB 增强原理]]

---

###### 4. @Bean 注解的作用是什么？

`@Bean` 加在 `@Configuration` 类的方法上，把该方法的返回值注册为 Spring 容器管理的 Bean。主要用于注册第三方库的对象（无法在源代码上加 `@Component`）。

```java
@Configuration
public class DataConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        ds.setMaximumPoolSize(20);
        return ds;
    }
    
    // 指定多个别名 + 设置为首选
    @Bean(name = {"primaryDS", "mainDataSource"})
    @Primary
    public DataSource primaryDataSource() {
        return new HikariDataSource();
    }
    
    // 指定初始化和销毁方法
    @Bean(initMethod = "init", destroyMethod = "close")
    public ExpensiveResource expensiveResource() {
        return new ExpensiveResource();
    }
}
```

`@Bean` 方法的参数会自动注入容器里的 Bean，比如 `@Bean public Service service(Repository repo)` 中的 `repo` 会自动从容器里找。

📖 [[../../../24_SpringKnowledge/05_配置与注解/01、配置方式与核心注解#三、@Bean 详解]]

---

###### 5. @ComponentScan 注解的作用是什么？

`@ComponentScan` 用来开启组件自动扫描，替代 XML 里的 `<context:component-scan>`。Spring 会扫描指定包路径下带有 `@Component`（及其派生注解）的类，自动注册为 Bean。

```java
@Configuration
@ComponentScan(
    basePackages = "com.example",                                // 扫描包路径
    includeFilters = @Filter(
        type = FilterType.ANNOTATION, classes = Service.class), // 只包含 @Service
    excludeFilters = @Filter(
        type = FilterType.REGEX, pattern = ".*Test.*")          // 排除测试类
)
public class AppConfig { }
```

Spring Boot 的 `@SpringBootApplication` 注解里就包含了 `@ComponentScan`，默认扫描启动类所在包及其子包。

底层实现：`ClassPathScanningCandidateComponentProvider` 扫描包路径下的 `.class` 文件，通过 ASM 读取字节码中的注解信息（不需要加载类），找到候选组件后注册为 `BeanDefinition`。

📖 [[../../../24_SpringKnowledge/05_配置与注解/01、配置方式与核心注解#三、@Bean 详解]]

---

###### 6. @Import 注解的作用是什么？

`@Import` 用于导入其他配置类，实现配置的模块化和按需组装：

```java
@Configuration
@Import({DatabaseConfig.class, SecurityConfig.class, CacheConfig.class})
public class AppConfig {
    // 将多个配置模块组合在一起
}
```

**高级用法：配合 `ImportSelector`** 实现动态条件化导入（这是 Spring Boot 自动配置的核心机制）：

```java
public class FeatureImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        // 根据注解属性、类路径等条件，动态决定导入哪些配置类
        if (isRedisAvailable()) {
            return new String[]{RedisConfig.class.getName()};
        } else {
            return new String[]{LocalCacheConfig.class.getName()};
        }
    }
}
```

`@EnableCaching`、`@EnableScheduling`、`@EnableAsync` 等 `@Enable*` 注解背后，都是通过 `@Import` 导入相应的配置类来实现功能激活的。

📖 [[../../../24_SpringKnowledge/05_配置与注解/01、配置方式与核心注解#四、@Import 与 ImportSelector]]

---

###### 7. @Conditional 系列注解的作用是什么？

`@Conditional` 系列注解提供条件化 Bean 注册机制，是 Spring Boot 自动配置的核心。只有在满足特定条件时，Bean 才会被注册：

```java
@Configuration
public class CacheConfig {
    
    @Bean
    @ConditionalOnClass(RedisTemplate.class)            // Redis 依赖存在时
    @ConditionalOnProperty(name = "cache.enabled", havingValue = "true") // 配置开启时
    @ConditionalOnMissingBean(CacheManager.class)       // 没有自定义 CacheManager 时
    public CacheManager redisCacheManager() {
        return new RedisCacheManager(...);
    }
}
```

**常用条件注解：**

- `@ConditionalOnClass`：类路径下存在指定类时生效
- `@ConditionalOnMissingClass`：类路径下不存在指定类时生效
- `@ConditionalOnBean`：容器中已有指定 Bean 时生效
- `@ConditionalOnMissingBean`：容器中没有指定 Bean 时生效（最常用）
- `@ConditionalOnProperty`：配置属性满足条件时生效
- `@ConditionalOnExpression`：SpEL 表达式为 true 时生效

底层原理：每个 `@Conditional` 注解对应一个实现 `Condition` 接口的类，在 `ConfigurationClassParser` 解析阶段调用 `matches()` 判断条件是否满足。

📖 [[../../../24_SpringKnowledge/05_配置与注解/01、配置方式与核心注解#五、@Conditional 条件装配]]

---

###### 8. @Profile 注解如何使用？

`@Profile` 是 `@Conditional` 的一个特化版本，用于根据激活的 Profile 环境来决定是否注册某个 Bean，实现多环境配置隔离：

```java
// 开发环境：用 H2 内存数据库
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2).build();
    }
}

// 生产环境：用 MySQL
@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://prod-server:3306/app");
        return ds;
    }
}
```

**激活 Profile：**

```properties
# application.properties
spring.profiles.active=dev

# 或命令行
java -jar app.jar --spring.profiles.active=prod
```

`@Profile` 也可以加在 `@Bean` 方法上，做更细粒度的控制。支持 `!dev`（非 dev 环境）、`dev | test`（dev 或 test）等表达式。

📖 [[../../../24_SpringKnowledge/05_配置与注解/01、配置方式与核心注解#六、@Profile 多环境配置]]

---

###### 9. @Value 注解如何使用？

`@Value` 用于注入单个配置属性，支持 `${}` 占位符和 SpEL 表达式：

```java
@Component
public class AppConfig {
    
    // 注入配置属性，带默认值
    @Value("${app.name:MyApp}")
    private String appName;
    
    @Value("${server.port}")
    private int serverPort;
    
    // SpEL 表达式：注入系统属性
    @Value("#{systemProperties['java.version']}")
    private String javaVersion;
    
    // SpEL 表达式：注入其他 Bean 的属性
    @Value("#{appSettings.maxRetry}")
    private int maxRetry;
    
    // 条件表达式
    @Value("#{${feature.debug:false} ? 'DEBUG' : 'PROD'}")
    private String mode;
}
```

注意：`@Value` 注入的是启动时的配置值，运行期改变配置不会动态刷新（需要配合 Spring Cloud Config 的 `@RefreshScope` 才能热更新）。

📖 [[../../../24_SpringKnowledge/05_配置与注解/01、配置方式与核心注解#七、@Value 与 @PropertySource]]

---

###### 10. @PropertySource 注解的作用是什么？

`@PropertySource` 用于把外部 properties 文件加载到 Spring 的 `Environment` 中，之后可以通过 `@Value` 或 `Environment.getProperty()` 读取：

```java
@Configuration
@PropertySource(value = "classpath:app.properties", ignoreResourceNotFound = true)
@PropertySource(value = "classpath:${spring.profiles.active:dev}/database.properties")
public class AppConfig {
    
    @Autowired
    private Environment env;
    
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(env.getProperty("database.url"));
        ds.setUsername(env.getProperty("database.username"));
        return ds;
    }
}
```

Spring Boot 项目通常不需要手动加 `@PropertySource`，`application.properties` / `application.yml` 会被自动加载。

📖 [[../../../24_SpringKnowledge/05_配置与注解/01、配置方式与核心注解#七、@Value 与 @PropertySource]]

---

###### 11. @Qualifier 注解的作用是什么？

当容器里有多个同类型的 Bean 时，`@Autowired` 不知道该注入哪个，`@Qualifier` 用来明确指定 Bean 名称解歧义：

```java
@Configuration
public class DataSourceConfig {
    @Bean
    @Qualifier("primaryDataSource")
    public DataSource primaryDataSource() { return new PrimaryDataSource(); }
    
    @Bean
    @Qualifier("secondaryDataSource")
    public DataSource secondaryDataSource() { return new SecondaryDataSource(); }
}

@Service
public class UserService {
    @Autowired
    @Qualifier("primaryDataSource") // 明确指定用哪个
    private DataSource dataSource;
}
```

`@Qualifier` 也可以自定义，作为元注解来创建语义更明确的限定符注解（比如 `@PrimaryDB`、`@ReadOnlyDB`）。

📖 [[../../../24_SpringKnowledge/05_配置与注解/01、配置方式与核心注解#八、@Qualifier 与 @Primary]]

---

###### 12. @Primary 注解的作用是什么？

当有多个同类型 Bean 时，`@Primary` 标记"首选 Bean"。如果注入点没有用 `@Qualifier` 指定，就优先用 `@Primary` 的那个：

```java
@Configuration
public class DataSourceConfig {
    @Bean
    @Primary // 默认用这个
    public DataSource primaryDataSource() { return new PrimaryDataSource(); }
    
    @Bean
    public DataSource secondaryDataSource() { return new SecondaryDataSource(); }
}

@Service
public class UserService {
    @Autowired // 自动注入 primaryDataSource
    private DataSource dataSource;
}
```

`@Primary` vs `@Qualifier`：`@Primary` 是"全局默认"，`@Qualifier` 是"精确指定"。有 `@Qualifier` 时忽略 `@Primary`。

📖 [[../../../24_SpringKnowledge/05_配置与注解/01、配置方式与核心注解#八、@Qualifier 与 @Primary]]

---

###### 13. @Lazy 注解的作用是什么？

`@Lazy` 让 Bean 延迟初始化，容器启动时不创建，第一次被用到时才创建，用于减少启动时间和内存占用：

```java
@Bean
@Lazy
public ExpensiveService expensiveService() {
    return new ExpensiveService(); // 第一次使用时才执行这段代码
}

// 也可以加在注入点，只对这一个注入点延迟
@Service
public class OrderService {
    @Autowired
    @Lazy
    private ReportService reportService; // 第一次调用 reportService 方法时才初始化
}
```

全局开启：`spring.main.lazy-initialization=true`。用 `@Lazy(false)` 强制某个 Bean 不延迟。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#八、@Lazy 懒加载]]

---

###### 14. @Scope 注解的作用是什么？

`@Scope` 定义 Bean 的作用域。Web 应用里的 request/session 作用域 Bean 注入到 singleton Bean 时，需要加 `proxyMode` 创建作用域代理，否则每次都拿到同一个对象（singleton 初始化时注入的那个）：

```java
// singleton Bean（默认）
@Bean
@Scope("singleton")
public Service singletonService() { return new SingletonService(); }

// prototype：每次获取新实例
@Bean
@Scope("prototype")
public Service prototypeService() { return new PrototypeService(); }

// request 作用域 + 代理（重要！）
@Bean
@Scope(
    value = WebApplicationContext.SCOPE_REQUEST,
    proxyMode = ScopedProxyMode.TARGET_CLASS  // 必须加，否则注入到 singleton 时有问题
)
public RequestContext requestContext() { return new RequestContext(); }
```

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#三、Bean 的 6 种作用域]]

---

###### 15. @PostConstruct 和 @PreDestroy 注解的作用是什么？

这两个注解用于管理 Bean 的生命周期回调：

- **`@PostConstruct`**：在 Bean 属性注入完成之后、容器把 Bean 放入使用状态之前执行，用于初始化操作（建立连接、加载数据等）
- **`@PreDestroy`**：在容器销毁 Bean 之前执行，用于清理资源（关闭连接、清空缓存等）

```java
@Service
public class DatabaseConnection {
    private Connection connection;
    
    @PostConstruct
    public void init() {
        this.connection = dataSource.getConnection(); // 属性注入后初始化
        log.info("数据库连接建立");
    }
    
    @PreDestroy
    public void cleanup() {
        if (connection != null) {
            connection.close(); // Bean 销毁前关闭连接
        }
    }
}
```

**执行时序：**`@PostConstruct` 在 `afterPropertiesSet()`（`InitializingBean` 接口）之前；`@PreDestroy` 在 `destroy()`（`DisposableBean` 接口）之前。推荐用这两个注解，语义更清晰且是 Java 标准注解（JSR-250），不依赖 Spring 接口。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#四、Bean 生命周期的 9 个阶段]]
