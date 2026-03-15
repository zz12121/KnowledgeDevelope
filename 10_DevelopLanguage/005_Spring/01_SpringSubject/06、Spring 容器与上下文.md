###### 1. 什么是 ApplicationContext？

`ApplicationContext` 是 Spring 的核心容器接口，是 `BeanFactory` 的超集。除了 `BeanFactory` 提供的基础 DI 功能，它还内置了一系列企业级服务：

- **Bean 生命周期管理**：启动时预实例化所有单例 Bean，底层用 `DefaultListableBeanFactory` 的 `beanDefinitionMap` 存储 Bean 定义
- **事件发布**：通过 `ApplicationEventPublisher` 发布事件，`ApplicationListener` 接收，基于观察者模式
- **国际化**：通过 `MessageSource` 接口支持多语言消息加载
- **资源抽象**：通过 `ResourceLoader` 统一处理类路径、文件系统、URL 等不同来源的资源
- **环境与配置**：通过 `Environment` 接口管理配置属性和 Profile

**和 BeanFactory 的关键差异**：`ApplicationContext` 启动时就预实例化所有单例 Bean（及早发现配置错误）；`BeanFactory` 是懒加载，第一次 `getBean()` 才创建。日常开发几乎都用 `ApplicationContext`。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/02、Spring容器与上下文#一、ApplicationContext 核心功能]]

---

###### 2. ApplicationContext 有哪些常见的实现类？

Spring 提供了多种实现，对应不同的使用场景：

**`ClassPathXmlApplicationContext`**：从类路径加载 XML 配置，最经典的实现，现在已少用。

**`FileSystemXmlApplicationContext`**：从文件系统路径加载 XML 配置。

**`AnnotationConfigApplicationContext`**：基于 Java 配置类（`@Configuration`）的现代实现，是非 Web 应用的首选：

```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
```

**`AnnotationConfigServletWebServerApplicationContext`**：Spring Boot Web 应用默认使用的上下文，内嵌 Tomcat/Jetty 并支持自动配置。

**`GenericWebApplicationContext`**：Spring MVC 的标准 Web 应用上下文。

所有实现类都通过统一的 `refresh()` 方法初始化容器，体现了模板方法模式。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/02、Spring容器与上下文#二、实现类继承体系]]

---

###### 3. Spring 容器的启动流程是怎样的？

Spring 容器的启动核心是 `AbstractApplicationContext.refresh()` 方法，包含 12 个标准步骤：

**1. `prepareRefresh()`**：初始化环境变量，校验必需属性，记录启动时间戳，设置 active 标志。

**2. `obtainFreshBeanFactory()`**：获取 `DefaultListableBeanFactory`，加载 Bean 定义（解析 XML 或扫描注解）。

**3. `prepareBeanFactory()`**：配置 BeanFactory，注册内置的 `BeanPostProcessor`（如 `ApplicationContextAwareProcessor`）和内置 Bean（如 `environment`）。

**4. `postProcessBeanFactory()`**：子类扩展点，Web 应用上下文在这里注册 Web 相关 Bean。

**5. `invokeBeanFactoryPostProcessors()`**：执行所有 `BeanFactoryPostProcessor`。**最重要的是 `ConfigurationClassPostProcessor`**，它在这里解析 `@Configuration`、`@ComponentScan`、`@Import`，把扫描到的类注册为 `BeanDefinition`。属性占位符替换也在这步完成。

**6. `registerBeanPostProcessors()`**：向 BeanFactory 注册所有 `BeanPostProcessor`（注意是注册，还没执行），比如 `AutowiredAnnotationBeanPostProcessor`（处理 `@Autowired`）和 `AnnotationAwareAspectJAutoProxyCreator`（AOP 代理创建）。

**7. `initMessageSource()`**：初始化国际化支持（`MessageSource`）。

**8. `initApplicationEventMulticaster()`**：初始化事件广播器（`ApplicationEventMulticaster`）。

**9. `onRefresh()`**：子类扩展点，Spring Boot 的 Web 服务器（Tomcat）就是在这里启动的。

