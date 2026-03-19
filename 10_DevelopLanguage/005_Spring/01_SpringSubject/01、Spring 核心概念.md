###### 1. 什么是 Spring 框架？它的主要特点是什么？

Spring 是一个开源的 Java 企业级应用框架，2003 年由 Rod Johnson 创建。它的核心价值是通过 **依赖注入（DI）** 和 **面向切面编程（AOP）** 降低代码之间的耦合，让代码更容易维护和测试。

说人话就是：以前你要用一个 Service，得自己 `new`，Service 又依赖 DAO，DAO 又依赖数据库连接……一串依赖全得手动管。Spring 帮你把这些都接管了，你只需要声明"我需要什么"，容器自动把依赖注进来。

**主要特点：**

- **控制反转（IoC）**：对象的创建和依赖管理交给 Spring 容器，不再由代码自己控制
- **依赖注入（DI）**：IoC 的具体实现手段，通过构造器、Setter 或字段把依赖注进来
- **AOP**：把日志、事务、权限这类横切关注点从业务逻辑里剥离出去，代码干净多了
- **轻量级非侵入**：应用代码通常不依赖 Spring 特定类，切换框架的成本低
- **模块化设计**：Core、AOP、Data Access、Web 等模块各司其职，按需引入
- **丰富的生态**：与 MyBatis、JPA、Redis、消息队列等主流技术无缝集成

**缺点也要了解：** 学习曲线比较陡，大量动态代理和反射让 debug 时调用栈很深，有时候排查问题比较麻烦。

