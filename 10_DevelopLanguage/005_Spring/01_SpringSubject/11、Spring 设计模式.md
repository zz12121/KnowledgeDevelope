###### 1. Spring 中使用了哪些设计模式？

Spring 框架可以说是设计模式的教科书级实现，几乎把《设计模式》里的主流模式都用了一遍，而且用得都很经典。

主要有这几类：**工厂模式**（`BeanFactory`/`ApplicationContext`，负责对象创建和管理）、**单例模式**（容器级单例，`DefaultSingletonBeanRegistry` 的三级缓存）、**代理模式**（AOP 的底层支撑，JDK 动态代理和 CGLIB）、**模板方法模式**（`JdbcTemplate`/`RestTemplate`/`TransactionTemplate`，封装算法骨架）、**观察者模式**（Spring 事件驱动模型）、**适配器模式**（`HandlerAdapter` 统一处理不同类型的 Controller）、**装饰器模式**（`BeanWrapper`/`TransactionAwareCacheDecorator`，动态增强功能）、**策略模式**（`Resource` 资源加载策略、各种 `Resolver`）、**责任链模式**（拦截器链、Filter 链）。

这些模式并不是孤立存在的，它们相互协作，共同撑起了 Spring 高内聚、低耦合、易扩展的架构风格。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/01、Spring设计模式应用#Spring设计模式概览]]

---

###### 2. 工厂模式在 Spring 中的应用？

工厂模式在 Spring 中最典型的体现是 **`BeanFactory`** 和 **`ApplicationContext`** 这两个核心接口，它们构成了 IoC 容器的基础。

从工厂模式的角度看，Spring 用的是三个层次：

**简单工厂**：`BeanFactory` 根据 bean 名称或类型返回对应实例，把对象创建细节全部隐藏起来，调用方只管 `getBean()`。

**工厂方法**：`FactoryBean` 接口允许用户自定义复杂对象的创建逻辑，比如 `SqlSessionFactoryBean` 创建 MyBatis 的 `SqlSessionFactory`。Spring 容器从 `FactoryBean` 中拿到的是 `getObject()` 返回的对象，而不是 `FactoryBean` 本身：

```java
public interface FactoryBean<T> {
    T getObject() throws Exception;
    Class<?> getObjectType();
    default boolean isSingleton() { return true; }
}
```

**抽象工厂**：通过 `BeanDefinitionRegistry` 和 `BeanFactory` 的体系，支持创建关联的对象族。

工厂模式的核心价值是**集中化管理对象生命周期**，通过配置元数据（注解、Java Config）控制对象创建，把创建逻辑和使用逻辑彻底分开。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/01、Spring设计模式应用#工厂模式]]

---

###### 3. 单例模式在 Spring 中的应用？

Spring 通过**注册式单例模式**管理 Bean 实例，跟传统的双重检测锁单例不同，Spring 的单例是**容器级别的单例**——同一个容器里 bean 名称对应唯一实例，但不是 ClassLoader 级别的全局唯一。

核心实现在 `DefaultSingletonBeanRegistry` 的三级缓存里：

