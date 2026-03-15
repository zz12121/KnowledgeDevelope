# Spring 容器与上下文

## 1 ApplicationContext 核心功能

`ApplicationContext` 是 Spring 的核心接口，是高级 IoC 容器。除了完整的 DI 功能外，还集成了：

- **Bean 生命周期管理**：启动时预实例化所有 singleton Bean（非懒加载），底层通过 `DefaultListableBeanFactory` 的 `beanDefinitionMap`（ConcurrentHashMap）存储 Bean 定义
- **事件驱动模型**：基于观察者模式，通过 `ApplicationEventPublisher` 发布事件
- **国际化支持**：通过 `MessageSource` 接口，支持多 Locale 消息解析
- **资源抽象**：通过 `ResourceLoader` 统一处理类路径、文件系统、URL 等资源

## 2 常用实现类

```
ApplicationContext
└── ConfigurableApplicationContext
    └── AbstractApplicationContext
        ├── AbstractRefreshableApplicationContext
        │   ├── ClassPathXmlApplicationContext        → 从类路径加载 XML
        │   └── FileSystemXmlApplicationContext       → 从文件系统加载 XML
        └── GenericApplicationContext
            ├── AnnotationConfigApplicationContext    → 基于 @Configuration 类（常用）
            └── AnnotationConfigServletWebServerApplicationContext  → Spring Boot Web 应用
```

- `ClassPathXmlApplicationContext`：`new ClassPathXmlApplicationContext("applicationContext.xml")`
- `AnnotationConfigApplicationContext`：`new AnnotationConfigApplicationContext(AppConfig.class)` 或 扫描包路径
- `AnnotationConfigServletWebServerApplicationContext`：Spring Boot Web 应用默认上下文，内嵌 Web 服务器

所有实现类都通过统一的 `refresh()` 方法初始化容器，体现了**模板方法模式**。

## 3 容器启动流程：refresh() 的 12 个步骤

`AbstractApplicationContext.refresh()` 是 Spring 容器启动的核心，定义了 12 个标准步骤：

```
prepareRefresh()                     → 准备环境，校验必需属性，记录启动时间
obtainFreshBeanFactory()             → 获取/刷新 BeanFactory（DefaultListableBeanFactory）
prepareBeanFactory()                 → 配置 BeanFactory：类加载器、BeanPostProcessor、内置依赖
postProcessBeanFactory()             → 子类扩展点，web 环境在此注册 web 相关 Bean
invokeBeanFactoryPostProcessors()    → 执行 BeanFactoryPostProcessor（核心！）
registerBeanPostProcessors()         → 注册 BeanPostProcessor 到容器
initMessageSource()                  → 初始化国际化消息源
initApplicationEventMulticaster()   → 初始化事件广播器
onRefresh()                         → 子类扩展点，Spring Boot 在此启动内嵌服务器
registerListeners()                  → 注册 ApplicationListener
finishBeanFactoryInitialization()   → 实例化所有非懒加载 singleton Bean
finishRefresh()                      → 发布 ContextRefreshedEvent，完成启动
```

**关键步骤解析**：

`invokeBeanFactoryPostProcessors()`：执行 `BeanFactoryPostProcessor`，其中 `ConfigurationClassPostProcessor` 负责解析 `@Configuration` 类、执行 `@ComponentScan`、处理 `@Import` 等——这是注解驱动的核心处理步骤。

`registerBeanPostProcessors()`：注册 `AutowiredAnnotationBeanPostProcessor`（处理 `@Autowired`）、`CommonAnnotationBeanPostProcessor`（处理 `@PostConstruct`）等处理器。

`finishBeanFactoryInitialization()`：调用 `DefaultListableBeanFactory.preInstantiateSingletons()`，触发所有 singleton Bean 的 `getBean()` 级联创建。

## 4 BeanDefinition

`BeanDefinition` 是 Bean 的"创建蓝图"，包含实例化 Bean 所需的所有元数据，但它本身不是 Bean 实例。

**主要实现类**：
- `GenericBeanDefinition`：通用场景，现代 Spring 默认
- `RootBeanDefinition`：合并后的最终定义，用于实际实例化
- `ScannedGenericBeanDefinition`：`@Component` 扫描生成，携带注解元数据
- `ConfigurationClassBeanDefinition`：`@Bean` 方法生成，包含工厂方法信息

