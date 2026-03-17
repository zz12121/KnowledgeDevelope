# AOP 切点与通知源码解析

> **核心类**：`AspectJExpressionPointcut`、`AspectJMethodBeforeAdvice` 等  
> **核心包**：`org.springframework.aop.aspectj`

---

## 一、Pointcut 接口体系

```java
// Pointcut.java —— 切点顶层接口
public interface Pointcut {
    // 类过滤器：判断类是否符合切点
    ClassFilter getClassFilter();
    // 方法匹配器：判断方法是否符合切点
    MethodMatcher getMethodMatcher();

    Pointcut TRUE = TruePointcut.INSTANCE;  // 匹配所有
}

// MethodMatcher.java
public interface MethodMatcher {
    // 静态匹配（编译期/类加载期，不依赖运行时参数）
    boolean matches(Method method, Class<?> targetClass);

    // 是否需要运行时匹配
    boolean isRuntime();

    // 动态匹配（运行时，带参数匹配，只有 isRuntime()=true 时才调用）
    boolean matches(Method method, Class<?> targetClass, Object... args);
}
```

---

## 二、AspectJExpressionPointcut —— @Pointcut 的实现

```java
public class AspectJExpressionPointcut extends AbstractExpressionPointcut
        implements ClassFilter, IntroductionAwareMethodMatcher, BeanFactoryAware {

    // AspectJ 的切点表达式解析器
    private transient PointcutExpression pointcutExpression;

    @Override
    public boolean matches(Class<?> targetClass) {
        // 使用 AspectJ 底层 API 匹配类
        PointcutExpression pointcutExpression = obtainPointcutExpression();
        try {
            FuzzyBoolean classMatchResult = pointcutExpression.couldMatchJoinPointsInType(targetClass);
            if (classMatchResult.alwaysFalse()) {
                return false;
            }
            return true;
        } catch (ReflectionWorldException ex) {
            return true;  // 无法判断时返回 true（保守策略）
        }
    }

    @Override
    public boolean matches(Method method, Class<?> targetClass, boolean hasIntroductions) {
        obtainPointcutExpression();
        ShadowMatch shadowMatch = getTargetShadowMatch(method, targetClass);

        if (shadowMatch.alwaysMatches()) {
            return true;
        } else if (shadowMatch.neverMatches()) {
            return false;
        } else {
            // 不确定 → 检查是否需要动态匹配
            if (hasIntroductions) {
                return true;
            }
            RuntimeTestWalker walker = getRuntimeTestWalker(shadowMatch);
            return (!walker.testsSubtypeSensitiveVars() || walker.testTargetInstanceOfResidue(targetClass));
        }
    }
}
```

---

## 三、Advisor 体系

```java
// Advisor.java —— 切面的基础接口（持有 Advice）
public interface Advisor {
    Advice EMPTY_ADVICE = new Advice() {};
    Advice getAdvice();
    boolean isPerInstance();
}

// PointcutAdvisor —— 带切点的 Advisor（最常用）
public interface PointcutAdvisor extends Advisor {
    Pointcut getPointcut();
}

// InstantiationModelAwarePointcutAdvisor —— @Before/@After 等注解对应的 Advisor
// 由 ReflectiveAspectJAdvisorFactory.getAdvisor() 创建
public class InstantiationModelAwarePointcutAdvisorImpl
        implements InstantiationModelAwarePointcutAdvisor {

    private final AspectJExpressionPointcut declaredPointcut;   // 切点
    private Advice instantiatedAdvice;                           // 通知

    @Override
    public Advice getAdvice() {
        if (this.instantiatedAdvice == null) {
            this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
        }
        return this.instantiatedAdvice;
    }

    private Advice instantiateAdvice(AspectJExpressionPointcut pointcut) {
        // 根据注解类型创建对应的 Advice
        return this.aspectJAdvisorFactory.getAdvice(
                this.aspectJAdviceMethod,   // @Before/@After 等注解所在的方法
                pointcut,
                this.aspectInstanceFactory,
                this.declarationOrder,
                this.aspectName
        );
    }
}
```

---

## 四、各种 Advice 的实现

### 4.1 @Before → MethodBeforeAdviceInterceptor

```java
// AspectJMethodBeforeAdvice.java（Spring 的包装）
public class AspectJMethodBeforeAdvice extends AbstractAspectJAdvice implements MethodBeforeAdvice {
    @Override
    public void before(Method method, Object[] args, @Nullable Object target) throws Throwable {
        // 反射调用 @Before 注解的方法
        invokeAdviceMethod(getJoinPointMatch(), null, null);
    }
}

// MethodBeforeAdviceInterceptor.java（适配为 MethodInterceptor 参与拦截链）
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, BeforeAdvice {
    private final MethodBeforeAdvice advice;

    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        // ① 先执行 @Before 方法
        this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
        // ② 再继续执行拦截链（调用目标方法）
        return mi.proceed();
    }
}
```

### 4.2 @AfterReturning → AfterReturningAdviceInterceptor

