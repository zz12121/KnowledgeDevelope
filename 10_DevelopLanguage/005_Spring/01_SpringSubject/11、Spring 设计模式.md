###### 1. Spring 中使用了哪些设计模式？
Spring框架广泛应用了多种设计模式，主要包括：**工厂模式**（BeanFactory、ApplicationContext）、**单例模式**（Bean作用域）、**代理模式**（AOP实现）、**模板方法模式**（JdbcTemplate等）、**观察者模式**（事件驱动模型）、**适配器模式**（HandlerAdapter等）、**装饰器模式**（BeanWrapper等）以及**策略模式**、**责任链模式**等。
这些模式并非孤立存在，而是相互协作，共同构筑了Spring框架高内聚、低耦合的特性。其核心价值在于：
- **解耦**：将对象创建、依赖管理与业务逻辑分离
- **复用**：通过模板化封装通用逻辑
- **扩展**：提供标准扩展点支持功能增强
- **管理**：统一管理对象生命周期和交互过程
###### 2. 工厂模式在 Spring 中的应用？
工厂模式在Spring中主要体现在**BeanFactory**和**ApplicationContext**这两个核心接口上，它们构成了Spring IoC容器的基础。
**源码实现分析：**
```java
// BeanFactory作为基础工厂接口
public interface BeanFactory {
    Object getBean(String name) throws BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException;
    // 其他工厂方法...
}

// ApplicationContext扩展了工厂功能
public interface ApplicationContext extends BeanFactory {
    // 扩展了消息、事件、环境等企业级功能
}
```
**具体应用场景：**
- **简单工厂模式**：`BeanFactory`根据bean名称或类型返回对应实例，隐藏了具体实现类的创建逻辑
- **工厂方法模式**：`FactoryBean`接口允许用户自定义复杂对象的创建逻辑
```java
public interface FactoryBean<T> {
    T getObject() throws Exception;  // 工厂方法
    Class<?> getObjectType();
    default boolean isSingleton() { return true; }
}
```
- **抽象工厂模式**：Spring通过`BeanDefinitionRegistry`和`BeanFactory`体系支持创建相关对象族
Spring的工厂模式优势在于**集中化管理对象生命周期**，通过配置元数据（XML、注解）控制对象创建，实现了创建逻辑与使用逻辑的彻底分离。
###### 3. 单例模式在 Spring 中的应用？
Spring通过**注册式单例模式**管理Bean实例，与传统的单例实现不同，Spring的单例是**容器级别的单例**而非ClassLoader级别的单例。
**源码级实现机制：**
```java
// DefaultSingletonBeanRegistry是单例注册表的核心实现
public class DefaultSingletonBeanRegistry {
    // 一级缓存：存储完整的单例Bean
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
    
    // 二级缓存：存储早期暴露的Bean（解决循环依赖）
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
    
    // 三级缓存：存储ObjectFactory（用于生成代理对象）
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
    
    public Object getSingleton(String beanName) {
        // 依次从一级、二级、三级缓存中查找Bean实例[10](@ref)
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            // 详细的三级缓存查询逻辑...
        }
        return singletonObject;
    }
}
```
**设计特点：**
- **双重检测锁机制**：在`getSingleton`方法中通过synchronized块确保线程安全
- **三级缓存解决循环依赖**：通过分层缓存策略解决Setter注入的循环依赖问题
- **灵活的单例控制**：虽然默认单例，但可通过`@Scope("prototype")`设置为原型模式
这种设计既保证了性能，又解决了传统单例模式难以处理的循环依赖等复杂场景。
###### 4. 代理模式在 Spring 中的应用？
代理模式是Spring **AOP（面向切面编程）的底层支撑**，Spring通过动态代理技术在运行时增强目标对象的功能。
**两种代理实现机制：**
**JDK动态代理**（基于接口）：
```java
// JdkDynamicAopProxy是实现JDK动态代理的核心类
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 拦截方法调用，执行增强逻辑
        MethodInvocation invocation = new ReflectiveMethodInvocation(
            proxy, target, method, args, targetClass, chain);
        return invocation.proceed();
    }
}
```
**CGLIB动态代理**（基于子类继承）：
- 当目标类没有实现接口时，Spring使用CGLIB生成子类代理
- 通过`MethodInterceptor`接口拦截方法调用并植入增强逻辑
**代理创建流程**（在`AbstractAutoProxyCreator`中）：
1. 判断是否应该为Bean创建代理
2. 收集匹配的Advisor（增强器）
3. 根据目标类特点选择JDK代理或CGLIB代理
4. 创建并返回代理对象
代理模式使得Spring能够**无侵入性地实现横切关注点**，如事务管理（`@Transactional`）、安全控制、日志记录等。
###### 5. 模板方法模式在 Spring 中的应用？
模板方法模式在Spring中广泛应用于**简化重复性操作**，定义算法骨架的同时允许子类重定义特定步骤。
**JdbcTemplate的模板模式实现：**
```java
public class JdbcTemplate extends JdbcAccessor implements JdbcOperations {
    public <T> T execute(StatementCallback<T> action) throws DataAccessException {
        // 定义算法骨架：获取连接、创建语句、执行回调、释放资源
        Connection con = DataSourceUtils.getConnection(obtainDataSource());
        Statement stmt = null;
        try {
            stmt = con.createStatement();
            applyStatementSettings(stmt);
            T result = action.doInStatement(stmt);  // 调用回调方法（可变部分）
            handleWarnings(stmt);
            return result;
        } catch (SQLException ex) {
            // 异常处理（固定部分）
            JdbcUtils.closeStatement(stmt);
            DataSourceUtils.releaseConnection(con, getDataSource());
            throw translateException("StatementCallback", getSql(action), ex);
        }
    }
}
```
**其他模板类应用：**
- `RestTemplate`：封装RESTful HTTP请求处理流程
- `TransactionTemplate`：提供声明式事务管理的模板
- `JmsTemplate`：简化JMS消息操作
模板方法模式的优势在于**去除重复代码**、**确保算法步骤一致性**、**提供标准扩展点**。
###### 6. 观察者模式在 Spring 中的应用？
Spring通过**事件驱动模型**实现观察者模式，实现ApplicationContext中Bean之间的松耦合通信。
**核心组件构成：**
- **事件源**：`ApplicationEventPublisher`（事件发布者）
- **观察者**：`ApplicationListener`（事件监听器）
- **事件对象**：`ApplicationEvent`（事件载体）
**源码实现细节：**
```java
// ApplicationEventMulticaster是事件广播的核心接口
public interface ApplicationEventMulticaster {
    void addApplicationListener(ApplicationListener<?> listener);
    void multicastEvent(ApplicationEvent event);
}

// SimpleApplicationEventMulticaster实现同步事件广播
public class SimpleApplicationEventMulticaster implements ApplicationEventMulticaster {
    public void multicastEvent(final ApplicationEvent event) {
        for (final ApplicationListener<?> listener : getApplicationListeners(event)) {
            // 遍历所有监听器并通知事件
            invokeListener(listener, event);
        }
    }
}
```
**使用示例：**
```java
// 自定义事件
public class OrderCreatedEvent extends ApplicationEvent {
    public OrderCreatedEvent(Order source) { super(source); }
}

// 事件监听器
@Component
public class OrderEventListener implements ApplicationListener<OrderCreatedEvent> {
    @Override
    public void onApplicationEvent(OrderCreatedEvent event) {
        // 处理订单创建事件
        System.out.println("处理订单: " + event.getSource());
    }
}

// 事件发布
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher publisher;
    
    public void createOrder(Order order) {
        // 业务逻辑...
        publisher.publishEvent(new OrderCreatedEvent(order));
    }
}
```
观察者模式使得Spring组件能够**解耦通信**，特别适合实现系统模块间的异步通知机制。
###### 7. 适配器模式在 Spring 中的应用？
适配器模式在Spring中主要用于**接口转换**，使不兼容的接口能够协同工作。
**Spring MVC中的HandlerAdapter：**
```java
public interface HandlerAdapter {
    boolean supports(Object handler);
    ModelAndView handle(HttpServletRequest request, 
                       HttpServletResponse response, Object handler) throws Exception;
}

// 多种适配器实现不同Controller类型
public class SimpleControllerHandlerAdapter implements HandlerAdapter {
    public boolean supports(Object handler) {
        return (handler instanceof Controller);
    }
    
    public ModelAndView handle(HttpServletRequest request, 
                              HttpServletResponse response, Object handler) throws Exception {
        return ((Controller) handler).handleRequest(request, response);
    }
}
```
**AOP中的适配器应用：**
Spring AOP通过`AdvisorAdapter`将各种Advice（通知）适配成统一的`MethodInterceptor`接口：
- `MethodBeforeAdviceAdapter`：将`MethodBeforeAdvice`适配成`MethodInterceptor`
- `AfterReturningAdviceAdapter`：将`AfterReturningAdvice`适配成`MethodInterceptor`
适配器模式的引入使得Spring能够**灵活支持多种实现策略**，保持框架的扩展性和兼容性。
###### 8. 装饰器模式在 Spring 中的应用？
装饰器模式在Spring中用于**动态增强对象功能**，通过包装原有对象添加额外职责。
**BeanWrapper的实现：**
`BeanWrapper`是Spring中典型的装饰器应用，它包装了Bean实例，提供更强的属性访问和能力：
```java
public class BeanWrapperImpl extends AbstractNestablePropertyAccessor implements BeanWrapper {
    private Object wrappedObject;  // 被包装的原始对象
    
    // 增强的属性访问方法
    public void setPropertyValue(String propertyName, Object value) {
        // 实现复杂的属性设置逻辑（类型转换、嵌套属性等）
    }
}
```
**其他装饰器应用：**
- `HttpServletRequestWrapper`：增强HttpServletRequest功能
- `TransactionAwareCacheDecorator`：为缓存添加事务支持
- 各种以`Wrapper`或`Decorator`结尾的类
装饰器模式与代理模式的区别在于：装饰器**关注于功能的增强**，而代理模式更注重**访问控制**。Spring通过装饰器模式能够透明地增强对象功能，而不影响客户端代码。