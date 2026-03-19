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

## 十一、PDF补充内容：AOP源码深度解析

### 11.1 AOP核心概念

| 概念 | 说明 |
|------|------|
| **Aspect（切面）** | 切面泛指交叉业务逻辑，常用的切面是通知（Advice），是对主业务逻辑的一种增强 |
| **Pointcut（切入点）** | 声明的一个或多个连接点的集合，通过切入点指定一组方法。被标记为final的方法不能作为连接点与切入点 |
| **Advice（通知/增强）** | 表示切面的执行时间，定义了增强代码切入到目标代码的时间点（之前/之后执行等） |
| **JoinPoint（连接点）** | 连接切面的业务方法，指可以被切面织入的具体方法，通常业务接口中的方法均为连接点 |
| **Target（目标对象）** | 指将要被增强的对象，即包含主业务逻辑的类的对象 |

### 11.2 @EnableAspectJAutoProxy 注册过程

```java
// @EnableAspectJAutoProxy → @Import(AspectJAutoProxyRegistrar.class)
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry) {
        // 注册AnnotationAwareAspectJAutoProxyCreator
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
        
        // 处理proxyTargetClass属性
        if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
        }
        // 处理exposeProxy属性
        if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
        }
    }
}
```

### 11.3 APC优先级别说明

Spring的抽象类AbstractAdvisorAutoProxyCreator默认有4个实现类，优先级如下（数字越大优先级越高）：

| 优先级 | APC实现类 |
|--------|----------|
| 第0级别 | InfrastructureAdvisorAutoProxyCreator |
| 第1级别 | AspectJAwareAdvisorAutoProxyCreator |
| 第2级别 | AnnotationAwareAspectJAutoProxyCreator |

```java
// APC优先级列表
private static final List<Class<?>> APC_PRIORITY_LIST = new ArrayList<>(3);
static {
    APC_PRIORITY_LIST.add(InfrastructureAdvisorAutoProxyCreator.class);  // 第0级别
    APC_PRIORITY_LIST.add(AspectJAwareAdvisorAutoProxyCreator.class);   // 第1级别
    APC_PRIORITY_LIST.add(AnnotationAwareAspectJAutoProxyCreator.class); // 第2级别
}
```

### 11.4 proxyTargetClass与exposeProxy属性

**proxyTargetClass属性**：
- 配置为`true`时，强制使用CGLIB代理
- 配置方式：`<aop:aspectj-autoproxy proxy-target-class="true"/>`

**exposeProxy属性**：
- 配置为`true`时，支持通过`AopContext.currentProxy()`获取当前代理类
- 解决内部方法调用不走代理的问题

**使用场景举例**：
```java
@Service
public class AServiceImpl implements AService {
    @Transactional(propagation = Propagation.REQUIRED)
    public void a() {
        // 由于this指向target对象，所以不会执行b事务切面
        this.b(); 
        
        // 暴露了当前的代理类，所以可以执行b事务切面
        ((AService) AopContext.currentProxy()).b(); 
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void b() {}
}
```

### 11.5 findCandidateAdvisors() 获取所有增强器

获取所有增强器分为两个步骤：

**步骤1**：寻找在IOC中注册过的Advisor接口的实现类
```java
protected List<Advisor> findCandidateAdvisors() {
    return this.advisorRetrievalHelper.findAdvisorBeans();
}
```

**步骤2**：寻找在IOC中注册过的使用@Aspect注解的增强类
```java
// 通过buildAspectJAdvisors()方法解析@Aspect注解的类
List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
```

### 11.6 @DeclareParents注解的使用

@DeclareParents是引介增强的注解形式实现，可以在代理目标类上动态添加新的方法（实现接口）。

**使用示例**：
```java
// 定义支付接口
public interface IPay {
    void pay();
}

// 定义支付插件接口
public interface IPayPlugin {
    void payPlugin();
}

// 支付插件实现类
@Service
public class AlipayPlugin implements IPayPlugin {
    @Override
    public void payPlugin() {
        System.out.println("-------Alipay--------");
    }
}

// 使用@DeclareParents配置切面
@Aspect
@Component
public class PayAspectJ {
    // value: 目标接口（IPay+表示IPay的子类）
    // defaultImpl: 实现的接口
    @DeclareParents(value = "com.muse.springbootdemo.entity.aop.IPay+",
                    defaultImpl = AlipayPlugin.class)
    public IPayPlugin alipayPlugin;
}
```

### 11.7 wrapIfNecessary() 不需要增强的情况

以下情况不需要增强：
1. **targetSourcedBeans中已存在**：已经处理过的bean
2. **advisedBeans中value为false**：明确不需要增强
3. **基础设施类**：实现Advices、Pointcut、Advisors、AopInfrastructureBeans四个接口的类
4. **Original实例**：以beanClass.getName()开头，并以".ORIGINAL"结尾的类

### 11.8 创建代理的流程（createProxy）

```java
protected Object createProxy(Class<?> beanClass, String beanName, Object[] specificInterceptors, 
                             TargetSource targetSource) {
    // 步骤1：创建ProxyFactory实例对象
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);
    
    // 步骤2-4：设置代理方式（JDK或CGLIB）
    if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) 
            proxyFactory.setProxyTargetClass(true);
        else 
            evaluateProxyInterfaces(beanClass, proxyFactory);
    }
    
    // 步骤5：获得所有增强器并添加到proxyFactory
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    
    // 步骤6：设置被代理的类
    proxyFactory.setTargetSource(targetSource);
    
    // 步骤7：定制代理（空方法）
    customizeProxyFactory(proxyFactory);
    
    // 步骤8：通过getProxy获得代理对象
    return proxyFactory.getProxy(classLoader);
}
```

### 11.9 JDK动态代理与CGLIB代理的选择

```java
public AopProxy createAopProxy(AdvisedSupport config) {
    if (!NativeDetector.inNativeImage() &&
        (config.isOptimize() || config.isProxyTargetClass() || 
         hasNoUserSuppliedProxyInterfaces(config))) {
        
        Class<?> targetClass = config.getTargetClass();
        
        // 如果是接口或代理类，使用JDK动态代理
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        // 否则，使用CGLIB代理
        return new ObjenesisCglibAopProxy(config);
    }
    return new JdkDynamicAopProxy(config);
}
```

**选择规则**：
- 目标类是接口 → JDK动态代理
- proxyTargetClass=true → CGLIB代理
- 没有实现接口 → CGLIB代理

### 11.10 ReflectiveMethodInvocation.proceed() 执行流程

```java
public Object proceed() throws Throwable {
    // 执行完所有增强后执行切点方法
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();
    }
    
    // 获取下一个要执行的拦截器
    Object interceptorOrInterceptionAdvice = 
        this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    
    // 动态匹配
    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
        // 运行时匹配切点
        return dm.interceptor.invoke(this);
    }
    // 普通拦截器，直接调用
    return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
}
```

拦截器链执行顺序（以@Around + @Before + @After为例）：
```
ExposeInvocationInterceptor.invoke()
└── AspectJAroundAdvice.invoke()      ← @Around前半段
    └── MethodBeforeAdviceInterceptor.invoke()
        ├── @Before方法执行
        └── proceed() → 目标方法执行
        ← @AfterReturning方法执行
    ← @After方法执行（finally中）
← @Around后半段
```

---

## 十二、相关源码文件

- [[02、AOP切点与通知源码]]
- [[../04_事务源码/01、@Transactional源码解析]]

