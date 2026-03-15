###### 1. 什么是 Spring Bean？

Spring Bean 就是由 Spring IoC 容器**实例化、组装和管理的对象**，是 Spring 应用的基本构建块。

跟普通 Java 对象最大的区别是：普通对象你自己 `new`，自己管，自己销毁；Bean 的整个生命周期全交给容器，你只需要告诉容器"我要用这个类当 Bean"，容器负责创建、注入依赖、初始化、最后销毁。

容器内部用 `BeanDefinition` 来描述每个 Bean 的元数据，包括类名、作用域、依赖关系、初始化方法等。`BeanDefinitionReader` 负责读取配置（XML/注解/Java Config），注册到 `BeanDefinitionRegistry`，最后 `BeanFactory` 根据这些元数据创建实例。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#一、Bean 的定义与注册]]

---

###### 2. Spring 中 Bean 的作用域有哪些？

Spring 支持 6 种作用域，控制 Bean 实例的创建方式：

**singleton（默认）**：整个容器只有一个实例，所有获取该 Bean 的地方拿到的都是同一个对象。适合无状态的 Service、DAO 类。注意：singleton 不等于线程安全，多线程访问时要自己保证线程安全。

**prototype**：每次 `getBean()` 都创建一个新实例。适合有状态的对象，天然线程安全，但创建销毁频繁会增加内存开销，且容器不负责销毁 prototype Bean。

**request**：每次 HTTP 请求创建一个实例，请求结束自动销毁。仅 Web 应用中有效。

**session**：每个 HTTP 会话一个实例，会话结束销毁。适合存用户会话数据。

**application**：整个 Web 应用生命周期只有一个实例，类似 singleton，但绑定的是 `ServletContext`。

**websocket**：每个 WebSocket 会话一个实例。

配置方式：`@Scope("prototype")`，或在 XML 里 `scope="prototype"`。

**一个坑**：singleton Bean 注入 prototype Bean 时，拿到的 prototype 对象不会每次都刷新（因为 singleton 只初始化一次，依赖注入也只发生一次）。解决方式：让 singleton Bean 实现 `ApplicationContextAware`，每次使用时手动 `getBean()`；或者用 `@Lookup` 注解。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#三、Bean 的 6 种作用域]]

---

###### 3. Spring Bean 的生命周期是怎样的？

Bean 的完整生命周期分 9 个阶段，整个流程在 `AbstractAutowireCapableBeanFactory.doCreateBean()` 里实现：

**1. 实例化**：调用构造函数创建原始对象（还没设置任何属性）

**2. 属性填充**：通过 setter 或字段注入依赖关系，调用 `populateBean()` 方法

**3. Aware 接口回调**：如果 Bean 实现了 `BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware`，依次调用对应方法，让 Bean 感知自己的容器环境

**4. BeanPostProcessor 前置处理**：调用所有 `BeanPostProcessor.postProcessBeforeInitialization()`，这里 Spring 自己的很多功能也是在这里插入的

**5. 初始化**：依次执行 `@PostConstruct` 方法 → `InitializingBean.afterPropertiesSet()` → 自定义 `init-method`

**6. BeanPostProcessor 后置处理**：调用所有 `BeanPostProcessor.postProcessAfterInitialization()`，**AOP 代理就是在这里创建的**

**7. 使用阶段**：Bean 完全就绪，进入正常使用状态

**8. 销毁前**：容器关闭时，执行 `@PreDestroy` 方法

**9. 销毁**：执行 `DisposableBean.destroy()` → 自定义 `destroy-method`

注意：prototype 作用域的 Bean，容器只负责创建，不负责销毁。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#四、Bean 生命周期的 9 个阶段]]

---

###### 4. 如何在 Spring 中配置 Bean？

Spring 提供三种配置方式，现代项目通常混合使用：

**注解配置（现代首选）**：在类上加 `@Component`/`@Service`/`@Repository`/`@Controller`，配合 `@ComponentScan` 自动扫描注册：

```java
@Service
public class UserService {
    @Autowired
    private UserDao userDao;
}
```

**Java Config（显式控制，适合第三方库集成）**：用 `@Configuration` + `@Bean` 方法显式定义：

```java
@Configuration
public class AppConfig {
    @Bean
    public UserService userService() {
        return new UserService(userDao());
    }

    @Bean
    public UserDao userDao() {
        return new UserDaoImpl();
    }
}
```

