# AOP 面向切面编程

## 1 什么是 AOP

**AOP（面向切面编程，Aspect-Oriented Programming）**是一种编程范式，旨在将**横切关注点**（如日志、事务、权限）与核心业务逻辑分离。

问题根源：这些横切逻辑（cross-cutting concerns）往往需要散布到很多业务方法中，导致代码纠缠（code tangling）——同一段日志代码出现在几十个方法里，既冗余又难以统一修改。AOP 把这些逻辑抽取成独立的**切面（Aspect）**，通过**织入（Weaving）**的方式动态应用到目标方法上，做到关注点分离。

## 2 AOP 核心术语

理解 AOP 必须先弄清楚这些概念：

**切面（Aspect）**：横切关注点的模块化封装。比如"日志切面"封装了所有日志相关逻辑，包含多种通知类型。

**连接点（Joinpoint）**：程序执行过程中可以被增强的位置。Spring AOP 中固定指**方法执行**这一类型（AspectJ 还支持字段访问、构造器等）。

**切入点（Pointcut）**：一个匹配连接点的规则表达式，用来筛选"要增强哪些方法"。比如 `execution(* com.example.service.*.*(..))` 匹配 service 包下所有方法。

**通知（Advice）**：在切入点匹配的连接点上要执行的具体动作，分为前置/后置/环绕等类型（详见下节）。

**织入（Weaving）**：将切面逻辑应用到目标对象、创建代理对象的过程。Spring AOP 在**运行时**完成织入，AspectJ 可在编译期或类加载期完成。

**目标对象（Target）**：被切面增强的原始业务对象。

**代理（Proxy）**：AOP 框架创建的增强对象，客户端实际调用的就是这个代理，它内部包含了切面逻辑和对目标方法的调用。

## 3 五种通知类型

Spring AOP 支持 5 种通知，定义了"在哪个时机执行增强逻辑"：

**@Before（前置通知）**：目标方法执行前触发，适用于参数校验、权限验证：

```java
@Before("execution(* com.example.service.*.*(..))")
public void logBefore(JoinPoint joinPoint) {
    System.out.println("准备执行: " + joinPoint.getSignature().getName());
}
```

**@AfterReturning（返回通知）**：目标方法**正常返回后**触发，可以拿到返回值：

```java
@AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
public void logAfterReturning(JoinPoint joinPoint, Object result) {
    System.out.println("返回值: " + result);
}
```

**@AfterThrowing（异常通知）**：目标方法**抛出异常后**触发，可以拿到异常对象：

```java
@AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))", throwing = "ex")
public void logAfterThrowing(JoinPoint joinPoint, Exception ex) {
    System.out.println("发生异常: " + ex.getMessage());
}
```

**@After（最终通知）**：无论正常返回还是抛出异常都会触发，类似 `finally` 块，常用于资源清理。

**@Around（环绕通知）**：功能最强，包裹整个方法执行，可以控制是否执行目标方法、修改参数和返回值：

```java
@Around("execution(* com.example.service.*.*(..))")
public Object timeAround(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();
    try {
        Object result = pjp.proceed();  // 手动调用目标方法
        System.out.println("耗时: " + (System.currentTimeMillis() - start) + "ms");
        return result;
    } catch (Exception e) {
        throw e;
    }
}
```

## 4 切入点表达式

`execution` 是最常用的指示符，语法：

```
execution([修饰符] 返回值类型 [包名.类名.]方法名(参数) [异常类型])
```

通配符：`*` 匹配单个任意字符；`..` 匹配多个包层级或任意参数；`+` 匹配指定类型及子类型。

常见示例：

```
execution(public * *(..))                        → 所有 public 方法
execution(* set*(..))                             → 所有 set 开头的方法
execution(* com.example.service.*.*(..))         → service 包下所有类所有方法
execution(* com.example.service..*.*(..))        → service 包及子包下所有方法
execution(* com.example.UserService.save(..))    → UserService 的 save 方法
@annotation(com.example.MyMonitor)               → 所有带 @MyMonitor 注解的方法
```

组合使用：`&&`（与）、`||`（或）、`!`（非）可以组合多个表达式。

## 5 如何实现自定义切面