**10. `registerListeners()`**：注册 `ApplicationListener`，发布早期事件。

**11. `finishBeanFactoryInitialization()`**：**实例化所有非懒加载的单例 Bean**，触发依赖注入和初始化回调。这步是启动最耗时的。

**12. `finishRefresh()`**：清理缓存，发布 `ContextRefreshedEvent`，启动生命周期 Bean。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/02、Spring容器与上下文#三、refresh() 的 12 个步骤]]

---

###### 4. 什么是 BeanDefinition？

`BeanDefinition` 是 Spring 里描述 Bean 的"蓝图"，包含了实例化这个 Bean 所需的所有元数据，但**不是 Bean 实例本身**。

类比一下：`BeanDefinition` 相当于建房子的设计图，Bean 实例才是建好的房子。

**BeanDefinition 包含的元数据：**

- **类信息**：Bean 的全限定类名
- **作用域**：singleton/prototype/request/session 等
- **懒加载**：是否延迟初始化
- **依赖关系**：构造器参数（`ConstructorArgumentValues`）和属性值（`PropertyValues`）
- **生命周期方法**：init-method 和 destroy-method 的名称
- **工厂信息**：如果是 `@Bean` 方法，记录工厂 Bean 名和工厂方法名

**常用实现类：**

- `GenericBeanDefinition`：通用场景
- `ScannedGenericBeanDefinition`：`@Component` 扫描结果，携带注解元数据
- `ConfigurationClassBeanDefinition`：`@Bean` 方法定义的 Bean

所有 `BeanDefinition` 注册到 `DefaultListableBeanFactory` 的 `beanDefinitionMap`（`ConcurrentHashMap`）里，以 beanName 为键。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/02、Spring容器与上下文#四、BeanDefinition 详解]]

---

###### 5. 什么是 BeanFactoryPostProcessor？

`BeanFactoryPostProcessor` 在**所有 Bean 定义加载完成后、Bean 实例化之前**执行，用于读取或修改 `BeanDefinition`。

**典型内置实现：**

- **`PropertySourcesPlaceholderConfigurer`**：把 Bean 定义里的 `${...}` 占位符替换为实际配置值
- **`ConfigurationClassPostProcessor`**：解析 `@Configuration`、`@ComponentScan`、`@Import`，这是 Spring 注解驱动的核心

**自定义示例：**

```java
@Component
public class CustomBFPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        BeanDefinition bd = beanFactory.getBeanDefinition("myService");
        bd.getPropertyValues().add("timeout", 5000);       // 修改属性值
        bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);       // 修改作用域
    }
}
```

**执行顺序**：实现 `PriorityOrdered` > `Ordered` > 普通接口的顺序执行。

**和 BeanPostProcessor 的区别**：`BeanFactoryPostProcessor` 操作的是 `BeanDefinition`（Bean 的蓝图），在实例化之前；`BeanPostProcessor` 操作的是已实例化的 Bean 对象，在初始化前后。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/02、Spring容器与上下文#五、BeanFactoryPostProcessor 与 BeanPostProcessor]]

---

###### 6. 什么是 BeanPostProcessor？

`BeanPostProcessor` 是 Spring 最重要的扩展点之一，在每个 Bean **实例化和依赖注入完成后、初始化回调前后** 执行，可以对 Bean 进行增强或替换。

**两个核心方法：**

- `postProcessBeforeInitialization()`：在 `@PostConstruct` / `afterPropertiesSet()` 之前调用
- `postProcessAfterInitialization()`：在初始化方法之后调用，**AOP 代理就是在这里创建的**

**Spring 内置的重要实现：**

- `AutowiredAnnotationBeanPostProcessor`：处理 `@Autowired`、`@Value` 注入（在依赖注入阶段，实际上比初始化更早）
- `CommonAnnotationBeanPostProcessor`：处理 `@PostConstruct`、`@PreDestroy`、`@Resource`
- `ApplicationContextAwareProcessor`：处理 `ApplicationContextAware` 等 Aware 接口回调
- `AnnotationAwareAspectJAutoProxyCreator`：在 `postProcessAfterInitialization` 里检查 Bean 是否匹配切入点，决定是否创建 AOP 代理

