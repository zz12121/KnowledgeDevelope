###### 1. 什么是 AOP？
**AOP（面向切面编程）是一种编程范式，旨在将横切关注点（如日志记录、事务管理、权限控制等）与核心业务逻辑分离。AOP通过**预编译方式**或**运行期动态代理**实现在不修改源代码的情况下，为程序动态统一添加功能。
其核心价值在于**解决代码纠缠问题：即交叉业务逻辑（如事务、日志）与主业务逻辑混合在一起，导致代码冗余、可维护性降低。AOP将这些横切关注点模块化为**切面（Aspect）**，然后通过**织入（Weaving）过程将它们应用到目标对象的方法中，从而实现关注点分离，降低耦合度，提高代码复用性和开发效率。
###### 2. AOP 的核心概念有哪些（切面、连接点、切入点、通知、织入等）？
理解AOP，需要掌握其关键术语，它们描述了AOP如何工作。

|概念|描述|简单比喻|
|---|---|---|
|**切面 (Aspect)**​|对横切关注点（如事务管理）的模块化。一个切面可以包含多种通知类型。|就像一个通用的“功能模块”，例如“日志记录模块”。|
|**连接点 (Joinpoint)**​|程序执行过程中一个明确的点，如方法调用、异常抛出。在Spring AOP中，通常指**方法的执行**。|目标对象中所有可以被增强的方法。|
|**切入点 (Pointcut)**​|一个匹配连接点的谓词（表达式）。通知将与一个切入点表达式关联，并在匹配的连接点上运行。**切入点用于筛选出需要被增强的连接点**。|通过规则（如`execution(* com.example.service.*.*(..))`）从众多连接点中筛选出需要增强的特定方法。|
|**通知 (Advice)**​|切面在特定连接点上执行的动作。它定义了“何时”做什么。|在目标方法**之前**、**之后**或**前后**需要执行的逻辑。|
|**织入 (Weaving)**​|将切面应用到目标对象并创建新的代理对象的过程。Spring AOP在**运行时**完成织入。|将“日志记录模块”的代码应用到“用户服务”的方法上，形成一个全新的、具有日志功能的代理对象。|
|**目标对象 (Target)**​|被一个或多个切面所通知的对象。|原始的、不包含增强逻辑的业务逻辑对象。|
|**代理 (Proxy)**​|AOP框架创建的对象，它是切面逻辑与目标对象方法融合后的结果。客户端调用的实际上是这个代理对象。|一个伪装成目标对象的对象，它内部包含了增强逻辑和原始目标方法。|
###### 3. Spring AOP 和 AspectJ 的区别是什么？
虽然Spring AOP和AspectJ都实现了AOP思想，但它们在目标和实现方式上有显著不同。

