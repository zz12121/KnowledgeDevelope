# IoC 控制反转与依赖注入

## 1 什么是 IoC

**IoC（控制反转，Inversion of Control）**是一种设计原则，核心思想是把对象的创建权和依赖管理权从对象自身"反转"给外部容器。

传统编程模式下，对象主动通过 `new` 创建自己的依赖，代码强耦合；IoC 模式下，对象只声明"我需要什么"，由容器负责组装注入，对象是被动接受者。

**Spring IoC 容器**实现这一原则的核心接口是 `BeanFactory`，默认实现类 `DefaultListableBeanFactory` 负责 Bean 的注册、依赖解析和生命周期管理。容器启动时解析配置信息，构建 `BeanDefinition` 对象（描述每个 Bean 的元数据），再通过反射机制实例化 Bean。

IoC 容器的核心职责：
- **Bean 实例化与配置**：根据配置元数据（XML / 注解 / Java Config）创建并配置 Bean
- **依赖解析与注入**：自动处理 Bean 之间的依赖关系
- **生命周期管理**：管理 Bean 从创建到销毁的整个过程，支持 `@PostConstruct` 等回调
- **作用域管理**：支持 Singleton、Prototype、Request、Session 等多种作用域

## 2 什么是 DI（依赖注入）

**DI（Dependency Injection，依赖注入）**是 IoC 的具体实现手段——由容器在运行时将依赖对象注入到目标对象，而非目标对象自行创建依赖。

Spring 支持三种注入方式：

### 构造函数注入（推荐）

依赖通过构造函数传入，保证依赖不可变、完全初始化，是 Spring 官方推荐的方式：

```java
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**优势**：依赖显式声明，不可为 null，天然支持不可变设计，也便于单元测试中直接 `new`。

### Setter 方法注入

通过 setter 方法注入可选依赖：

```java
public class OrderService {
    private PaymentService paymentService;

    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

适用于可选依赖，或者需要在对象构建后才能确定的依赖。

### 字段注入（@Autowired）

直接在字段上标注 `@Autowired`，写法最简洁，但会**隐藏依赖关系**，且不便于单元测试（无法通过构造函数传入 Mock）：

```java
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService;
}
```

**生产代码不推荐**，测试类中可以接受。

## 3 BeanFactory 和 ApplicationContext 的区别

两者都是 Spring IoC 容器，`ApplicationContext` 是 `BeanFactory` 的子接口，功能更完整。

**BeanFactory**：
- 最基础的 IoC 容器接口，只提供基本的 DI 功能
- **懒加载策略**：只有在 `getBean()` 时才实例化 Bean
- 资源占用低，但首次请求较慢，且无法提前发现配置错误

**ApplicationContext**：
- 容器**启动时就预实例化所有单例 Bean**，能尽早暴露配置问题
- 在 BeanFactory 基础上额外提供：国际化（i18n）、事件发布机制、AOP 集成、资源加载抽象等企业级功能
- 内部持有一个 `DefaultListableBeanFactory` 作为 Bean 管理委托

常用实现类：
- `ClassPathXmlApplicationContext`：从类路径加载 XML 配置
- `FileSystemXmlApplicationContext`：从文件系统路径加载 XML 配置
- `AnnotationConfigApplicationContext`：基于 `@Configuration` Java 配置类

**结论**：绝大多数场景用 `ApplicationContext`，`BeanFactory` 只在资源极度受限的嵌入式场景下才有价值。

源码层面：`ApplicationContext` 内部持有 `DefaultListableBeanFactory` 实例，Bean 的定义和创建工作都委托给它，`ApplicationContext` 自身负责附加的企业级功能层。

## 4 Spring 框架的核心模块

Spring 采用模块化架构，按需引入：

**Core Container（核心容器）**
- `spring-core`：框架基础，IoC 和 DI 功能的底层支撑
- `spring-beans`：`BeanFactory` 实现，Bean 的创建和管理
- `spring-context`：`ApplicationContext` 等高级容器功能
- `spring-expression`（SpEL）：运行时表达式语言，支持动态查询对象图

**AOP 与 Instrumentation**
- `spring-aop`：方法拦截器、切入点定义
- `spring-aspects`：与 AspectJ 集成

**数据访问与集成**
- `spring-jdbc`：JDBC 抽象层，`JdbcTemplate`
- `spring-orm`：JPA、Hibernate 等 ORM 集成
- `spring-tx`：编程式和声明式事务管理

**Web 模块**
- `spring-web`：Web 应用基础功能
- `spring-webmvc`：Spring MVC 和 REST 支持
- `spring-webflux`：响应式 Web 框架（5.0+）

**测试**
- `spring-test`：与 JUnit/TestNG 集成的测试支持

---

## 参考资料

> 本章节对应的原始参考资料和深入学习资源：

- **[IoC 容器源码](./E:/md/1/Spring/IoC/)** — Spring IoC 容器完整源码分析
- **[clazz 类加载与 Bean 解析](./E:/md/1/Spring/clazz/)** — BeanDefinition 解析与注册
- **[Spring 整体脉络](./E:/md/1/Spring/Spring整体脉络/)** — Spring 框架整体架构
- **[参考资料索引](../参考资料索引.md)** — 所有参考资料的总索引

**知识库双链**：
- 面试题库：[10_DevelopLanguage/005_Spring/01_SpringSubject/01、Spring 核心概念.md](../../10_DevelopLanguage/005_Spring/01_SpringSubject/01、Spring核心概念.md) — Spring 核心概念面试题

---

相关面试题 → [[../../10_DevelopLanguage/005_Spring/01_SpringSubject/01、Spring 核心概念|01、Spring核心概念]]
