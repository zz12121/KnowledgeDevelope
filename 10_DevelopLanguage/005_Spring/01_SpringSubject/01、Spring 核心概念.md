###### 1. 什么是 Spring 框架？它的主要特点是什么？
Spring框架是一个**开源的Java/JavaEE应用框架**，由Rod Johnson在2003年创建，旨在简化企业级应用的开发。其核心价值在于通过**依赖注入（DI）**​ 和**面向切面编程（AOP）**​ 等机制，降低应用程序组件之间的耦合度，提高代码的可维护性和可测试性。
**主要特点包括：**
- **轻量级与非侵入性**：Spring框架本身体积小，对应用代码的侵入性低，应用对象可以不依赖Spring特定类存在。
- **控制反转（IoC）**：将对象的创建和依赖关系的管理交由Spring容器完成，而非由对象自身控制。
- **依赖注入（DI）**：作为IoC的实现方式，通过构造函数、Setter方法或字段注入依赖对象，实现组件间的松耦合。
- **面向切面编程（AOP）**：将横切关注点（如日志、事务管理）与业务逻辑分离，提高代码的模块化。
- **模块化设计**：Spring由多个模块（如Core、AOP、Data Access、Web等）组成，开发者可根据需求选择使用，增强灵活性。
- **简化数据访问与事务管理**：提供对JDBC、ORM框架的集成支持，以及声明式事务管理，简化数据库操作和事务处理。
- **丰富的生态集成**：与多种开源框架（如Hibernate、MyBatis）和技术（如消息服务、缓存）无缝集成。
Spring框架通过以上特性，帮助开发者构建可维护、可扩展且高效的企业级Java应用。
###### 2. 什么是控制反转（IoC）？
控制反转是一种**设计原则**，其核心思想是将对象的创建、依赖管理和生命周期的控制权从应用程序代码中反转给外部容器（如Spring IoC容器）。传统编程中，对象主动创建和查找依赖（如`new Service()`），导致代码紧耦合；而IoC模式下，对象被动接收依赖，由容器负责组装。
**IoC容器的核心作用包括：**
- **Bean的实例化与配置**：根据配置元数据（XML、注解或Java配置）创建并配置Bean实例。
- **依赖解析与注入**：自动处理Bean之间的依赖关系。
- **生命周期管理**：管理Bean从创建到销毁的整个生命周期，支持回调方法（如`@PostConstruct`）。
- **作用域管理**：支持Singleton（默认）、Prototype、Request、Session等作用域。
**源码角度**：Spring IoC容器的核心接口是`BeanFactory`，其默认实现类`DefaultListableBeanFactory`负责Bean的注册、依赖解析和生命周期管理。容器启动时，会解析配置信息，构建Bean定义（`BeanDefinition`），并通过反射机制实例化Bean。
###### 3. 什么是依赖注入（DI）？
依赖注入是**实现IoC的具体技术手段**，由容器在运行时将依赖对象注入到目标对象中，而非由目标对象主动创建或查找依赖。DI进一步降低了组件间的耦合度，提升了代码的可测试性和可维护性。
**Spring支持三种依赖注入方式：**
1. **构造函数注入**（推荐）：通过构造函数注入依赖，保证依赖不可变且完全初始化。
```java
    public class OrderService {
        private final PaymentService paymentService;
        // 容器自动注入PaymentService实例
        public OrderService(PaymentService paymentService) {
            this.paymentService = paymentService;
        }
    }
```
2. **Setter方法注入**：通过Setter方法注入可选依赖。
```java
    public class OrderService {
        private PaymentService paymentService;
        // 容器调用Setter方法注入
        public void setPaymentService(PaymentService paymentService) {
            this.paymentService = paymentService;
        }
    }
```
3. **字段注入**（通过`@Autowired`注解直接注入字段，需谨慎使用）：可能隐藏依赖关系，不利于测试。
**设计优势**：DI使代码更符合开闭原则，依赖可替换（如测试时注入Mock对象），且依赖关系集中管理。
###### 4. Spring 中的 IoC 容器有哪些类型？
Spring提供了两种类型的IoC容器，核心接口分别为 **`BeanFactory` 和 `ApplicationContext`**（后者是前者的子接口）。
- **BeanFactory**（`org.springframework.beans.factory.BeanFactory`）：
    - 是Spring最基础的IoC容器接口，提供基本的DI功能。
    - **特点**：采用**懒加载**策略，只有在请求获取Bean（如调用`getBean()`方法）时才实例化Bean。这节省了资源，但可能导致应用启动后第一次请求响应较慢。
    - **典型实现**：`XmlBeanFactory`（已过时，通过读取XML配置文件工作）。
