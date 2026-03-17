# AOP 代理创建源码解析

> **核心类**：`AnnotationAwareAspectJAutoProxyCreator`  
> **核心包**：`org.springframework.aop.framework.autoproxy`  
> **触发时机**：Bean 生命周期的 `BeanPostProcessor.postProcessAfterInitialization()`

---

## 一、AOP 整体架构

```
AOP 代理创建链路：
@EnableAspectJAutoProxy
└── 注册 AnnotationAwareAspectJAutoProxyCreator（BeanPostProcessor）
    └── postProcessAfterInitialization()
        └── wrapIfNecessary()
            ├── getAdvicesAndAdvisorsForBean()  ← 查找匹配的切面/增强
            └── createProxy()                   ← 创建代理对象
                ├── JDK 动态代理（实现了接口）
                └── CGLIB 代理（没有接口 / proxyTargetClass=true）

AOP 运行时拦截链路：
代理对象方法调用
└── JdkDynamicAopProxy.invoke() / CglibAopProxy.intercept()
    └── ReflectiveMethodInvocation.proceed()
        ├── 执行拦截器链（MethodInterceptor）
        └── 最终调用目标方法
```

---

## 二、@EnableAspectJAutoProxy 注册创建器

```java
// @EnableAspectJAutoProxy → @Import(AspectJAutoProxyRegistrar.class)
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry) {
        // 注册 AnnotationAwareAspectJAutoProxyCreator 的 BeanDefinition
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

        AnnotationAttributes enableAspectJAutoProxy =
                AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        if (enableAspectJAutoProxy != null) {
            if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
                AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
            }
            if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
                AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
            }
        }
    }
}
```

---

## 三、AbstractAutoProxyCreator —— 代理创建核心

### 3.1 postProcessAfterInitialization

```java
// AbstractAutoProxyCreator.java
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        // 不在早期引用处理列表中（避免重复代理）
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```

### 3.2 wrapIfNecessary —— 判断是否需要代理

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    // 已处理过的 Bean 或基础设施 Bean，跳过
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    // AOP 基础设施类（Advice/Pointcut/Advisor 等）本身不被代理
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // ★ 查找所有适用于当前 Bean 的增强（Advisor）
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);

    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // ★ 创建代理对象
        Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors,
                new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

---

## 四、查找 Advisor（切面匹配）

```java
// AbstractAdvisorAutoProxyCreator.java
@Override
@Nullable
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName,
        @Nullable TargetSource targetSource) {
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}

protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    // ① 获取所有候选 Advisor（从容器中找出所有 Advisor Bean）
    List<Advisor> candidateAdvisors = findCandidateAdvisors();

    // ② 筛选出能匹配当前 Bean 的 Advisor（通过 Pointcut 判断）
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);

    // ③ 添加 ExposeInvocationInterceptor（拦截链第一个，用于存储当前调用）
    extendAdvisors(eligibleAdvisors);

    if (!eligibleAdvisors.isEmpty()) {
        // ④ 按照 @Order / Ordered 接口排序
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}

// AnnotationAwareAspectJAutoProxyCreator 重写了 findCandidateAdvisors
@Override
protected List<Advisor> findCandidateAdvisors() {
    // 找 Spring 传统 Advisor Bean
    List<Advisor> advisors = super.findCandidateAdvisors();
    // ★ 找 @Aspect 注解的 Bean，并将其中的 @Before/@After 等解析为 Advisor
    if (this.aspectJAdvisorsBuilder != null) {
        advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
    }
    return advisors;
}
```

---

## 五、@Aspect 解析 —— buildAspectJAdvisors

```java
// BeanFactoryAspectJAdvisorsBuilder.java
public List<Advisor> buildAspectJAdvisors() {
    List<String> aspectNames = this.aspectBeanNames;

    if (aspectNames == null) {
        synchronized (this) {
            aspectNames = this.aspectBeanNames;
            if (aspectNames == null) {
                List<Advisor> advisors = new ArrayList<>();
                List<String> aspectNamesLocal = new ArrayList<>();
                String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                        this.beanFactory, Object.class, true, false);

                for (String beanName : beanNames) {
                    if (!isEligibleBean(beanName)) continue;
                    Class<?> beanType = this.beanFactory.getType(beanName, false);
                    if (beanType == null) continue;

                    if (this.advisorFactory.isAspect(beanType)) {
                        // 是 @Aspect 类
                        aspectNamesLocal.add(beanName);
                        AspectMetadata amd = new AspectMetadata(beanType, beanName);

                        if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                            MetadataAwareAspectInstanceFactory factory =
                                    new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                            // ★ 将 @Before/@After/@Around 等解析为 InstantiationModelAwarePointcutAdvisor
                            List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                            this.advisorsCache.put(beanName, classAdvisors);
                            advisors.addAll(classAdvisors);
                        }
                    }
                }
                this.aspectBeanNames = aspectNamesLocal;
                return advisors;
            }
        }
    }
    // 从缓存获取
    List<Advisor> advisors = new ArrayList<>();
    for (String aspectName : aspectNames) {
        List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
        if (cachedAdvisors != null) {
            advisors.addAll(cachedAdvisors);
        }
    }
    return advisors;
}
```