**关键元数据**：Bean 类名、作用域（singleton/prototype/...）、懒加载标志、构造器参数、属性值、初始化/销毁方法名、工厂信息。

BeanDefinition 通过 `BeanDefinitionRegistry`（通常是 `DefaultListableBeanFactory`）注册到容器，内部使用 `ConcurrentHashMap<String, BeanDefinition>（beanDefinitionMap）`存储，以 beanName 为键。

## 5 BeanFactoryPostProcessor

`BeanFactoryPostProcessor` 是在 **Bean 实例化前**修改 BeanDefinition 的扩展点：

```java
@Component
public class CustomBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        BeanDefinition bd = beanFactory.getBeanDefinition("myService");
        bd.getPropertyValues().add("timeout", 5000);     // 修改属性
        bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);      // 修改作用域
    }
}
```

**典型内置实现**：
- `PropertySourcesPlaceholderConfigurer`：替换 `${...}` 占位符为实际属性值
- `ConfigurationClassPostProcessor`：解析 `@Configuration`、`@ComponentScan`、`@Import` 等

注意与 `BeanPostProcessor` 的区别：前者操作 **BeanDefinition**（实例化前），后者操作 **Bean 实例**（实例化后）。

## 6 BeanPostProcessor

`BeanPostProcessor` 在 Bean 实例化和依赖注入完成后、初始化回调执行前后介入，是 Spring 扩展的核心机制：

```java
public interface BeanPostProcessor {
    // 在初始化回调（@PostConstruct、afterPropertiesSet）之前调用
    Object postProcessBeforeInitialization(Object bean, String beanName);

    // 在初始化回调之后调用（AOP 代理在这里创建！）
    Object postProcessAfterInitialization(Object bean, String beanName);
}
```

**重要内置实现**：
- `AutowiredAnnotationBeanPostProcessor`：处理 `@Autowired`、`@Value` 注入
- `CommonAnnotationBeanPostProcessor`：处理 `@PostConstruct`、`@PreDestroy`、`@Resource`
- `ApplicationContextAwareProcessor`：注入 `ApplicationContext` 等 Aware 对象
- `AnnotationAwareAspectJAutoProxyCreator`：在 `postProcessAfterInitialization` 中为匹配切面的 Bean 创建 AOP 代理

## 7 FactoryBean vs BeanFactory

名字很像，但完全不同：

**BeanFactory**：Spring 容器的基础接口，负责管理所有 Bean 的创建和依赖注入。

**FactoryBean**：一种特殊的 Bean，用于封装复杂对象的创建逻辑：

```java
@Component("sqlSessionFactory")
public class SqlSessionFactoryBean implements FactoryBean<SqlSessionFactory> {
    @Override
    public SqlSessionFactory getObject() throws Exception {
        return buildSqlSessionFactory();  // 返回的是 SqlSessionFactory，不是 SqlSessionFactoryBean
    }

    @Override
    public Class<?> getObjectType() {
        return SqlSessionFactory.class;
    }

    @Override
    public boolean isSingleton() { return true; }
}
```

**获取规则**：
- `ctx.getBean("sqlSessionFactory")` → 返回 `getObject()` 的结果（SqlSessionFactory）
- `ctx.getBean("&sqlSessionFactory")` → 返回 FactoryBean 实例本身（带 `&` 前缀）

**设计价值**：隔离复杂对象的构建逻辑（比如 MyBatis 的 `SqlSessionFactoryBean`），使配置更简洁。

## 8 ApplicationContextAware

Bean 实现 `ApplicationContextAware` 接口后，容器会在初始化阶段注入 `ApplicationContext` 引用：

```java
@Component
public class SpringContextHolder implements ApplicationContextAware {
    private static ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        context = ctx;
    }

    // 静态工具方法，用于编程式获取 Bean
    public static <T> T getBean(Class<T> type) {
        return context.getBean(type);
    }
}
```

`ApplicationContextAwareProcessor`（BeanPostProcessor 实现）在 `postProcessBeforeInitialization` 中检测到该接口并调用注入方法。

**注意**：大多数场景优先用依赖注入（`@Autowired`），`ApplicationContextAware` 适合需要静态访问容器或动态获取 Bean 的工具类场景，但会让 Bean 与 Spring API 耦合。

---

相关面试题 → [[../../10_DevelopLanguage/005_Spring/01_SpringSubject/06、Spring 容器与上下文|06、Spring容器与上下文]]
