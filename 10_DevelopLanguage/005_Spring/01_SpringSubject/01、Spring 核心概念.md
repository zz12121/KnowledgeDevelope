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