```java
// 一级缓存：完整的单例 Bean
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

// 二级缓存：早期暴露的 Bean（用于解决循环依赖，存的是半成品）
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

// 三级缓存：ObjectFactory（用于生成代理对象，支持 AOP 增强）
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

获取 Bean 时，依次从一级、二级、三级缓存查找，层层降级。如果三级缓存里有 `ObjectFactory`，调用它的 `getObject()` 生成对象（可能是 AOP 代理），然后提升到二级缓存。

三级缓存解决循环依赖的关键就在这里：A 在创建过程中把自己的 `ObjectFactory` 放到三级缓存，B 注入 A 时能从三级缓存拿到 A 的早期引用，避免了死循环。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/01、Spring设计模式应用#单例模式]]

---

###### 4. 代理模式在 Spring 中的应用？

代理模式是 Spring **AOP 的底层支撑**，Spring 通过动态代理在运行时增强目标对象的功能，不需要修改目标类的源码。

Spring 使用两种代理实现，自动选择：

**JDK 动态代理**：目标类实现了接口时使用，`JdkDynamicAopProxy` 实现 `InvocationHandler` 接口，通过反射调用目标方法，前后织入增强逻辑。创建快，但方法调用稍慢（反射开销）。

**CGLIB 代理**：目标类没有实现接口时使用，通过字节码技术生成目标类的子类，覆盖父类方法来实现拦截。创建慢，但方法调用快（直接方法调用）。

代理创建流程在 `AbstractAutoProxyCreator` 中：判断是否需要创建代理 → 收集匹配的 `Advisor`（增强器）→ 根据目标类选择 JDK 代理还是 CGLIB → 创建并返回代理对象。

代理模式的实际应用非常广泛：`@Transactional` 事务管理、`@Cacheable` 缓存、`@Async` 异步、`@Secured` 安全控制——这些注解背后全是代理在干活。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/01、Spring设计模式应用#代理模式]]

---

###### 5. 模板方法模式在 Spring 中的应用？

模板方法模式的核心思想是：**在父类中定义算法骨架，把可变的步骤留给子类或回调来实现**。Spring 大量使用这个模式来消除重复代码。

`JdbcTemplate` 是最经典的例子。操作数据库的流程是固定的（获取连接 → 创建 Statement → 执行 → 处理结果 → 释放资源），但具体 SQL 和结果处理逻辑是可变的。`JdbcTemplate` 把固定流程封装在模板方法里，可变部分通过回调接口（`StatementCallback`/`RowMapper` 等）让调用方填入：

```java
// 固定的算法骨架
public <T> T execute(StatementCallback<T> action) {
    Connection con = DataSourceUtils.getConnection(obtainDataSource());
    Statement stmt = null;
    try {
        stmt = con.createStatement();
        T result = action.doInStatement(stmt);  // 可变部分：回调
        return result;
    } catch (SQLException ex) {
        throw translateException(...);
    } finally {
        JdbcUtils.closeStatement(stmt);
        DataSourceUtils.releaseConnection(con, getDataSource());
    }
}
```

其他模板类同理：`RestTemplate` 封装 HTTP 请求流程、`TransactionTemplate` 封装事务管理流程、`JmsTemplate` 封装 JMS 消息操作。模板方法模式的价值在于：**保证了算法步骤的一致性**，同时提供了灵活的扩展点，让调用方只关注业务逻辑。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/01、Spring设计模式应用#模板方法模式]]

---

###### 6. 观察者模式在 Spring 中的应用？

Spring 通过**事件驱动模型**实现观察者模式，四个核心角色：事件（`ApplicationEvent`）、发布者（`ApplicationEventPublisher`）、监听器（`ApplicationListener`/`@EventListener`）、广播器（`ApplicationEventMulticaster`）。

当发布者调用 `publishEvent()` 时，广播器 `SimpleApplicationEventMulticaster` 遍历所有匹配的监听器并调用它们的处理方法：

```java
public void multicastEvent(final ApplicationEvent event) {
    for (final ApplicationListener<?> listener : getApplicationListeners(event)) {
        invokeListener(listener, event);  // 同步或异步，取决于是否配置了 TaskExecutor
    }
}
```

实际应用场景：订单创建后通知库存系统、用户注册后发送欢迎邮件、文件上传后触发处理任务——这些跨模块的通知场景，用事件机制比直接调用优雅得多，发布方和监听方完全解耦。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/01、Spring设计模式应用#观察者模式]]

---

###### 7. 适配器模式在 Spring 中的应用？

适配器模式解决的是**接口不兼容**的问题，让原本不能协作的类能够一起工作。Spring 里最典型的是 **`HandlerAdapter`**。

Spring MVC 支持多种类型的 Controller（实现 `Controller` 接口的、标注 `@RequestMapping` 的、实现 `HttpRequestHandler` 的），每种 Controller 的调用方式都不一样。`HandlerAdapter` 对每种类型的 Controller 提供适配器，统一暴露 `handle()` 接口，`DispatcherServlet` 只跟 `HandlerAdapter` 打交道，不需要关心底层 Controller 的类型：

```java
public interface HandlerAdapter {
    boolean supports(Object handler);
    ModelAndView handle(HttpServletRequest request, 
                       HttpServletResponse response, Object handler) throws Exception;
}
```

AOP 里也有适配器：`AdvisorAdapter` 将各种 `Advice`（`MethodBeforeAdvice`/`AfterReturningAdvice`）适配为统一的 `MethodInterceptor` 接口，让拦截器链能统一处理所有通知类型。

适配器模式让 Spring 能够**灵活支持多种实现策略**，新增一种 Controller 类型只需新增对应的适配器，不需要改核心逻辑。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/01、Spring设计模式应用#适配器模式]]

---

###### 8. 装饰器模式在 Spring 中的应用？

装饰器模式通过**包装原有对象来动态添加功能**，不修改原类，不影响调用方代码。Spring 里的典型应用是 `BeanWrapper`。

`BeanWrapper` 包装了 Bean 实例，在原始对象的基础上提供了更强的属性访问能力：支持属性路径（如 `address.city`）、自动类型转换、嵌套属性设置：

```java
BeanWrapperImpl wrapper = new BeanWrapperImpl(user);
wrapper.setPropertyValue("address.city", "北京");  // 嵌套属性
wrapper.setPropertyValue("age", "25");  // 字符串自动转 int
```

其他装饰器应用：
- `HttpServletRequestWrapper`：增强 `HttpServletRequest`，常用于参数加解密、日志记录
- `TransactionAwareCacheDecorator`：为缓存实例添加事务感知能力
- `DelegatingApplicationContext`：代理另一个 `ApplicationContext`

装饰器和代理模式看起来很像，区别在于：**装饰器关注功能增强**，被装饰者和装饰者通常实现同一接口，可以层层包装；**代理模式关注访问控制**，代理和真实对象实现同一接口，但代理控制对真实对象的访问，通常只有一层。

📖 [[../../../../24_SpringKnowledge/08_高级特性与性能/01、Spring设计模式应用#装饰器模式]]
