###### 1. 什么是 AOP？

AOP（面向切面编程）是一种编程范式，目的是把**横切关注点**（日志、事务、权限、性能监控等）从业务逻辑里抽出来，单独管理，不再散落在每个方法里。

举个例子：没有 AOP 时，你要在 100 个方法里加日志，就得改 100 个地方。有了 AOP，写一个切面统一处理，100 个方法一行都不用动。

Spring AOP 的实现原理是**运行时动态代理**：在应用运行过程中，为目标对象生成代理对象，通过代理对象拦截方法调用，在调用前后插入横切逻辑。代码织入发生在运行期，不需要修改源代码，也不需要特殊的编译工具。

📖 [[../../../24_SpringKnowledge/02_AOP面向切面/01、AOP面向切面编程#一、AOP 是什么]]

---

###### 2. AOP 的核心概念有哪些（切面、连接点、切入点、通知、织入等）？

AOP 有几个关键术语，理解了这些概念才能看懂切面代码：

**切面（Aspect）**：横切关注点的模块化封装，比如"日志切面"、"事务切面"。一个切面包含切入点 + 通知的完整定义。

**连接点（Joinpoint）**：程序执行过程中的一个明确节点，在 Spring AOP 里，连接点特指**方法的执行**（Spring AOP 只支持方法级别）。

**切入点（Pointcut）**：用表达式描述"匹配哪些连接点"，决定通知在哪些方法上生效。比如 `execution(* com.example.service.*.*(..))` 表示匹配 service 包下所有方法。

**通知（Advice）**：切面在连接点执行的具体动作，定义"什么时候做什么"。分为前置/返回/异常/最终/环绕五种。

**织入（Weaving）**：把切面应用到目标对象、创建代理对象的过程。Spring AOP 在**运行时**织入。

**目标对象（Target）**：被切面增强的原始业务对象。

**代理（Proxy）**：AOP 创建的代理对象，包含了原始业务逻辑 + 切面逻辑的融合体。客户端实际调用的是这个代理。

📖 [[../../../24_SpringKnowledge/02_AOP面向切面/01、AOP面向切面编程#二、七个核心术语]]

---

###### 3. Spring AOP 和 AspectJ 的区别是什么？

两者都实现了 AOP，但定位不同：

**Spring AOP**：轻量级 AOP 实现，基于运行时动态代理，只支持方法级别的拦截。跟 Spring 容器深度集成，使用简单，覆盖日常 90% 的需求。

**AspectJ**：完整的 AOP 框架，通过编译期或类加载期把切面代码织入字节码，性能更高，而且支持字段访问、构造器调用、静态初始化块等更细粒度的拦截点。需要 ajc 编译器或 Java Agent。

**关键对比：**

- 织入时机：Spring AOP 运行时，AspectJ 编译期/加载期
- 支持的连接点：Spring AOP 只有方法执行，AspectJ 支持字段/构造器/异常处理块等
- 性能：AspectJ 编译后直接是字节码，没有代理开销；Spring AOP 有代理层调用开销
- 上手难度：Spring AOP 更简单，AspectJ 需要额外学习 ajc 工具链

**Spring 其实支持使用 AspectJ 注解语法**（`@Aspect`、`@Pointcut` 等），但底层织入机制还是 Spring 自己的代理方式，两套注解语法可以混用。

**选择建议**：日常方法级别的事务、日志、权限用 Spring AOP 完全够用。需要拦截字段访问或追求极致性能才考虑 AspectJ。

📖 [[../../../24_SpringKnowledge/02_AOP面向切面/01、AOP面向切面编程#八、Spring AOP vs AspectJ]]

---

###### 4. Spring AOP 支持哪些类型的通知（Advice）？

Spring AOP 支持 5 种通知类型：

**@Before（前置通知）**：目标方法执行前触发，常用于参数校验、权限验证、日志记录。

```java
@Before("execution(* com.example.service.*.*(..))")
public void logBefore(JoinPoint joinPoint) {
    log.info("准备执行方法: {}", joinPoint.getSignature().getName());
}
```

**@AfterReturning（返回通知）**：目标方法正常返回后触发，可以拿到返回值。

```java
@AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
public void logAfterReturning(JoinPoint joinPoint, Object result) {
    log.info("方法执行成功，返回值: {}", result);
}
```

**@AfterThrowing（异常通知）**：目标方法抛出异常后触发，可以拿到异常对象。

```java
@AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))", throwing = "ex")
public void logAfterThrowing(JoinPoint joinPoint, Exception ex) {
    log.error("方法 {} 抛出异常: {}", joinPoint.getSignature().getName(), ex.getMessage());
}
```

**@After（最终通知）**：无论方法正常返回还是抛异常，都会触发，类似 `finally` 块，常用于资源清理。

**@Around（环绕通知）**：最强大的通知，完全控制目标方法的执行，可以决定是否执行、修改参数、修改返回值。必须手动调用 `pjp.proceed()` 才会执行目标方法。

```java
@Around("execution(* com.example.service.*.*(..))")
public Object timeAround(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();
    try {
        Object result = pjp.proceed(); // 执行目标方法
        log.info("方法耗时: {}ms", System.currentTimeMillis() - start);
        return result;
    } catch (Exception e) {
        log.error("方法执行异常", e);
        throw e;
    }
}
```

**执行顺序**（正常情况）：`@Around（前）→ @Before → 目标方法 → @Around（后）→ @AfterReturning → @After`

📖 [[../../../24_SpringKnowledge/02_AOP面向切面/01、AOP面向切面编程#四、五种通知类型详解]]

---

###### 5. 如何实现一个自定义切面？

四步搞定：

**第一步：定义切面类**，加 `@Aspect` + `@Component`：

```java
@Aspect
@Component
public class LoggingAspect {
    // 切入点和通知方法写在这里
}
```

**第二步：定义切入点**，用 `@Pointcut` 声明可复用的切入点表达式：

```java
// 匹配 service 包下所有类的所有方法
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceLayer() {}

// 匹配带有自定义注解的方法
@Pointcut("@annotation(com.example.annotation.Monitor)")
public void monitoredMethod() {}
```

**第三步：编写通知方法**，绑定到切入点：

```java
@Before("serviceLayer()")
public void beforeLog(JoinPoint jp) {
    log.info("调用方法: {}", jp.getSignature().getName());
}

@Around("monitoredMethod()")
public Object monitorPerformance(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();
    Object result = pjp.proceed();
    log.info("{} 耗时 {}ms", pjp.getSignature().toShortString(), System.currentTimeMillis() - start);
    return result;
}
```

**第四步：开启 AOP 支持**（Spring Boot 项目通常不需要手动加，自动配置会处理）：

```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig { }
```

📖 [[../../../24_SpringKnowledge/02_AOP面向切面/01、AOP面向切面编程#五、自定义切面四步骤]]

---

###### 6. 什么是代理模式？JDK 动态代理和 CGLIB 代理的区别是什么？

代理模式是 AOP 的底层实现机制，代理对象充当客户端和目标对象之间的"中间人"，在转发方法调用前后可以插入额外逻辑。

Spring 根据目标类的情况选择代理策略：

**JDK 动态代理**：基于接口，代理类实现目标类的接口。要求目标类必须有接口，代理对象只能转型为接口类型。创建速度快，但方法调用通过反射，有一定开销。

**CGLIB 代理**：基于继承，代理类是目标类的子类，通过字节码技术（ASM）生成。目标类不需要实现接口，方法调用不走反射，性能更好。但 `final` 类和 `final` 方法无法被代理（子类无法覆盖）。

**Spring 的选择策略：**
- 目标类实现了接口 → 默认用 JDK 动态代理
- 目标类没有接口 → 自动用 CGLIB
- 配置 `proxyTargetClass = true` → 强制全部用 CGLIB

Spring Boot 2.x 开始默认强制使用 CGLIB，避免了因为"目标类有接口但代理对象无法注入到接口变量"带来的问题。

```java
// 强制使用 CGLIB
@EnableAspectJAutoProxy(proxyTargetClass = true)
```

📖 [[../../../24_SpringKnowledge/02_AOP面向切面/01、AOP面向切面编程#七、JDK 动态代理 vs CGLIB 代理]]

---

###### 7. Spring AOP 的底层实现原理是什么？

Spring AOP 的底层实现可以分四个环节理解：

**1. 检测 Bean 是否需要代理**：容器初始化 Bean 时，`AbstractAutoProxyCreator`（一个 `BeanPostProcessor`）会检查该 Bean 是否匹配任何切入点表达式，如果匹配，就会为它创建代理。

**2. 选择代理策略**：根据目标类是否有接口，选 JDK 动态代理还是 CGLIB。

**3. 构建拦截器链**：收集所有匹配当前方法的通知（`Advisor`），按照 `@Order` 排序，组成有序的拦截器链（`MethodInterceptor` 列表）。

**4. 运行时拦截**：客户端调用代理方法 → 进入 `JdkDynamicAopProxy.invoke()` 或 CGLIB 的 `MethodInterceptor` → 构建 `ReflectiveMethodInvocation` → 调用 `proceed()` 方法递归遍历拦截器链 → 最终调用目标方法。

这就是为什么 AOP 的"同类内部调用不生效"：同类内部调用不经过代理对象，直接调目标方法，自然绕过了拦截器链。

📖 [[../../../24_SpringKnowledge/02_AOP面向切面/01、AOP面向切面编程#六、底层实现原理]]

---

###### 8. 切入点表达式如何编写？

最常用的是 `execution` 表达式，语法结构是：

```
execution([修饰符] 返回值类型 [包名.类名.]方法名(参数类型列表) [throws 异常类型])
```

**常用通配符：**
- `*`：匹配任意单个元素（包名、类名、方法名、返回类型）
- `..`：用在包名里匹配任意多级子包；用在参数里匹配任意参数列表
- `+`：匹配指定类及其子类型

**常用示例：**

```
execution(public * *(..))                    匹配所有 public 方法
execution(* set*(..))                        匹配所有以 set 开头的方法
execution(* com.example.service.*.*(..))     匹配 service 包下所有类的所有方法
execution(* com.example.service..*.*(..))    匹配 service 包及其子包下所有方法
execution(* com.example..UserService.save(..)) 匹配 UserService 的 save 方法（任意子包）
```

**其他常用指示符：**

- `@annotation(com.example.Monitor)`：匹配带有 `@Monitor` 注解的方法
- `within(com.example.service.*)`：匹配 service 包下所有类型（类级别匹配）
- `@within(org.springframework.stereotype.Service)`：匹配带 `@Service` 注解的类里的所有方法
- `args(java.lang.String, ..)`：匹配第一个参数是 String 类型的方法

多个条件可以用 `&&`、`||`、`!` 组合：

```java
@Pointcut("execution(* com.example.service.*.*(..)) && @annotation(com.example.Log)")
public void serviceLogMethod() {}
```

📖 [[../../../24_SpringKnowledge/02_AOP面向切面/01、AOP面向切面编程#三、切入点表达式语法]]

---

###### 9. AOP 有哪些应用场景？

AOP 特别适合那些"每个业务方法都要做，但又不应该写进业务代码"的场景：

**声明式事务管理**：Spring `@Transactional` 就是 AOP 的最经典应用，开启事务/提交/回滚全由 AOP 处理，业务代码一行事务 API 不用写。

**统一日志记录**：在方法入口/出口记录方法名、参数、返回值、耗时，不用在每个方法里手动加 log。

**权限控制**：方法执行前检查当前用户是否有权限，没有就抛异常，业务代码不需要关心权限逻辑。

**性能监控**：用 `@Around` 计算方法执行时间，慢方法记录告警，配合 Micrometer 做指标上报。

**接口幂等性控制**：在方法执行前检查幂等 key，防止重复提交。

**限流与熔断**：基于 AOP 拦截方法，集成令牌桶或 Sentinel 规则。

**缓存管理**：Spring `@Cacheable`、`@CacheEvict` 背后就是 AOP 驱动的。

**数据脱敏**：返回给前端的数据自动对手机号、身份证等字段做脱敏处理。

📖 [[../../../24_SpringKnowledge/02_AOP面向切面/01、AOP面向切面编程#九、AOP 应用场景]]

---

###### 9. Spring AOP 代理创建的源码流程是什么？——高频面试引导问题

**代理创建核心链路**：

```
@EnableAspectJAutoProxy
└── 注册 AnnotationAwareAspectJAutoProxyCreator（BeanPostProcessor）
    └── postProcessAfterInitialization()
        └── wrapIfNecessary()
            ├── getAdvicesAndAdvisorsForBean()  ← 查找匹配的切面
            └── createProxy()                   ← 创建代理
                ├── 有接口 → JDK动态代理
                └── 无接口或proxyTargetClass=true → CGLIB
```

**回答示例**：

> Spring AOP 代理是在 Bean 初始化完成后（`postProcessAfterInitialization`）创建的：
> 1. `AnnotationAwareAspectJAutoProxyCreator` 是个 `BeanPostProcessor`，容器创建每个 Bean 时都会经过它
> 2. `wrapIfNecessary()` 检查该 Bean 是否有匹配的切面，没有就直接返回
> 3. 有切面则调用 `createProxy()`，根据目标类是否有接口决定用 JDK 还是 CGLIB
>
> 追问：切面是怎么查找的？
> 从 IoC 容器里找所有带 `@Aspect` 的 Bean，解析里面的 `@Before/@After/@Around` 等方法，生成 `Advisor` 列表，然后和当前 Bean 的方法做切点表达式匹配。

---

###### 10. Spring AOP 方法调用拦截器链是怎么工作的？——高频面试引导问题

**拦截器链执行原理**：

```java
// 代理方法调用入口（CGLIB）
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) {
    List<Object> chain = advisedSupport.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
    if (chain.isEmpty()) {
        return methodProxy.invoke(target, args);  // 无拦截器，直接调用
    }
    // 构建拦截器链，递归执行
    ReflectiveMethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
    return invocation.proceed();
}

// proceed() 递归执行：当前拦截器调用proceed()才会走到下一个
public Object proceed() throws Throwable {
    if (this.currentInterceptorIndex == interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();  // 最后一个：调用目标方法
    }
    Object interceptorOrInterceptionAdvice = interceptorsAndDynamicMethodMatchers.get(++currentInterceptorIndex);
    return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);  // 调用下一个
}
```

**通知执行顺序**：

```
@Around前 → @Before → 目标方法 → @AfterReturning/@AfterThrowing → @After → @Around后
```

**回答示例**：

> 拦截器链是个递归调用：
> - `ReflectiveMethodInvocation.proceed()` 维护一个下标 `currentInterceptorIndex`
> - 每个拦截器执行完自己的逻辑后调用 `invocation.proceed()` 推进下标
> - 下标到头就调用目标方法，然后一层层返回
> - 这就是为什么 `@Around` 能控制目标方法是否执行—— `@Around` 的 `pjp.proceed()` 就是这个递归调用

---

###### 11. Spring AOP 为什么自调用不生效？怎么解决？——高频面试引导问题

**原因**：

```java
// AOP 通过代理对象拦截方法调用
// this.b() = 目标对象直接调用，不经过代理，AOP失效

@Service
public class OrderService {
    @Transactional
    public void createOrder() {
        this.sendMessage();  // ❌ this是目标对象，不是代理
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendMessage() {}
}
```

**解决方案对比**：

| 方案 | 做法 | 优缺点 |
|------|------|--------|
| 注入自身 | `@Autowired private OrderService self` | 简单，有循环依赖风险 |
| AopContext | `((OrderService) AopContext.currentProxy()).sendMessage()` | 耦合度高，需要开启exposeProxy |
| 抽出新类 | 把 sendMessage 移到另一个 Service | 最干净，推荐 |

**回答示例**：

> 我们项目遇到自调用失效，最终采用的是抽出新类：
> - 把需要独立事务的方法抽到 `NotificationService`
> - 注入 `NotificationService` 调用，就经过代理了
>
> 为什么不用 AopContext？代码可读性差，而且有隐式依赖，测试时容易出问题。

---

📖 [[../../../24_SpringKnowledge/09_源码深度解析/03_AOP源码/01、AOP代理创建源码]]