**XML 配置（遗留项目）**：

```xml
<bean id="userService" class="com.example.UserService">
    <property name="userDao" ref="userDao"/>
</bean>
```

**选择建议**：业务代码用注解，第三方库或需要精细控制的 Bean 用 Java Config，尽量避免 XML（维护成本高）。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#二、三种 Bean 配置方式]]

---

###### 5. 什么是 Bean 的装配？有哪些装配方式？

Bean 装配就是**建立 Bean 之间协作关系的过程**，也就是依赖注入的实现。

**显式装配**：手动指定依赖，控制精确：

- 构造器注入：依赖通过构造函数传入，推荐，保证不可变
- Setter 注入：通过 Setter 方法设置，适合可选依赖

**隐式自动装配（Auto-wiring）**：容器根据规则自动匹配：

- `byType`：按类型找，最常用，找不到或找到多个会报错
- `byName`：按 Bean 名称找，要求属性名和 Bean 名一致
- `constructor`：构造器参数按类型匹配
- `no`：不自动装配，手动指定（XML 的默认值）

实际开发中，`@Autowired`（byType）+ `@Qualifier`（指定名称解歧义）是最常见的组合。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#五、@Autowired vs @Resource]]

---

###### 6. 什么是自动装配？自动装配有哪些方式？

自动装配是容器**自动建立 Bean 之间依赖关系**的机制，不需要手动写注入代码。

Spring 的自动装配主要通过 `@Autowired` 注解实现，核心逻辑是 `byType`：先按类型找，找到多个再按名称筛选（结合 `@Qualifier`）。

```java
@Component
public class UserService {
    // byType 自动装配
    @Autowired
    private UserDao userDao;

    // byType 找到多个时，@Qualifier 指定具体 Bean
    @Autowired
    @Qualifier("primaryUserDao")
    private UserDao primaryDao;
}
```

`@Autowired` 默认 `required = true`，找不到依赖直接报错。如果是可选依赖，可以设置 `@Autowired(required = false)` 或者用 `Optional<T>` 包装。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#五、@Autowired vs @Resource]]

---

###### 7. @Autowired 和 @Resource 的区别是什么？

这两个注解都是用来注入依赖的，但来源和默认行为不同：

**`@Autowired`** 是 Spring 自己的注解，默认按**类型（byType）**查找，找到多个再配合 `@Qualifier` 按名称筛选。支持 `required = false`。

**`@Resource`** 是 JSR-250 标准注解（Java 规范，不依赖 Spring），默认按**名称（byName）**查找，找不到再按类型退化。自带 `name` 属性可以直接指定 Bean 名称。

```java
@Component
public class ExampleService {
    @Autowired                          // byType，找 UserDao 类型的 Bean
    @Qualifier("mainDao")               // 找到多个时，指定名称
    private UserDao userDao;

    @Resource(name = "secondaryDao")    // 直接按名称 "secondaryDao" 找
    private UserDao anotherDao;
}
```

**如何选？** 纯 Spring 项目用 `@Autowired`，配合 `@Qualifier` 够用；需要跨框架可移植性（比如 Jakarta EE 项目）用 `@Resource`。日常开发两个都行，统一风格就好。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#五、@Autowired vs @Resource]]

---

###### 8. @Component、@Service、@Controller、@Repository 的区别是什么？

这四个注解的底层功能完全一样，都是用来标记"这个类要被 Spring 管理为 Bean"。区别在于语义层面，体现分层架构：

- **`@Component`**：通用组件，不属于特定层次，哪里都能用
- **`@Service`**：业务逻辑层，标记 Service 类
- **`@Controller`**：表现层，标记 MVC 的 Controller，Spring MVC 会识别它来处理请求映射
- **`@Repository`**：持久层，标记 DAO 类。有一个特殊能力：通过 `PersistenceExceptionTranslationPostProcessor`，会把底层的 JDBC/JPA/MyBatis 异常统一转换为 Spring 的 `DataAccessException` 体系

源码上，`@Service`、`@Controller`、`@Repository` 都是用 `@Component` 作为元注解，组件扫描时统一处理，没有本质区别。

**推荐按语义选**：代码可读性更好，团队看一眼注解就知道这个类的职责。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#六、@Component 系列注解的语义差异]]

---

###### 9. 如何解决循环依赖问题？

