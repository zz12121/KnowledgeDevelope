###### 1. Spring 有哪些配置方式？
Spring提供了三种主要的配置方式，体现了框架的演进历程：
**XML配置**：Spring最早期的配置方式，通过XML文件定义Bean及其依赖关系
```xml
<beans xmlns="http://www.springframework.org/schema/beans">
    <bean id="userService" class="com.example.UserService">
        <property name="userDao" ref="userDao"/>
    </bean>
</beans>
```
**注解配置**：Spring 2.5+引入，使用`@Component`、`@Service`、`@Autowired`等注解实现自动扫描和依赖注入
```java
@Service
public class UserService {
    @Autowired
    private UserDao userDao;
}
```
**Java配置**：Spring 3.0+引入，使用`@Configuration`和`@Bean`完全替代XML配置
```java
@Configuration
public class AppConfig {
    @Bean
    public UserService userService() {
        return new UserService(userDao());
    }
}
```
**混合配置**：实际项目中常组合使用上述方式，通过`@ImportResource`引入XML配置，或通过`<context:component-scan>`扫描注解组件
###### 2. XML 配置和 Java 配置的区别是什么？

|**特性**​|**XML配置**​|**Java配置**​|
|---|---|---|
|**类型安全**​|运行时才能发现类型错误|编译期类型检查，更安全|
|**重构友好**​|重命名类或方法需手动修改XML|IDE自动重构支持良好|
|**可读性**​|配置复杂时难以维护|Java代码更符合开发者习惯|
|**条件化配置**​|有限支持|强大的`@Conditional`系列注解支持|
|**动态性**​|修改配置无需重新编译|配置变更需重新编译|
**源码设计差异**：XML配置通过`BeanDefinitionReader`解析XML元素生成`BeanDefinition`；Java配置通过`ConfigurationClassPostProcessor`处理`@Configuration`类，使用CGLIB代理增强配置类方法调用
###### 3. @Configuration 注解的作用是什么？
`@Configuration`是Spring Java配置的核心注解，其主要作用包括：
**标识配置类**：标记一个类为Spring配置类，替代传统的XML配置文件
```java
@Configuration
public class AppConfig {
    // 相当于XML中的<beans>根元素
}
```
**启用CGLIB代理**：这是`@Configuration`与普通`@Component`的关键区别。被`@Configuration`标记的类会被CGLIB代理，确保`@Bean`方法调用被拦截，维护Bean的单例特性
**源码机制**：在`ConfigurationClassPostProcessor`中，Spring会检查类是否有`@Configuration`注解，如果有则通过`ConfigurationClassEnhancer`使用CGLIB增强，重写`@Bean`方法逻辑，确保多次调用返回同一实例。
**组合配置**：支持与`@ComponentScan`、`@Import`、`@PropertySource`等注解组合使用
###### 4. @Bean 注解的作用是什么？
`@Bean`注解用于标注方法，将该方法的返回值注册为Spring容器管理的Bean
**基本用法**：
```java
@Configuration
public class DataConfig {
    @Bean
    public DataSource dataSource() {
        return new DriverManagerDataSource(url, username, password);
    }
    
    @Bean(name = {"primaryDS", "mainDataSource"}) // 指定多个名称
    @Primary
    public DataSource primaryDataSource() {
        return new DataSource();
    }
}
```
**生命周期控制**：
```java
@Bean(initMethod = "init", destroyMethod = "close")
public ExpensiveResource expensiveResource() {
    return new ExpensiveResource();
}
```
**源码解析**：`@Bean`方法被`ConfigurationClassParser`解析为`BeanDefinition`，其`beanClass`属性实际为`FactoryMethod`类型，Spring通过调用工厂方法创建Bean实例
###### 5. @ComponentScan 注解的作用是什么？
`@ComponentScan`用于自动扫描和注册组件，替代XML中的`<context:component-scan>`
**基本配置**：
```java
@Configuration
@ComponentScan(
    basePackages = "com.example",
    includeFilters = @Filter(type = FilterType.ANNOTATION, classes = Service.class),
    excludeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Test.*")
)
public class AppConfig {
}
```
**扫描机制**：Spring使用`ClassPathScanningCandidateComponentProvider`扫描指定包路径下的类，检查是否带有`@Component`或其派生注解，符合条件的类会被注册为Bean定义
###### 6. @Import 注解的作用是什么？
`@Import`用于导入其他配置类，实现配置的模块化
```java
@Configuration
@Import({DatabaseConfig.class, SecurityConfig.class})
public class AppConfig {
}

@Configuration
public class DatabaseConfig {
    @Bean
    public DataSource dataSource() {
        // 数据源配置
    }
}
```
**高级用法**：配合`ImportSelector`接口实现条件化导入
```java
public class ProfileBasedImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        // 根据环境条件动态返回配置类
    }
}
```
###### 7. @Conditional 系列注解的作用是什么？
`@Conditional`系列注解提供条件化Bean注册机制，是Spring Boot自动配置的核心**常用条件注解**：
- `@ConditionalOnClass`：类路径下存在指定类时生效
- `@ConditionalOnMissingBean`：容器中不存在指定Bean时生效
- `@ConditionalOnProperty`：配置属性满足条件时生效
```java
@Configuration
public class CacheConfig {
    @Bean
    @ConditionalOnClass(RedisTemplate.class)
    @ConditionalOnProperty(name = "cache.enabled", havingValue = "true")
    public CacheManager redisCacheManager() {
        return new RedisCacheManager();
    }
}
```
**源码原理**：每个`@Conditional`注解对应一个实现`Condition`接口的类，在`ConfigurationClassParser`解析阶段会调用`matches()`方法进行条件判断
###### 8. @Profile 注解如何使用？
`@Profile`实现基于环境的配置切换
**配置类级别**：
```java
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource devDataSource() {
        return new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.H2).build();
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource prodDataSource() {
        return new DriverManagerDataSource("jdbc:mysql://prod:3306/db");
    }
}
```
**激活Profile**：
```properties
spring.profiles.active=dev
```
###### 9. @Value 注解如何使用？
`@Value`用于属性注入，支持SpEL表达式
**基本注入**：
```java
@Component
public class AppConfig {
    @Value("${app.name:DefaultApp}") // 默认值支持
    private String appName;
    
    @Value("${server.port}")
    private int serverPort;
    
    @Value("#{systemProperties['java.version']}") // SpEL表达式
    private String javaVersion;
}
```
**配置属性文件**：
```properties
# application.properties
database.url=jdbc:mysql://localhost:3306/mydb
app.timeout=5000
```
###### 10. @PropertySource 注解的作用是什么？
`@PropertySource`用于加载属性文件到Spring Environment中
```java
@Configuration
@PropertySource(value = "classpath:app.properties", ignoreResourceNotFound = true)
@PropertySource(value = "classpath:${ENV_VAR:dev}/config.properties")
public class AppConfig {
    @Autowired
    private Environment env;
    
    @Bean
    public DataSource dataSource() {
        String url = env.getProperty("database.url");
        // 创建数据源
    }
}
```
**源码机制**：`PropertySource`注解被`PropertySourceProcessor`处理，将属性文件内容加载到`Environment`的`PropertySource`列表中
###### 11. @Qualifier 注解的作用是什么？
`@Qualifier`用于解决自动装配时的歧义性问题
```java
@Configuration
public class DataSourceConfig {
    @Bean
    @Qualifier("primaryDataSource")
    public DataSource primaryDataSource() {
        return new PrimaryDataSource();
    }
    
    @Bean
    @Qualifier("secondaryDataSource")
    public DataSource secondaryDataSource() {
        return new SecondaryDataSource();
    }
}

@Service
public class UserService {
    @Autowired
    @Qualifier("primaryDataSource") // 明确指定Bean名称
    private DataSource dataSource;
}
```
###### 12. @Primary 注解的作用是什么？
`@Primary`标记首选的Bean，当有多个相同类型的Bean时，优先使用被`@Primary`标记的Bean
```java
@Configuration
public class DataSourceConfig {
    @Bean
    @Primary
    public DataSource primaryDataSource() {
        return new PrimaryDataSource();
    }
    
    @Bean
    public DataSource secondaryDataSource() {
        return new SecondaryDataSource();
    }
}

@Service
public class UserService {
    @Autowired // 会自动注入primaryDataSource
    private DataSource dataSource;
}
```
###### 13. @Lazy 注解的作用是什么？
`@Lazy`实现Bean的延迟初始化，减少应用启动时间
```java
@Configuration
public class AppConfig {
    @Bean
    @Lazy
    public ExpensiveBean expensiveBean() {
        // 这个Bean在第一次被使用时才会初始化
        return new ExpensiveBean();
    }
}
```
**作用域**：可在配置类、`@Bean`方法或注入点使用，控制粒度灵活
###### 14. @Scope 注解的作用是什么？
`@Scope`定义Bean的作用域，控制Bean的生命周期和创建方式
```java
@Configuration
public class ScopeConfig {
    @Bean
    @Scope("singleton") // 默认，整个容器一个实例
    public Service singletonService() {
        return new SingletonService();
    }
    
    @Bean
    @Scope("prototype") // 每次获取新实例
    public Service prototypeService() {
        return new PrototypeService();
    }
    
    @Bean
    @Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
    public UserSession userSession() {
        return new UserSession();
    }
}
```
###### 15. @PostConstruct 和 @PreDestroy 注解的作用是什么？
这两个注解用于管理Bean的生命周期回调
```java
@Service
public class DatabaseConnection {
    private Connection connection;
    
    @PostConstruct
    public void init() {
        // 在Bean属性设置完成后执行，相当于init-method
        this.connection = createConnection();
    }
    
    @PreDestroy
    public void cleanup() {
        // 在Bean销毁前执行，相当于destroy-method
        if (connection != null) {
            connection.close();
        }
    }
    
    private Connection createConnection() {
        // 创建数据库连接
        return null;
    }
}
```
**执行时机**：`@PostConstruct`在Bean属性注入完成后、`afterPropertiesSet()`之后执行；`@PreDestroy`在Bean销毁前，`DisposableBean.destroy()`之前执行