# Spring 中的设计模式

Spring 框架几乎就是设计模式教科书的活体实现。理解 Spring 里这些设计模式的使用，不仅能帮你更好地理解框架，还能在自己设计代码时有所借鉴。

---

## 一、工厂模式 —— 对象创建的核心

Spring 的 IoC 容器就是一个超级工厂，`BeanFactory` 和 `ApplicationContext` 是工厂模式最直接的体现。

**简单工厂**：`BeanFactory.getBean()` 根据 Bean 名称或类型返回实例，所有创建细节都被封装在容器里，调用方不需要关心如何 new 对象。

**工厂方法**：`FactoryBean` 接口允许你自定义复杂对象的创建逻辑。如果一个 Bean 的创建过程很复杂，直接用注解不方便，就可以实现 `FactoryBean`：

```java
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

注意：从容器里 `getBean("sqlSessionFactory")` 拿到的是 `getObject()` 返回的对象；如果要拿 `FactoryBean` 本身，需要加 `&` 前缀：`getBean("&sqlSessionFactory")`。

---

## 二、单例模式 —— 容器级别的单例

Spring 的单例不是传统的"一个 JVM 只有一个实例"，而是**容器级别的单例**。同一个 `ApplicationContext` 容器里，一个 Bean 默认只有一个实例。

底层实现是 `DefaultSingletonBeanRegistry`，用三层 Map 缓存实现（就是解决循环依赖那个三级缓存）：

```java
// 一级缓存：完整的单例 Bean
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

// 二级缓存：早期暴露的 Bean（未完全初始化）
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

// 三级缓存：ObjectFactory，用于生成代理对象
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

`getSingleton()` 方法依次从一、二、三级缓存中查找，获取单例时加了 `synchronized` 锁保证线程安全，但这个锁只在首次创建时生效，后续从一级缓存直接拿，性能没有影响。

---

## 三、代理模式 —— AOP 的底层基础

Spring AOP 的核心就是代理模式。无论是 `@Transactional`、`@Cacheable` 还是自定义切面，底层都是通过代理对象来拦截方法调用，织入额外逻辑。

Spring 根据目标类是否有接口来选择代理方式：

**JDK 动态代理**：目标类实现了接口时使用。核心类是 `JdkDynamicAopProxy`，实现了 `InvocationHandler`，在 `invoke()` 方法里执行增强链路：

```java
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler {
    
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 获取该方法的拦截器链
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
        
        if (chain.isEmpty()) {
            // 没有增强，直接执行目标方法
            return AopUtils.invokeJoinpointUsingReflection(target, method, args);
        } else {
            // 通过责任链依次执行通知
            MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
            return invocation.proceed();
        }
    }
}
```

**CGLIB 代理**：目标类没有接口，或者配置了 `proxyTargetClass = true`。CGLIB 通过字节码技术生成目标类的子类，在子类方法里织入增强逻辑。final 类和 final 方法无法被代理。

---

## 四、模板方法模式 —— 消除重复代码

模板方法模式定义了一个算法骨架，把可变的步骤留给子类实现。Spring 里大量运用这个模式来封装"固定流程 + 可变逻辑"的场景。

**`JdbcTemplate`** 是最典型的例子。数据库操作的固定步骤（获取连接、创建 Statement、处理异常、释放资源）都封装在模板里，可变的 SQL 执行逻辑通过回调传入：

```java
public class JdbcTemplate extends JdbcAccessor implements JdbcOperations {
    
    public <T> T execute(StatementCallback<T> action) throws DataAccessException {
        Connection con = DataSourceUtils.getConnection(obtainDataSource());
        Statement stmt = null;
        try {
            stmt = con.createStatement();
            T result = action.doInStatement(stmt); // 可变部分：你的 SQL 逻辑
            return result;
        } catch (SQLException ex) {
            JdbcUtils.closeStatement(stmt);
            DataSourceUtils.releaseConnection(con, getDataSource());
            throw translateException("StatementCallback", getSql(action), ex);
        } finally {
            // 固定的资源清理
        }
    }
}
```