- **ApplicationContext**（`org.springframework.context.ApplicationContext`）：
    - 是`BeanFactory`的扩展，提供了更多企业级功能。
    - **特点**：在**容器启动时就会预实例化所有单例Bean**。这虽然增加了启动时的资源开销，但能尽早发现配置错误，且所有Bean立即可用。
    - **常用实现**：
        - `ClassPathXmlApplicationContext`：从类路径加载XML配置文件。
        - `FileSystemXmlApplicationContext`：从文件系统路径加载XML配置文件。
        - `AnnotationConfigApplicationContext`：基于Java配置类（如`@Configuration`）加载配置。
**总结**：`ApplicationContext`功能更全面，是大多数Java应用的首选；`BeanFactory`在资源极度受限的场景下可能仍有价值。
###### 5. BeanFactory 和 ApplicationContext 的区别是什么？

|**特性**​|**BeanFactory**​|**ApplicationContext**​|
|---|---|---|
|**Bean加载时机**​|**懒加载**（Lazy Loading），使用时创建|**启动时预实例化**（Eager Loading）单例Bean|
|**企业级功能**​|仅提供基本DI功能|提供**完整的框架服务**，如国际化、事件发布、AOP集成等|
|**实现方式**​|基础接口，功能相对单一|`BeanFactory`的子接口，功能丰富|
|**资源消耗**​|较低|相对较高（因提前初始化Bean）|
|**适用场景**​|资源极度受限的轻量级应用|绝大多数企业级应用|
**源码层面**：`ApplicationContext`内部持有一个`BeanFactory`实例（通常为`DefaultListableBeanFactory`）作为其Bean定义注册和管理的核心委托对象。`ApplicationContext`在此基础上添加了其他服务层。
###### 6. Spring 框架的核心模块有哪些？
Spring框架采用模块化架构，主要核心模块包括：
- **Spring Core Container**：
    - **spring-core**：提供框架的基础组成部分，包括IoC和依赖注入功能。
    - **spring-beans**：提供`BeanFactory`的实现，负责Bean的创建和管理。
    - **spring-context**：建立在Core和Beans模块之上，提供应用上下文（`ApplicationContext`）等高级功能。
    - **spring-expression（SpEL）**：提供强大的表达式语言，用于在运行时查询和操作对象图。
- **AOP与Aspects**：
    - **spring-aop**：提供面向切面编程的实现，允许定义方法拦截器和切入点。
    - **spring-aspects**：提供与AspectJ框架的集成。
- **数据访问与集成**：
    - **spring-jdbc**：提供JDBC抽象层，简化JDBC操作。
    - **spring-orm**：提供对流行的对象关系映射API的集成，如JPA、Hibernate。
    - **spring-tx**：提供编程式和声明式事务管理。
- **Web模块**：
    - **spring-web**：提供面向Web的基本功能。
    - **spring-webmvc**：提供模型视图控制（MVC）和REST Web服务的实现。
- **测试模块**：**spring-test**：支持对具有JUnit或TestNG框架的Spring组件的测试。
###### 7. Spring 的优缺点是什么？
**优点：**
- **降低耦合度**：通过IoC/DI，使组件间依赖关系清晰，易于管理和替换。
- **提高可测试性**：依赖注入使得单元测试中可以轻松注入Mock对象。
- **简化开发**：提供大量模板类（如`JdbcTemplate`）和声明式服务（如事务管理），减少样板代码。
- **良好的模块化**：可根据需要引入特定模块，避免冗余。
- **强大的生态集成**：与众多开源项目无缝集成，生态系统丰富。
- **非侵入性设计**：应用代码通常不依赖于Spring特定类，保持了纯洁性。
**缺点：**
- **较高的学习曲线**：Spring框架功能丰富且抽象程度高，对初学者而言，掌握其所有特性和最佳实践需要一定时间。
- **配置可能变得复杂**：尤其是在大型项目中，XML配置（虽然现代Spring Boot已极大简化）或注解配置可能变得冗长复杂。
- **依赖性强**：项目一旦采用Spring，其一些高级功能会依赖于Spring特有的API和环境。
- **调试难度**：由于大量的动态代理和反射机制，在遇到复杂问题时，栈跟踪可能较深，增加调试难度。