标准四步：

```java
// 步骤一：定义切面类
@Aspect
@Component
public class PerformanceAspect {

    // 步骤二：定义可复用的切入点
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}

    @Pointcut("@annotation(com.example.Monitor)")
    public void monitoredMethod() {}

    // 步骤三：编写通知方法
    @Around("serviceLayer()")
    public Object timeMethod(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        log.info("{} 耗时 {}ms", pjp.getSignature().getName(),
                 System.currentTimeMillis() - start);
        return result;
    }
}

// 步骤四：开启 AOP 支持
@Configuration
@EnableAspectJAutoProxy
public class AppConfig { }
```

## 6 JDK 动态代理 vs CGLIB 代理

Spring AOP 底层基于动态代理，根据目标类是否实现接口选择代理策略：

**JDK 动态代理**：
- 要求目标类**必须实现至少一个接口**
- 代理类实现相同接口，本质是接口代理
- 创建速度快，但反射调用有一定开销
- 无法代理没有接口的类

**CGLIB 代理**：
- 通过**字节码生成子类**实现代理，目标类无需实现接口
- 无法代理 `final` 类或 `final` 方法（因为无法继承/覆写）
- 方法调用比 JDK 动态代理快，但生成代理对象时开销更大

**Spring 的默认策略**：目标类有接口 → JDK 动态代理；没有接口 → 强制使用 CGLIB。也可以通过配置强制全部使用 CGLIB：

```java
@EnableAspectJAutoProxy(proxyTargetClass = true)
```

Spring Boot 2.x 起，`proxyTargetClass` 默认为 `true`，统一使用 CGLIB。

## 7 Spring AOP 底层实现原理

核心流程：

1. **扫描阶段**：容器启动时，`InfrastructureAdvisorAutoProxyCreator`（一个 `BeanPostProcessor`）检查每个 Bean 是否匹配某个切面的切入点
2. **代理创建**：匹配到切面的 Bean，在初始化后阶段（`postProcessAfterInitialization`）被替换为代理对象（JDK 或 CGLIB）
3. **拦截器链**：代理对象内部持有一个**方法拦截器链**，包含了所有匹配当前方法的通知（`MethodInterceptor`）
4. **方法调用**：客户端调用代理方法 → 触发 `InvocationHandler.invoke()`（JDK）或 `MethodInterceptor.intercept()`（CGLIB）→ 构建 `ReflectiveMethodInvocation` → 按顺序遍历拦截器链执行通知 → 最终调用目标方法

关键类：`ProxyFactory`（创建代理）、`JdkDynamicAopProxy`（JDK 代理实现）、`CglibAopProxy`（CGLIB 代理实现）、`ReflectiveMethodInvocation`（拦截器链执行器）。

## 8 Spring AOP vs AspectJ

**Spring AOP**：
- 运行时织入，基于动态代理
- 只支持**方法级别**的连接点
- 与 Spring IoC 无缝集成，开箱即用
- 适合绝大多数企业应用场景

**AspectJ**：
- 支持编译期/类加载期/运行时三种织入方式
- 支持更丰富的连接点：字段访问、构造器调用、静态初始化块等
- 织入后是普通字节码，性能更高，无运行时代理开销
- 需要 `ajc` 编译器，配置相对复杂

**选择原则**：95% 的场景用 Spring AOP 就够了；需要拦截字段访问、构造器或追求极致性能时考虑 AspectJ。

## 9 AOP 常见应用场景

- **声明式事务**：`@Transactional` 是 AOP 最经典的应用，通知负责开启/提交/回滚事务
- **日志记录**：统一在方法入口/出口记录日志，避免散落的 `log.info()` 调用
- **权限控制**：在方法执行前验证用户权限
- **性能监控**：`@Around` 通知计算方法执行时间
- **异常统一处理**：捕获 Service 层异常，转为统一的错误码响应
- **缓存**：`@Cacheable` 等缓存注解底层也是 AOP 实现
- **数据验证**：方法参数的前置校验

---

相关面试题 → [[../../10_DevelopLanguage/005_Spring/01_SpringSubject/03、Spring AOP（面向切面编程）|03、Spring AOP]]
