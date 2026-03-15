# Bean 管理详解

## 1 什么是 Spring Bean

**Spring Bean** 是由 Spring IoC 容器**实例化、组装和管理的对象**，是 Spring 应用的基本构建块。与普通 Java 对象的区别在于，Bean 的生命周期完全由容器控制。

Spring 通过 `BeanDefinition` 接口描述 Bean 的元数据（类名、作用域、属性值、初始化方法等）。容器启动时，`BeanDefinitionReader` 读取配置并注册到 `BeanDefinitionRegistry`，再由 `BeanFactory` 根据这些定义创建 Bean 实例。

```java
// 普通对象：直接 new，完全由程序控制
User user = new User();

// Spring Bean：由容器管理，通过 ApplicationContext 获取
ApplicationContext ctx = new ClassPathXmlApplicationContext("config.xml");
User user = ctx.getBean("user", User.class);
```

## 2 Bean 的作用域

Spring 支持 6 种 Bean 作用域：

**singleton（默认）**：整个容器中只有一个实例。适用于无状态的 Service、DAO 等。注意线程安全问题，singleton Bean 应尽量设计为无状态的。

**prototype**：每次 `getBean()` 都创建新实例，天然线程安全，但容器不负责销毁，内存开销相对大。适用于有状态、需要隔离的 Bean。

**request**：每次 HTTP 请求创建一个新实例，请求结束销毁。仅 Web 应用可用，适合存储请求级数据。

**session**：每个 HTTP 会话一个实例。适合存储用户会话数据。

**application**：整个 Web 应用生命周期中一个实例，类似全局 singleton，但绑定到 `ServletContext`。

**websocket**：每个 WebSocket 会话一个实例。

作用域通过 `@Scope` 注解配置：

```java
@Component
@Scope("prototype")
public class OrderContext { }
```

底层机制：`SingletonScope` 用 `ConcurrentHashMap` 缓存实例；`PrototypeScope` 每次调用 `getBean()` 都通过反射创建新对象。

## 3 Bean 的生命周期

Spring Bean 生命周期是面试高频考点，完整流程如下：

1. **实例化**：容器调用构造函数（或工厂方法）创建 Bean 实例
2. **属性填充**：通过 setter 或字段注入依赖关系
3. **Aware 接口回调**：
   - `BeanNameAware.setBeanName()`
   - `BeanFactoryAware.setBeanFactory()`
   - `ApplicationContextAware.setApplicationContext()`
4. **BeanPostProcessor 前置处理**：`postProcessBeforeInitialization()`
5. **初始化**（三种方式，按顺序执行）：
   - `@PostConstruct` 注解方法
   - `InitializingBean.afterPropertiesSet()`
   - 自定义 `init-method`
6. **BeanPostProcessor 后置处理**：`postProcessAfterInitialization()`（AOP 代理就在这里创建）
7. **Bean 进入使用阶段**
8. **销毁前处理**：`@PreDestroy` 注解方法
9. **销毁**：`DisposableBean.destroy()` 和自定义 `destroy-method`

源码实现：`AbstractAutowireCapableBeanFactory.doCreateBean()` 方法完整实现了这一流程；`initializeBean()` 处理初始化回调；`destroyBean()` 处理销毁逻辑。

## 4 Bean 的三种配置方式

### XML 配置（传统方式）

```xml
<bean id="userService" class="com.example.UserService">
    <property name="userDao" ref="userDao"/>
    <property name="timeout" value="5000"/>
</bean>
```

### 注解配置（现代首选）

```java
@Component
public class UserService {
    @Autowired
    private UserDao userDao;
}
```

### Java 配置类（显式控制）

适合第三方库或需要精细控制的场景：

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

**选择建议**：现代 Spring 项目以注解配置为主，第三方库集成用 Java 配置类，遗留系统维护 XML 配置。

## 5 自动装配：@Autowired 和 @Resource 的区别

这两个注解都能实现依赖注入，但来源和默认行为不同：

**@Autowired**（Spring 注解）：
- 默认按**类型（byType）**匹配
- 类型匹配到多个 Bean 时，配合 `@Qualifier` 指定名称
- 支持 `required = false`，允许注入 null
- 纯 Spring 环境的标准选择

```java
@Autowired
@Qualifier("mainDao")
private UserDao userDao;
```

**@Resource**（JSR-250 标准注解）：
- 默认按**名称（byName）**匹配，找不到再按类型
- 自带 `name` 属性直接指定 Bean 名称
- 标准 Java 注解，不依赖 Spring

```java
@Resource(name = "secondaryDao")
private UserDao anotherDao;
```

**实践建议**：Spring 项目中优先用 `@Autowired`；需要跨框架兼容时选 `@Resource`。

## 6 @Component 系列注解的区别

`@Component`、`@Service`、`@Controller`、`@Repository` 四个注解**本质功能相同**，都是组件扫描标记，区别在于语义层面：

- `@Component`：通用组件，没有具体语义
- `@Service`：标记业务逻辑层，体现"服务"语义
- `@Controller`：标记 MVC 控制层，会被 Spring MVC 识别
- `@Repository`：标记数据访问层，额外功能是通过 `PersistenceExceptionTranslationPostProcessor` 将数据库原生异常转换为 Spring 统一的 `DataAccessException` 体系

源码角度：这四个注解都被 `@Component` 元注解标注，组件扫描时统一处理，没有功能差异。

## 7 循环依赖与三级缓存

**循环依赖**指 Bean 之间相互依赖形成闭环，如 A 依赖 B，B 又依赖 A。

**Spring 的三级缓存解决方案**：

```
一级缓存 singletonObjects：    完整的、初始化完毕的 Bean 实例
二级缓存 earlySingletonObjects：提前暴露的"半成品" Bean（已实例化但未完成属性填充）
三级缓存 singletonFactories：  Bean 的 ObjectFactory，用于在需要时生成代理对象
```

**解决流程**：A 开始实例化 → 将 A 的 ObjectFactory 放入三级缓存 → A 填充属性，发现需要 B → 创建 B → B 填充属性，发现需要 A → 从三级缓存取出 A 的 ObjectFactory，生成 A 的早期引用放入二级缓存 → B 完成初始化放入一级缓存 → A 继续完成初始化，最终放入一级缓存。

**三级缓存的意义**：第三级的 ObjectFactory 保证了即使 A 需要 AOP 代理，也能在 B 注入时拿到的是 A 的代理对象而非原始对象，保证了一致性。

**适用条件**（同时满足才能解决）：
- singleton 作用域
- setter 注入或字段注入（**构造器注入无法解决**，因为构造器注入时对象还没创建）

**规避策略**：
- 使用 `@Lazy` 延迟加载其中一方
- 将公共逻辑提取到第三个 Bean
- 重新设计依赖关系，消除循环

## 8 懒加载 @Lazy

懒加载（Lazy Initialization）让 Bean 延迟到**第一次被请求时**才实例化，而非容器启动时。

```java
@Component
@Lazy
public class HeavyService {
    // 第一次被注入时才会实例化
}

@Configuration
public class AppConfig {
    @Bean
    @Lazy
    public ExpensiveBean expensiveBean() {
        return new ExpensiveBean();
    }
}
```

**适用场景**：启动成本高的 Bean、不常用的功能模块、加速测试环境启动。

**注意**：懒加载会推迟配置错误的暴露时机——原本启动时就能发现的问题，被延迟到第一次使用时才报错。生产环境需权衡。

---

相关面试题 → [[../../10_DevelopLanguage/005_Spring/01_SpringSubject/02、Spring Bean 管理|02、Spring Bean管理]]