```java
public class AfterReturningAdviceInterceptor implements MethodInterceptor, AfterAdvice {
    private final AfterReturningAdvice advice;

    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        // ① 先继续执行（调用目标方法）
        Object retVal = mi.proceed();
        // ② 目标方法正常返回后执行 @AfterReturning
        this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
        return retVal;
    }
}
```

### 4.3 @AfterThrowing → AspectJAfterThrowingAdvice

```java
public class AspectJAfterThrowingAdvice extends AbstractAspectJAdvice
        implements MethodInterceptor, AfterAdvice {

    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        try {
            return mi.proceed();
        } catch (Throwable ex) {
            // 匹配异常类型
            if (shouldInvokeOnThrowing(ex)) {
                invokeAdviceMethod(getJoinPointMatch(), null, ex);
            }
            throw ex;
        }
    }
}
```

### 4.4 @After → AspectJAfterAdvice

```java
public class AspectJAfterAdvice extends AbstractAspectJAdvice
        implements MethodInterceptor, AfterAdvice {

    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        try {
            return mi.proceed();
        } finally {
            // finally 保证无论正常/异常都执行 @After 方法
            invokeAdviceMethod(getJoinPointMatch(), null, null);
        }
    }
}
```

### 4.5 @Around → AspectJAroundAdvice

```java
public class AspectJAroundAdvice extends AbstractAspectJAdvice implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        // 将 MethodInvocation 包装成 ProceedingJoinPoint
        ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(mi);
        JoinPointMatch jpm = getJoinPointMatch(mi);
        // 调用 @Around 方法，用户在其中调用 pjp.proceed() 继续执行
        return invokeAdviceMethod(pjp, jpm, null, null);
    }
}

// ProceedingJoinPoint.proceed() 最终调用的就是 MethodInvocation.proceed()
// 因此 @Around 的"前后"由用户代码控制
```

---

## 五、切面通知执行顺序

**同一切面内的顺序（Spring 5.2.7+ 修正后）：**

```
方法正常执行：
@Around(前) → @Before → 目标方法 → @AfterReturning → @After → @Around(后)

方法抛异常：
@Around(前) → @Before → 目标方法抛异常 → @AfterThrowing → @After → @Around(重新抛出)
```

**多切面排序：**

```java
// 通过 @Order 注解控制切面优先级（值越小，越先进入，越后退出）
@Aspect
@Order(1)  // 外层切面
public class OuterAspect { ... }

@Aspect
@Order(2)  // 内层切面
public class InnerAspect { ... }

// 执行顺序：
// OuterAspect.before → InnerAspect.before → 目标方法
//                   ← InnerAspect.after  ← OuterAspect.after
```

---

## 六、JoinPoint 和 ProceedingJoinPoint 的实现

```java
// MethodInvocationProceedingJoinPoint.java
public class MethodInvocationProceedingJoinPoint implements ProceedingJoinPoint, JoinPoint.StaticPart {

    private final ProxyMethodInvocation methodInvocation;

    // 获取目标方法签名
    @Override
    public Signature getSignature() {
        return new MethodSignatureImpl();
    }

    // 获取目标对象
    @Override
    public Object getTarget() {
        return this.methodInvocation.getThis();
    }

    // 获取方法参数
    @Override
    public Object[] getArgs() {
        return this.methodInvocation.getArguments().clone();
    }

    // @Around 中调用 proceed()
    @Override
    public Object proceed() throws Throwable {
        return this.methodInvocation.invocableClone().proceed();
    }

    @Override
    public Object proceed(Object[] arguments) throws Throwable {
        // 用新参数继续执行
        this.methodInvocation.setArguments(arguments);
        return this.methodInvocation.invocableClone(arguments).proceed();
    }
}
```

---

## 七、常见面试问题

| 问题 | 答案要点 |
|------|---------|
| Spring AOP 的五种通知类型？ | @Before / @AfterReturning / @AfterThrowing / @After / @Around |
| @Around 和 @Before 的区别？ | @Around 包裹整个执行，可以修改参数/返回值/异常；@Before 只在方法前执行 |
| 通知执行顺序？ | @Around前 → @Before → 目标 → @AfterReturning/@AfterThrowing → @After → @Around后 |
| execution 表达式语法？ | `execution(修饰符? 返回类型 包名.类名.方法名(参数) 异常?)` |
| within 和 execution 的区别？ | `within` 按类型匹配，`execution` 按方法签名匹配 |

---

## 八、常用切点表达式速查

```java
// 匹配 service 包下所有类的所有方法
@Pointcut("execution(* com.example.service.*.*(..))")

// 匹配所有 @Service 注解的 Bean 的方法
@Pointcut("within(@org.springframework.stereotype.Service *)")

// 匹配所有带 @Log 注解的方法
@Pointcut("@annotation(com.example.annotation.Log)")

// 匹配所有 Controller 的方法，且参数包含 HttpServletRequest
@Pointcut("within(@org.springframework.web.bind.annotation.RestController *) && args(request,..)")
```

---

## 九、相关源码文件

- [[01、AOP代理创建源码]]
- [[../04_事务源码/01、@Transactional源码解析]]