---

## 六、创建代理对象 —— createProxy

```java
// AbstractAutoProxyCreator.java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
        @Nullable Object[] specificInterceptors, TargetSource targetSource) {

    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);

    // 设置目标对象
    proxyFactory.setTargetSource(targetSource);

    // 设置代理接口
    evaluateProxyInterfaces(beanClass, proxyFactory);

    // 整合拦截器：将 MethodInterceptor/Advice 包装成 Advisor
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);

    // ★ 通过 ProxyFactory 创建代理
    return proxyFactory.getProxy(classLoader);
}

// ProxyFactory.java
public Object getProxy(@Nullable ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}

// DefaultAopProxyFactory.java —— 决定用 JDK 还是 CGLIB
@Override
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (!NativeDetector.inNativeImage() &&
            (config.isOptimize() || config.isProxyTargetClass()
                    || hasNoUserSuppliedProxyInterfaces(config))) {
        Class<?> targetClass = config.getTargetClass();
        // 目标类是接口 或 已经是代理类 → 用 JDK 动态代理
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)
                || ClassUtils.isLambdaClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        // 否则用 CGLIB
        return new ObjenesisCglibAopProxy(config);
    } else {
        return new JdkDynamicAopProxy(config);
    }
}
```

---

## 七、方法拦截执行链 —— JdkDynamicAopProxy

```java
// JdkDynamicAopProxy.java
@Override
@Nullable
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;
    TargetSource targetSource = this.advised.targetSource;
    Object target = null;

    try {
        // equals/hashCode 方法不走代理
        if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
            return equals(args[0]);
        }
        if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
            return hashCode();
        }

        // exposeProxy=true 时，将代理对象暴露到 AopContext（解决内部调用不走代理的问题）
        if (this.advised.exposeProxy) {
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);

        // ★ 获取当前方法的拦截器链
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

        if (chain.isEmpty()) {
            // 无拦截器：直接反射调用目标方法
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
        } else {
            // ★ 创建 ReflectiveMethodInvocation，沿链执行
            MethodInvocation invocation = new ReflectiveMethodInvocation(
                    proxy, target, method, args, targetClass, chain);
            retVal = invocation.proceed();
        }
        return retVal;
    } finally {
        if (setProxyContext) {
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```

---

## 八、拦截器链执行 —— ReflectiveMethodInvocation.proceed()

```java
// ReflectiveMethodInvocation.java
@Override
@Nullable
public Object proceed() throws Throwable {
    // 递归终止：所有拦截器执行完毕，调用目标方法
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();  // 反射调用目标方法
    }

    Object interceptorOrInterceptionAdvice =
            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);

    // 动态匹配（运行时再次判断切点是否匹配）
    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher dm) {
        Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
        if (dm.matcher().matches(this.method, targetClass, this.arguments)) {
            return dm.interceptor().invoke(this);
        } else {
            return proceed();  // 不匹配跳过
        }
    } else {
        // ★ 调用拦截器（MethodInterceptor.invoke(this)），并将自身传入
        //    拦截器在合适时机调用 invocation.proceed() 继续链式调用
        return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    }
}
```

**拦截器链执行顺序（以 @Around + @Before + @After 为例）：**

```
ExposeInvocationInterceptor.invoke()
└── AspectJAroundAdvice.invoke()          ← @Around 前半段（ProceedingJoinPoint.proceed() 之前）
    └── MethodBeforeAdviceInterceptor.invoke()
        ├── @Before 方法执行
        └── proceed() → AfterReturningAdviceInterceptor.invoke()
            └── proceed() → 目标方法执行
            ← @AfterReturning 方法执行
    ← @After 方法执行（AspectJAfterAdvice 在 finally 中）
← @Around 后半段（ProceedingJoinPoint.proceed() 之后）
```

---

## 九、常见面试问题

| 问题 | 源码位置 |
|------|---------|
| JDK 代理和 CGLIB 代理的选择条件？ | `DefaultAopProxyFactory.createAopProxy()` |
| Spring AOP 为何无法拦截内部调用？ | 内部调用走的是 `this.method()`，不经过代理 |
| 如何解决内部调用问题？ | `exposeProxy=true` + `AopContext.currentProxy()` |
| @Transactional 的代理是在哪创建的？ | 同一流程，`TransactionInterceptor` 也是 MethodInterceptor |
| 同一个类多个切面如何排序？ | `@Order`/`Ordered` 接口，`sortAdvisors()` |

---

## 十、相关源码文件

- [[../02_Bean生命周期源码/01、Bean实例化源码]]
- [[02、AOP切点与通知源码]]
- [[../04_事务源码/01、@Transactional源码解析]]