**自定义 BeanPostProcessor** 可以实现：性能监控（在初始化后包装代理）、自定义注解处理、Bean 替换等。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/02、Spring容器与上下文#五、BeanFactoryPostProcessor 与 BeanPostProcessor]]

---

###### 7. FactoryBean 和 BeanFactory 的区别是什么？

名字很像，但是两个完全不同的东西：

**`BeanFactory`**：Spring 容器的基础接口，是整个 IoC 容器的核心，负责管理所有 Bean 的创建、配置和生命周期。

**`FactoryBean`**：是一种特殊的 Bean，实现了这个接口的类本身由 Spring 管理，但它的职责是创建另一个复杂对象。调用 `getBean("myFactory")` 得到的是 `getObject()` 返回的对象，不是 `FactoryBean` 实例本身；要拿 `FactoryBean` 实例，需要加 `&` 前缀：`getBean("&myFactory")`。

```java
@Component("sqlSessionFactory")
public class SqlSessionFactoryBean implements FactoryBean<SqlSessionFactory> {
    
    @Override
    public SqlSessionFactory getObject() throws Exception {
        // 复杂的 MyBatis 初始化逻辑
        return buildSqlSessionFactory();
    }
    
    @Override
    public Class<?> getObjectType() {
        return SqlSessionFactory.class;
    }
}
```

这就是为什么你只需要配置 `SqlSessionFactoryBean`，容器里就自动有了 `SqlSessionFactory`—— `FactoryBean` 做了中间那层创建逻辑的封装。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/02、Spring容器与上下文#六、FactoryBean vs BeanFactory]]

---

###### 8. 如何在 Spring 容器中获取 Bean？

**最推荐：依赖注入**，让 Spring 自动把 Bean 注入进来，代码最干净：

```java
@Service
public class OrderService {
    @Autowired
    private UserRepository userRepository;
}
```

**编程式获取（动态场景）**：当需要在运行时根据条件动态决定用哪个 Bean 时：

```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);

MyService service1 = ctx.getBean(MyService.class);           // 按类型
MyService service2 = ctx.getBean("myService", MyService.class); // 按名称+类型
```

**实现 `ApplicationContextAware`**：让 Bean 能拿到 `ApplicationContext` 引用，之后用来动态获取其他 Bean（适合工具类/服务定位器场景）：

```java
@Component
public class SpringContextHolder implements ApplicationContextAware {
    private static ApplicationContext context;
    
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        context = ctx;
    }
    
    public static <T> T getBean(Class<T> type) {
        return context.getBean(type);
    }
}
```

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/02、Spring容器与上下文#七、ApplicationContextAware]]

---

###### 9. ApplicationContextAware 接口的作用是什么？

`ApplicationContextAware` 是一个回调接口，实现它的 Bean 在初始化过程中会被 Spring 自动调用 `setApplicationContext()` 方法，把当前 `ApplicationContext` 引用注入进来。

底层由 `ApplicationContextAwareProcessor`（`BeanPostProcessor` 实现）在 `postProcessBeforeInitialization()` 里处理。

**典型应用场景：**

1. **服务定位器**：需要在静态方法或非 Spring 管理的代码里获取 Bean
2. **事件发布**：Bean 内部需要发布自定义 Spring 事件
3. **读取配置**：通过 `applicationContext.getEnvironment()` 读取配置属性

```java
@Component
public class ServiceLocator implements ApplicationContextAware {
    private static ApplicationContext context;
    
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        ServiceLocator.context = applicationContext;
    }
    
    public static <T> T getBean(Class<T> beanType) {
        return context.getBean(beanType);
    }
}
```

**使用建议**：虽然很方便，但会使 Bean 与 Spring API 耦合，破坏可测试性。非必要情况优先用依赖注入，只有真正需要动态获取 Bean 的场景才用这个。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/02、Spring容器与上下文#七、ApplicationContextAware]]