循环依赖是 A 依赖 B、B 又依赖 A（或更长的依赖环）。Spring 通过**三级缓存**解决了 singleton + setter/字段注入的循环依赖。

三级缓存的三层 Map：
- **一级缓存（singletonObjects）**：存放完全初始化好的单例 Bean
- **二级缓存（earlySingletonObjects）**：存放早期暴露的 Bean（已实例化但未完成初始化）
- **三级缓存（singletonFactories）**：存放 `ObjectFactory`，用于在需要时生成代理对象

**解决过程**：A 开始创建 → 实例化 A（半成品）→ 把 A 的 `ObjectFactory` 放入三级缓存 → 注入依赖发现需要 B → 开始创建 B → B 需要 A → 从三级缓存拿到 A 的工厂，生成早期 A 放入二级缓存 → B 完成初始化 → A 拿到 B，继续完成初始化 → A 进入一级缓存。

**但有限制**：只能解决 singleton Bean 的 setter/字段注入循环依赖。**构造器注入的循环依赖无法解决**（因为实例化阶段就需要依赖，还没机会放入缓存）。Spring Boot 2.6+ 默认禁止了循环依赖。

**规避方式：**
- 用 `@Lazy` 延迟加载其中一方
- 把公共逻辑提取到第三个 Bean 里（更推荐，从设计上消除循环）

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#七、三级缓存与循环依赖]]

---

###### 10. Spring 如何处理线程安全问题？

Spring 不保证 Bean 的线程安全，需要开发者根据 Bean 类型自己处理。

**singleton Bean 的线程安全**：singleton 是单实例，多个线程共享同一个对象。如果 Bean 里有可变的实例变量，就有线程安全问题。

解决思路有两种：**无状态设计**（最推荐）和 **ThreadLocal 隔离**：

```java
// 推荐：无状态 Service，所有数据通过方法参数传入，线程安全
@Service
public class StatelessService {
    public String process(String input) {
        return input.toUpperCase(); // 不依赖任何实例变量
    }
}

// 必须有状态时：ThreadLocal 保证线程隔离
@Service
public class ContextHolder {
    private final ThreadLocal<UserContext> contextHolder = new ThreadLocal<>();

    public void setContext(UserContext ctx) { contextHolder.set(ctx); }
    public UserContext getContext() { return contextHolder.get(); }
    public void clear() { contextHolder.remove(); } // 用完必须清理，防止内存泄漏
}
```

**prototype Bean**：每次请求创建新实例，天然线程隔离，但频繁创建销毁有内存压力。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#三、Bean 的 6 种作用域]]

---

###### 11. 什么是懒加载（Lazy Initialization）？

懒加载就是**延迟 Bean 的实例化时机**，容器启动时不创建，第一次被用到时才创建。

```java
@Component
@Lazy
public class HeavyService {
    // 启动时不会初始化，第一次注入或 getBean() 时才创建
}

// 注入懒加载 Bean 时，代理对象立即注入，但真正的 Bean 在第一次调用方法时才初始化
@Service
public class OrderService {
    @Autowired
    @Lazy
    private HeavyService heavyService;
}
```

全局开启懒加载：`spring.main.lazy-initialization=true`。

**适用场景**：初始化耗时的 Bean、不常用的功能模块、开发测试环境快速启动。

**副作用**：懒加载会把配置错误延迟到第一次使用时才暴露；首次请求响应时间会变长，生产环境可以配合应用预热解决。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解#八、@Lazy 懒加载]]

---

###### 12. 如何在 Spring 中注入集合类型？

Spring 支持直接注入 `List`、`Set`、`Map`、`Properties` 等集合类型。

```java
@Component
public class CollectionInjection {

    // 注入所有 Validator 类型的 Bean，按 @Order 排序
    @Autowired
    private List<Validator> validators;

    // 注入所有 Processor 类型的 Bean，key 是 Bean 名称
    @Autowired
    private Map<String, Processor> processors;

    // 用 SpEL 注入配置值列表
    @Value("#{'${server.ports:8080,8081}'.split(',')}")
    private List<Integer> ports;
}
```

当注入同类型的 Bean 集合时，Spring 会自动收集容器里所有匹配类型的 Bean。用 `@Order(1)` 或实现 `Ordered` 接口可以控制 List 里的顺序。

📖 [[../../../24_SpringKnowledge/04_Bean管理与容器/01、Bean管理详解]]