类似的还有 `RestTemplate`（HTTP 请求）、`TransactionTemplate`（编程式事务）、`JmsTemplate`（消息队列）、`RedisTemplate`（Redis 操作）等。

`ApplicationContext.refresh()` 的 12 步初始化流程本身也是模板方法，留了 `onRefresh()` 等钩子给子类扩展。

---

## 五、观察者模式 —— 事件驱动解耦

Spring 的事件机制就是观察者模式：事件发布者不知道谁在监听，监听者也不直接依赖发布者，完全解耦。

四个核心角色：`ApplicationEvent`（事件）、`ApplicationEventPublisher`（发布者）、`ApplicationListener`（监听者）、`ApplicationEventMulticaster`（广播器）。

详细内容参考：[[01、Spring事件机制]]

---

## 六、适配器模式 —— 接口兼容性

适配器模式让不兼容的接口能协同工作，Spring MVC 里用得最多。

`DispatcherServlet` 需要统一处理各种 Controller，但 Controller 可能有很多种形式（实现 `Controller` 接口的、用注解的、实现 `HttpRequestHandler` 的）。`HandlerAdapter` 就是适配器，让不同类型的 Handler 都能以统一方式被调用：

```java
public interface HandlerAdapter {
    boolean supports(Object handler);  // 我能处理这种 handler 吗？
    ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler);
}
```

Spring AOP 里也有适配器：`AdvisorAdapter` 把各种 Advice（`MethodBeforeAdvice`、`AfterReturningAdvice`）适配成统一的 `MethodInterceptor` 接口，这样拦截器链的执行逻辑可以统一化处理。

---

## 七、装饰器模式 —— 透明增强

装饰器模式通过包装原对象来扩展功能，对外接口不变，但能力增强了。

`BeanWrapper` 是典型的装饰器，它包装了 Bean 实例，提供更强大的属性访问能力（类型转换、嵌套属性访问、集合操作等）：

```java
BeanWrapper wrapper = new BeanWrapperImpl(myBean);
wrapper.setPropertyValue("name", "张三");       // 自动类型转换
wrapper.setPropertyValue("address.city", "北京"); // 嵌套属性
```

其他例子：
- `HttpServletRequestWrapper`：增强 HttpServletRequest 功能
- `TransactionAwareCacheDecorator`：为缓存添加事务感知能力
- `SynchronizedCache`：为缓存加同步保护

**装饰器 vs 代理**：装饰器注重**功能增强**，调用方能感知；代理注重**访问控制**，通常对调用方透明。Spring AOP 更像代理，`BeanWrapper` 更像装饰器。

---

## 八、策略模式 —— 算法可替换

策略模式定义一族算法，让它们可以互相替换。

Spring 中典型的策略模式应用：

- **`CacheManager`**：不同缓存实现（Caffeine/Redis/Ehcache）可以互换，业务代码不用改
- **`ResourceLoader`**：支持 classpath/file/URL 等不同资源加载策略
- **`TransactionManager`**：`DataSourceTransactionManager`、`JpaTransactionManager` 可以按场景切换
- **`KeyGenerator`**：缓存键生成策略，默认是 `SimpleKeyGenerator`，可以自定义替换

---

## 九、责任链模式 —— 拦截器链

责任链让请求在多个处理者之间传递，每个处理者可以选择处理或传给下一个。

Spring AOP 的拦截器链（`MethodInterceptor` chain）就是责任链。多个切面的通知按顺序组成调用链，通过 `MethodInvocation.proceed()` 把控制权传递下去：

```
前置通知1 → 前置通知2 → 目标方法 → 后置通知2 → 后置通知1
```

Spring MVC 的 `HandlerInterceptor` 链也是同样的模式，`preHandle` 返回 `false` 就中断后续处理。

---

## 相关面试题 →

- [[../../10_Developlanguage/005_Spring/01_SpringSubject/11、Spring 设计模式]]