|特性|Spring AOP|AspectJ|
|---|---|---|
|**目标**​|提供一种**简单、易用**的AOP实现，专注于解决最常见的企业级应用问题（如声明式事务）。|提供**完整**的AOP解决方案，功能更强大。|
|**织入时机**​|**运行时织入**。基于动态代理。|主要是在**编译期**（编译时织入）或**类加载期**（加载时织入）织入。|
|**性能**​|由于在运行时生成代理，性能有轻微开销。|代码直接被织入字节码，与普通Java代码无异，性能更高。|
|**功能范围**​|仅支持**方法级别**的连接点。|支持更丰富的连接点类型，如**字段访问、构造器调用、静态初始化块**等。|
|**依赖关系**​|轻量级，非强制依赖。可独立用于任何应用。|需要额外的编译器（ajc）或织入器，依赖AspectJ运行时库。|
|**集成度**​|与Spring IoC容器无缝集成。|可以独立使用，也可与Spring应用集成（Spring支持AspectJ切面）。|
**选择建议**：对于大多数Spring应用，如果只需要处理方法级别的横切关注点（如事务、日志、安全），**Spring AOP**因其简单性和与Spring生态的完美集成而绰绰有余。如果需要更细粒度的控制（如拦截对象构造、字段修改），或追求极致性能，则应选择**AspectJ**。
###### 4. Spring AOP 支持哪些类型的通知（Advice）？
通知定义了切面代码的执行时机。Spring AOP支持以下5种类型的通知：
1. **`@Before`（前置通知）**：在目标方法**执行之前**执行。适用于参数校验、权限验证等。
```java
    @Aspect
    @Component
    public class LoggingAspect {
        @Before("execution(* com.example.service.*.*(..))")
        public void logBefore(JoinPoint joinPoint) {
            System.out.println("准备执行方法: " + joinPoint.getSignature().getName());
        }
    }
```
2. **`@AfterReturning`（返回通知）**：在目标方法**成功执行完毕之后**执行。可以访问方法的返回值。
 ```java
    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        System.out.println("方法执行成功，返回值: " + result);
    }
```
3. **`@AfterThrowing`（异常通知）**：在目标方法**抛出异常后**执行。可以访问抛出的异常。
```java
    @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))", throwing = "ex")
    public void logAfterThrowing(JoinPoint joinPoint, Exception ex) {
        System.out.println("方法执行抛出异常: " + ex.getMessage());
    }
```
4. **`@After`（最终通知）**：在目标方法**执行结束后**执行，**不管是否成功还是异常**。类似于`try-catch-finally`中的`finally`块，常用于资源清理。
5. **`@Around`（环绕通知）**：**功能最强大**的通知。它包围了目标方法的整个执行过程，可以控制是否执行目标方法、何时执行、修改返回值等。
```java
    @Around("execution(* com.example.service.*.*(..))")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            // 执行目标方法，并可获取其返回值
            Object result = joinPoint.proceed();
            long elapsedTime = System.currentTimeMillis() - start;
            System.out.println("方法执行耗时: " + elapsedTime + "ms");
            return result;
        } catch (Exception e) {
            // 处理异常
            throw e;
        }
    }
```
###### 5. 如何实现一个自定义切面？
实现一个自定义切面通常包含以下步骤，这里以**注解方式**为例（现代Spring应用的首选）：
**第一步：定义切面类**
使用`@Aspect`注解标记一个类，并将其作为组件交由Spring管理（使用`@Component`）。
```java
@Aspect
@Component
public class MyCustomAspect {
    // 后续的切入点定义和通知方法将写在这里
}
```
**第二步：定义切入点**
使用`@Pointcut`注解定义一个可重用的切入点表达式。
```java
@Aspect
@Component
public class MyCustomAspect {
    // 定义一个切入点，匹配com.example.service包下所有类的所有方法
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}
    
    // 可以定义更精细的切入点，例如匹配所有带有特定注解的方法
    @Pointcut("@annotation(com.example.MyMonitorAnnotation)")
    public void monitoredMethod() {}
}
```
**第三步：编写通知方法**
将通知注解（`@Before`, `@Around`等）与切入点表达式绑定。
```java
@Aspect
@Component
public class MyCustomAspect {
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}
    
    // 使用定义好的切入点
    @Before("serviceLayer()")
    public void beforeServiceMethod(JoinPoint joinPoint) {
        // 前置通知逻辑
    }
    
    @Around("monitoredMethod()")
    public Object monitorMethodPerformance(ProceedingJoinPoint pjp) throws Throwable {
        // 环绕通知逻辑，例如性能监控
        return pjp.proceed();
    }
}
```
**第四步：启用AOP支持**
在配置类上添加`@EnableAspectJAutoProxy`注解，开启Spring对AspectJ注解的支持。
```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {
}
```
###### 6. 什么是代理模式？JDK 动态代理和 CGLIB 代理的区别是什么？
**代理模式**是一种设计模式，为其他对象提供一种代理以控制对这个对象的访问。代理对象在客户端和目标对象之间起到中介作用。
Spring AOP的底层正是基于**动态代理**技术。根据目标类是否实现接口，Spring会选择不同的代理策略：