📖 [[../../../24_SpringKnowledge/01_IoC与DI/01、IoC控制反转与依赖注入#一、IoC 的核心思想]]

---

###### 2. 什么是控制反转（IoC）？

IoC（Inversion of Control，控制反转）是一种设计原则，核心思想是：**把对象的创建和依赖管理的控制权，从代码里"反转"给外部容器**。

传统写法里，对象要用谁，就自己 `new` 谁，主动权在代码手里。IoC 反过来，对象被动等着容器把依赖"送上门"，自己不管也不知道依赖怎么来的。这就是所谓的"控制反转"。

**IoC 容器的核心职责：**

- 读取配置（XML/注解/Java Config），构建 `BeanDefinition`（Bean 的元数据）
- 根据 `BeanDefinition` 通过反射创建对象实例
- 自动解析并注入 Bean 之间的依赖关系
- 管理 Bean 从创建到销毁的完整生命周期

Spring IoC 的核心接口是 `BeanFactory`，最重要的实现类是 `DefaultListableBeanFactory`，负责 Bean 的注册、解析和管理。

📖 [[../../../24_SpringKnowledge/01_IoC与DI/01、IoC控制反转与依赖注入#二、IoC 容器的工作流程]]

---

###### 3. 什么是依赖注入（DI）？

DI（Dependency Injection，依赖注入）是 IoC 的具体实现手段。容器在运行时主动把依赖对象注入给目标对象，目标对象不需要知道依赖从哪来，也不需要自己创建。

**Spring 支持三种注入方式：**

**构造器注入（推荐）**：依赖通过构造函数传入，保证依赖不可变，创建时就完全初始化好了。

```java
@Service
public class OrderService {
    private final PaymentService paymentService; // final 保证不可变

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**Setter 注入**：通过 Setter 方法注入，适合可选依赖或需要在创建后重新设置的场景。

```java
public class OrderService {
    private PaymentService paymentService;

    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**字段注入**（通过 `@Autowired` 直接注入字段）：写起来最简洁，但会隐藏依赖关系，不利于测试，Spring 官方也不推荐用在生产代码里。

**为什么推荐构造器注入？** 依赖关系一目了然（看构造函数就知道），不可变设计更安全，写单元测试时直接 `new` 就能注入 Mock 对象，不需要 Spring 容器。

📖 [[../../../24_SpringKnowledge/01_IoC与DI/01、IoC控制反转与依赖注入#三、三种依赖注入方式]]

---

###### 4. Spring 中的 IoC 容器有哪些类型？

Spring 提供两种核心 IoC 容器，接口分别是 **`BeanFactory`** 和 **`ApplicationContext`**（后者是前者的子接口）。

**BeanFactory**：Spring 最基础的容器接口，提供基本的 DI 功能。采用**懒加载**策略，只有调用 `getBean()` 时才实例化 Bean。优点是启动快、资源消耗少，缺点是配置错误要到第一次使用时才能发现。已过时的 `XmlBeanFactory` 是早期实现。

**ApplicationContext**：`BeanFactory` 的扩展，提供更多企业级功能。容器启动时就**预实例化所有单例 Bean**，启动稍慢但能尽早暴露配置错误。常用实现：

- `ClassPathXmlApplicationContext`：从类路径加载 XML 配置
- `FileSystemXmlApplicationContext`：从文件系统加载 XML 配置
- `AnnotationConfigApplicationContext`：基于 `@Configuration` 注解类加载配置（最常用）
- `GenericWebApplicationContext`：Spring MVC 的 Web 应用上下文

**两者关系**：`ApplicationContext` 内部持有一个 `DefaultListableBeanFactory` 作为真正的 Bean 管理核心，自身在此之上提供了国际化、事件发布、AOP 集成等附加能力。

📖 [[../../../24_SpringKnowledge/01_IoC与DI/01、IoC控制反转与依赖注入#四、BeanFactory vs ApplicationContext]]

---

###### 5. BeanFactory 和 ApplicationContext 的区别是什么？

两者的核心区别主要体现在以下几个维度：

**Bean 加载时机**：`BeanFactory` 是懒加载，用到时才创建；`ApplicationContext` 是启动时就把所有单例 Bean 全部实例化好。所以 `ApplicationContext` 启动会慢一些，但运行时直接从缓存拿，更稳定。

**功能丰富度**：`BeanFactory` 只提供基础的 DI；`ApplicationContext` 还内置了国际化消息（`MessageSource`）、事件发布（`ApplicationEventPublisher`）、资源加载（`ResourcePatternResolver`）、AOP 自动代理等功能。

**适用场景**：日常开发几乎都用 `ApplicationContext`，资源极度受限（比如嵌入式设备、小型工具程序）才考虑 `BeanFactory`。

**源码层面**：`ApplicationContext` 内部有个 `DefaultListableBeanFactory`，实际的 Bean 定义存储和 Bean 实例化都委托给它，`ApplicationContext` 自己做的是包装和扩展。

📖 [[../../../24_SpringKnowledge/01_IoC与DI/01、IoC控制反转与依赖注入#四、BeanFactory vs ApplicationContext]]

---

###### 6. Spring 框架的核心模块有哪些？

Spring 是模块化设计，核心模块按功能分为几大类：

**核心容器（Core Container）**：
- `spring-core`：框架基础，提供 IoC 和 DI 核心功能
- `spring-beans`：`BeanFactory` 实现，Bean 创建与管理
- `spring-context`：`ApplicationContext`，支持国际化、事件等企业级功能
- `spring-expression`（SpEL）：强大的运行时表达式语言

**AOP 与切面**：
- `spring-aop`：基于代理的 AOP 实现
- `spring-aspects`：与 AspectJ 框架集成

**数据访问**：
- `spring-jdbc`：JDBC 抽象层，`JdbcTemplate` 在这里
- `spring-orm`：JPA、Hibernate 等 ORM 框架集成
- `spring-tx`：编程式和声明式事务管理

**Web**：
- `spring-web`：Web 基础功能，`DispatcherServlet` 的上层基础
- `spring-webmvc`：Spring MVC 完整实现
- `spring-webflux`：响应式 Web 框架（Spring 5+）

**测试**：
- `spring-test`：JUnit/TestNG 集成，支持上下文缓存、事务测试等

📖 [[../../../24_SpringKnowledge/01_IoC与DI/01、IoC控制反转与依赖注入#五、Spring 核心模块总览]]

---

###### 7. Spring 的优缺点是什么？

**优点：**

- **降低耦合**：IoC/DI 把依赖关系集中管理，组件之间解耦，替换某个实现非常方便
- **可测试性强**：构造器注入让单元测试可以直接注入 Mock，不需要启动容器
- **开发效率高**：`JdbcTemplate`、声明式事务、AOP 等减少了大量样板代码
- **生态极其丰富**：Spring 家族（Boot、Cloud、Security、Data 等）覆盖了企业开发的几乎所有场景
- **非侵入性**：业务代码通常不依赖 Spring 特定类，框架迁移成本低

**缺点：**

- **学习曲线陡**：概念多、抽象层次高，初学者需要花时间理解 IoC、AOP、代理机制等
- **配置复杂度**：大型项目里注解/Java Config 管理起来可能变得混乱
- **调试困难**：动态代理和反射导致调用栈很深，遇到奇怪问题时排查成本高
- **启动较慢**：Spring Boot 应用随着 Bean 增多，启动时间会明显变长

📖 [[../../../24_SpringKnowledge/01_IoC与DI/01、IoC控制反转与依赖注入]]

---

###### 7.1 你项目中是怎么使用Spring的？——高频面试引导问题

面试官问这个问题是想了解你是否有过**真实的Spring开发经验**，以及你对Spring生态的理解深度。

**常见使用场景**：

| 场景 | 使用方式 | 注意事项 |
|------|---------|---------|
| Bean管理 | @Component/@Service/@Repository/@Controller | 按层选择注解 |
| 依赖注入 | 构造器注入（推荐） | 避免循环依赖 |
| 事务管理 | @Transactional | 注意失效场景 |
| AOP | @Aspect | 理解JDK动态代理和CGLIB |
| 配置管理 | @Configuration + @Bean | 区分配置类和组件类 |

**回答示例**：

> 我们项目里用的是 Spring Boot，主要使用方式：
> 1. **Bean 管理**：日常开发用 `@Service` 和 `@Component` 管理 Bean，通过构造器注入依赖（不用字段注入，方便单元测试）
> 2. **配置管理**：用 `@ConfigurationProperties` 读取配置到 POJO，支持配置校验（`@Validated`），敏感配置用 `@Encrypted` 自己实现
> 3. **事务管理**：service 层方法加 `@Transactional`，传播行为用 `REQUIRED`，隔离级别根据业务选（读多写少用 READ_COMMITTED）
> 4. **异常处理**：用 `@ControllerAdvice` 统一处理异常，返回统一格式
> 5. **参数校验**：用 JSR-303 注解（`@NotNull`、`@Valid` 等），在 controller 层做第一道校验，service 层做业务校验
>
> 追问：为什么用构造器注入不用字段注入？
> - 构造器注入保证依赖不为空，编译期就能发现循环依赖
> - 字段注入容易让人忽略依赖关系，单元测试也不方便 mock
> - 当然，构造器注入依赖多了（比如超过5个）要考虑拆分服务

**技术选型建议**：
- 微服务 → Spring Cloud
- 快速开发 → Spring Boot
- 响应式编程 → Spring WebFlux
- 传统企业级开发 → Spring MVC + Spring

---

###### 7.2 Spring的IoC和DI是什么？——高频面试引导问题

**概念解释**：

| 概念 | 全称 | 说明 |
|------|------|------|
| IoC | Inversion of Control | 控制反转，把对象创建权交给Spring |
| DI | Dependency Injection | 依赖注入，Spring自动注入依赖对象 |

**回答示例**：

> IoC和DI其实是一回事：
> - IoC是思想：把对象创建权反转给容器
> - DI是实现：通过反射等方式注入依赖
>
> 追问：为什么需要IoC？
> - 降低耦合：对象间不直接依赖，通过容器注入
> - 方便测试：可以注入mock对象
> - 便于管理生命周期：单例、多例由容器控制

---

###### 7.3 Spring的Bean生命周期了解吗？——高频面试引导问题

**生命周期阶段**：

```
1. 实例化 → new对象
2. 属性赋值 → setXxx注入
3. 初始化 → 
   - BeanNameAware.setBeanName()
   - BeanFactoryAware.setBeanFactory()
   - @PostConstruct
   - InitializingBean.afterPropertiesSet()
   - 自定义init-method
4. 销毁 →
   - @PreDestroy
   - DisposableBean.destroy()
   - 自定义destroy-method
```

**回答示例**：

> 我们常用：
> - @PostConstruct：初始化后执行
> - @PreDestroy：销毁前执行
>
> 追问：为什么不用构造函数初始化？
> - 构造函数时依赖可能还没注入完成
> - @PostConstruct能确保所有依赖都注入完毕

---

###### 7.4 Spring的事务传播行为有哪些？——高频面试传播问题

**7种传播行为**：

| 行为 | 说明 | 常见场景 |
|------|------|---------|
| REQUIRED | 有事务加入，没有创建 | 默认 |
| REQUIRES_NEW | 总是创建新事务 | 日志 |
| SUPPORTS | 有事务加入，没有则非事务 | 查询 |
| NOT_SUPPORTED | 非事务执行 | 异步任务 |
| MANDATORY | 必须在事务中 | 核心业务 |
| NEVER | 必须不在事务中 | 非事务场景 |
| NESTED | 嵌套事务 | 批量操作 |

**回答示例**：

> 我们项目用法：
> - SERVICE层默认REQUIRED
> - 记录日志用REQUIRES_NEW（保证日志记录成功，即使业务回滚）
> - 查询用SUPPORTS（不需要事务）
>
> 追问：REQUIRED和REQUIRES_NEW区别？
> REQUIRED会加入外层事务，外层回滚内层也回滚；REQUIRES_NEW独立事务，外层回滚不影响内层。

---

###### 7.5 Spring的事务失效场景有哪些？——高频面试引导问题

**失效场景**：

| 场景 | 原因 | 解决方案 |
|------|------|---------|
| 非public方法 | AOP代理限制 | 改为public |
| 自调用 | 内部方法不经过代理 | 注入自身或AopContext |
| 异常被catch | 异常未抛出 | throw e |
| 非Spring管理的bean | 无法代理 | 配置component-scan |

**回答示例**：

> 我们项目遇到的坑：
> 1. **非public方法**：@Transactional只能加在public方法上
> 2. **自调用**：this.save()不会触发事务，因为没有经过代理
> 3. **异常被catch**：必须在finally throw 或直接 throw
>
> 追问：自调用怎么解决？
> - 注入自己：`@Autowired private UserService userService;`
> - 用AopContext：`enableAspectJProxy(exposeProxy=true)`

---

###### 7.6 Spring如何解决循环依赖？——高频面试引导问题

**循环依赖类型**：

| 类型 | 能否解决 | 方案 |
|------|---------|------|
| 构造器循环 | ❌ | 报错 |
| setter单例 | ✅ | 三级缓存 |
| prototype | ❌ | 报错 |

**三级缓存**：

```
singletonObjects（一级）→ 成品bean
earlySingletonObjects（二级）→ 提前暴露的bean（半成品）
singletonFactories（三级）→ lambda表达式，生成代理对象
```

**回答示例**：

> Spring解决setter循环依赖：
> 1. A创建时发现依赖B，把A的lambda放入三级缓存
> 2. B创建时发现依赖A，从三级缓存拿到A
> 3. B创建完成，放入一级缓存
> 4. A拿到B，完成创建
>
> 追问：为什么需要三级缓存？
> - 二级缓存是为了解决代理问题
> - 如果不需要代理，二级缓存就够了
> - 三级缓存主要为了性能，延迟代理对象的创建

---

###### 7.7 Spring的AOP原理是什么？——高频面试引导问题

**AOP核心概念**：

| 概念 | 说明 |
|------|------|
| Aspect | 切面（类） |
| Joinpoint | 连接点（方法） |
| Pointcut | 切入点（表达式） |
| Advice | 通知（增强逻辑） |
| Weaving | 织入 |

**通知类型**：

| 通知 | 执行时机 |
|------|---------|
| @Before | 方法前 |
| @After | 方法后 |
| @AfterReturning | 返回后 |
| @AfterThrowing | 异常后 |
| @Around | 环绕（前后） |

**回答示例**：

> 我们项目AOP用法：
> - 统一日志：记录请求参数和返回结果
> - 统一异常：@ControllerAdvice
> - 事务管理：@Transactional
> - 性能监控：记录方法执行时间
>
> 追问：JDK动态代理和CGLIB区别？
> - JDK：必须实现接口
> - CGLIB：继承类
> - Spring默认CGLIB（性能更好）

---

###### 7.8 Spring Boot的自动装配原理？——高频面试引导问题

**自动装配流程**：

```
1. @SpringBootApplication = @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
2. @EnableAutoConfiguration = @Import(AutoConfigurationImportSelector)
3. AutoConfigurationImportSelector读取META-INF/spring.factories
4. 加载所有AutoConfiguration类
5. 按条件@Conditional过滤
6. 注册Bean
```

**常见条件注解**：

| 注解 | 条件 |
|------|------|
| @ConditionalOnClass | class存在 |
| @ConditionalOnMissingBean | bean不存在 |
| @ConditionalOnProperty | 配置存在 |
| @ConditionalOnWebApplication | 是Web应用 |

**回答示例**：

> 自动装配原理：
> - Spring Boot启动时扫描classpath下的配置文件
> - 根据条件注解决定是否加载某些配置
> - 把符合条件的配置类注册为Bean
>
> 追问：如何自定义Starter？
> 1. 创建autoconfigure模块：写@Configuration + @Conditional
> 2. 创建starter模块：依赖autoconfigure + META-INF/spring.factories
> 3. 其他项目引入即可自动配置

---