|特性|JDK 动态代理|CGLIB 动态代理|
|---|---|---|
|**原理**​|基于**接口**。代理类会实现和目标类相同的接口。|基于**继承**。代理类是目标类的子类。|
|**使用条件**​|目标类**必须实现至少一个接口**。|目标类可以**没有实现任何接口**。|
|**性能**​|创建代理对象速度较快，但方法调用稍慢。|创建代理对象速度较慢，但方法调用较快。|
|**最终类限制**​|不适用此问题。|无法为声明为`final`的类或方法创建代理。|
|**Spring默认策略**​|如果目标类实现了接口，则默认使用。|如果目标类未实现接口，则强制使用。可通过配置强制使用CGLIB。|
**强制使用CGLIB代理的配置**：
```java
@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = true) // 设置为true表示强制使用CGLIB代理
public class AppConfig {
}
```
###### 7. Spring AOP 的底层实现原理是什么？
Spring AOP的底层实现可以概括为以下几个核心步骤：
1. **解析配置与创建代理对象**：当Spring IoC容器初始化一个Bean时，如果发现该Bean匹配了某个切入点表达式，容器就不会直接返回这个Bean的实例，而是会为其**创建一个代理对象**。
2. **选择代理机制**：根据上述规则（目标类是否有接口、`proxyTargetClass`设置）决定使用JDK动态代理还是CGLIB。
3. **织入通知链（核心）**：代理对象内部会包含一个**“拦截器链”（也叫通知链）**。这个链由与当前方法匹配的所有通知（`MethodInterceptor`）组成。
4. **执行过程（反射调用）**：当客户端调用代理对象的方法时，实际上会触发一个**反射调用**，被`InvocationHandler`（JDK代理）或`MethodInterceptor`（CGLIB代理）拦截。
**源码层面的简化流程（以JDK动态代理为例）**：
- Spring通过`ProxyFactory`类来创建AOP代理。
- 核心类`JdkDynamicAopProxy`实现了`InvocationHandler`接口。其`invoke`方法是拦截的核心。
- 在`invoke`方法中，Spring会为当前方法调用构建一个`ReflectiveMethodInvocation`对象，它封装了目标对象、方法、参数以及拦截器链。
- 然后调用`ReflectiveMethodInvocation`的`proceed()`方法，该方法会按顺序遍历并执行拦截器链中的每个通知，最后调用目标方法。
###### 8. 切入点表达式如何编写？
切入点表达式是AOP中用于定义**在哪些方法上执行通知**的关键。Spring使用AspectJ的切入点表达式语言。
**表达式结构**：
```
execution([修饰符] 返回值类型 [包名.类名.]方法名(参数) [异常类型])
```
其中，`[]`内的部分是可选的。
**通配符**：
- `*`：匹配任意字符（单个包、单个类名或方法名的一部分）。
- `..`：匹配任意多个字符（用于包名）或任意多个参数（用于参数列表）。
- `+`：匹配指定类型及其子类型（仅用于类型模式）。
**常见示例**：
- `execution(public * *(..))`：匹配所有公共方法。
- `execution(* set*(..))`：匹配所有以“set”开头的方法。
- `execution(* com.example.service.UserService.*(..))`：匹配`UserService`接口中定义的所有方法。
- `execution(* com.example.service.*.*(..))`：匹配`service`包下所有类的所有方法。
- `execution(* com.example.service..*.*(..))`：匹配`service`包及其所有子包下所有类的所有方法。
- `execution(* com.example.service.UserService.save(..))`：匹配`UserService`的`save`方法。
**其他指示符**：
除了`execution`，还有`within`（匹配类型）、`this`（匹配代理对象类型）、`target`（匹配目标对象类型）、`@annotation`（匹配带有指定注解的方法）等，可以组合使用以实现更精细的控制。
###### 9. AOP 有哪些应用场景？
AOP在实际项目中有着广泛的应用，以下是一些典型场景：
- **声明式事务管理**：这是Spring AOP最经典的应用。通过`@Transactional`注解，Spring在方法调用前开启事务，在方法成功后提交，在异常时回滚，业务代码无需关心事务API的细节。
- **日志记录**：在方法执行前后、异常时统一记录日志，避免在业务代码中散落大量的`log.info()`语句。
- **权限控制和安全性检查**：在方法执行前，通过切面判断当前用户是否具有执行该操作的权限。
- **性能监控和统计**：使用`@Around`通知来计算方法执行时间，用于性能分析和优化。
- **异常处理和统一响应封装**：捕获服务层抛出的异常，将其转换为统一的错误码和消息格式返回给前端。
- **缓存**：在方法执行前检查缓存，如果存在则直接返回；方法执行后将结果存入缓存。
- **数据验证**：在方法执行前对参数进行校验。