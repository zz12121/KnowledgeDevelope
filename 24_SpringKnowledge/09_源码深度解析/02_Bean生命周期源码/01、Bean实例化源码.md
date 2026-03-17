# Bean 实例化源码解析（createBean / doCreateBean）

> **核心类**：`AbstractAutowireCapableBeanFactory`  
> **核心方法**：`createBean()` → `doCreateBean()`

---

## 一、Bean 创建总流程

```
getBean(beanName)
└── doGetBean()
    └── createBean(beanName, mbd, args)          ← AbstractAutowireCapableBeanFactory
        ├── resolveBeforeInstantiation()          ← ① 实例化前置处理（InstantiationAwareBeanPostProcessor）
        └── doCreateBean(beanName, mbd, args)
            ├── createBeanInstance()              ← ② 实例化（反射调用构造器）
            ├── applyMergedBeanDefinitionPostProcessors() ← ③ 合并 BeanDefinition 后置处理
            ├── addSingletonFactory()             ← ④ 放入三级缓存（提前暴露引用）
            ├── populateBean()                    ← ⑤ 属性填充（依赖注入）
            └── initializeBean()                  ← ⑥ 初始化
                ├── invokeAwareMethods()           ← 6.1 Aware 回调
                ├── applyBeanPostProcessorsBeforeInitialization() ← 6.2 BPP 前置处理
                ├── invokeInitMethods()            ← 6.3 调用 init 方法
                └── applyBeanPostProcessorsAfterInitialization()  ← 6.4 BPP 后置处理（AOP 在此）
```

---

## 二、createBean() 入口

```java
// AbstractAutowireCapableBeanFactory.java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {

    RootBeanDefinition mbdToUse = mbd;

    // 解析 Bean 的 Class（如果是字符串类名，此处反射加载）
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    // 处理 lookup-method 和 replaced-method（方法替换，较少用）
    mbdToUse.prepareMethodOverrides();

    // ① InstantiationAwareBeanPostProcessor 前置处理
    //    如果返回非 null（如动态代理替换），直接返回，跳过后续所有步骤
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
        return bean;
    }

    // ② 正式创建 Bean
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    return beanInstance;
}
```

---

## 三、doCreateBean() —— 核心创建逻辑

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {

    BeanWrapper instanceWrapper = null;

    // 如果是单例，先清除 FactoryBean 缓存
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }

    // ② 实例化 Bean（只是 new 出来，属性还未填充）
    if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();

    // ③ 允许 MergedBeanDefinitionPostProcessor 修改 BeanDefinition
    //    AutowiredAnnotationBeanPostProcessor 在此收集 @Autowired 字段/方法信息
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            mbd.postProcessed = true;
        }
    }

    // ④ 提前暴露：将 Bean 工厂放入三级缓存（解决循环依赖）
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences
            && isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    Object exposedObject = bean;
    try {
        // ⑤ 属性填充（@Autowired 依赖注入在此执行）
        populateBean(beanName, mbd, instanceWrapper);

        // ⑥ 初始化（Aware → BPP前置 → init → BPP后置）
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, ex.getMessage(), ex);
    }

    // 处理早期引用（如果存在循环依赖，确保暴露的是代理对象）
    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;  // 使用代理对象替换
            }
        }
    }

    // 注册销毁回调（@PreDestroy / DisposableBean / destroy-method）
    registerDisposableBeanIfNecessary(beanName, bean, mbd);

    return exposedObject;
}
```

---

## 四、createBeanInstance() —— 实例化策略

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    // ① 如果有 Supplier（函数式注册 Bean），直接调用
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }

    // ② 如果有工厂方法（@Bean 注解、factory-method XML），走工厂方法创建
    if (mbd.getFactoryMethodName() != null) {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // ③ 短路优化：如果已缓存了构造器/工厂方法，直接用
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    if (resolved) {
        if (autowireNecessary) {
            return autowireConstructor(beanName, mbd, null, null);  // 构造器注入
        } else {
            return instantiateBean(beanName, mbd);  // 无参构造
        }
    }

    // ④ 让 SmartInstantiationAwareBeanPostProcessor 决定构造器
    //    AutowiredAnnotationBeanPostProcessor 在此推断 @Autowired 构造器
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR
            || mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        return autowireConstructor(beanName, mbd, ctors, args);  // 有参构造器注入
    }

    // ⑤ 默认：使用无参构造器 + CGLIB/反射实例化
    ctors = mbd.getPreferredConstructors();
    if (ctors != null) {
        return autowireConstructor(beanName, mbd, ctors, null);
    }
    return instantiateBean(beanName, mbd);  // 反射调用无参构造器
}
```

---

## 五、populateBean() —— 属性填充（依赖注入）

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    // ① InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()
    //    返回 false 可跳过属性填充（很少用）
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                return;  // 跳过属性填充
            }
        }
    }

    // ② 处理 XML/Java Config 中配置的 autowireMode（BY_NAME / BY_TYPE）
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
        autowireByName(beanName, mbd, bw, newPvs);
    } else if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        autowireByType(beanName, mbd, bw, newPvs);
    }

    // ③ ★ InstantiationAwareBeanPostProcessor.postProcessProperties()
    //    AutowiredAnnotationBeanPostProcessor 在此处理 @Autowired/@Value
    //    CommonAnnotationBeanPostProcessor 在此处理 @Resource
    if (hasInstantiationAwareBeanPostProcessors()) {
        PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
    }

    // ④ 将属性值应用到 Bean 实例（XML 中配置的 <property>）
    if (pvs != null) {
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

---

## 六、initializeBean() —— 初始化

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {

    // 6.1 Aware 接口回调
    invokeAwareMethods(beanName, bean);
    // 处理：BeanNameAware / BeanClassLoaderAware / BeanFactoryAware

    // 6.2 BeanPostProcessor 前置处理（@PostConstruct 由此触发）
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
        // CommonAnnotationBeanPostProcessor 在此调用 @PostConstruct 方法
    }

    // 6.3 调用 init 方法
    invokeInitMethods(beanName, wrappedBean, mbd);
    // 顺序：① InitializingBean.afterPropertiesSet() → ② 自定义 initMethod

    // 6.4 BeanPostProcessor 后置处理（★ AOP 代理在此创建）
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        // AbstractAutoProxyCreator 在此创建 AOP 代理，替换原始 Bean
    }

    return wrappedBean;
}

protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd)
        throws Throwable {
    // ① InitializingBean 接口
    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean && (mbd == null || !mbd.hasAnyExternallyManagedInitMethod("afterPropertiesSet"))) {
        ((InitializingBean) bean).afterPropertiesSet();
    }

    // ② @Bean(initMethod = "xxx") 或 XML <bean init-method="xxx">
    if (mbd != null && bean.getClass() != NullBean.class) {
        String[] initMethodNames = mbd.getInitMethodNames();
        if (initMethodNames != null) {
            for (String initMethodName : initMethodNames) {
                invokeCustomInitMethod(beanName, bean, mbd, initMethodName);
            }
        }
    }
}
```

---

## 七、Bean 完整生命周期总结

```
① 实例化（Instantiation）
   ├── InstantiationAwareBPP.postProcessBeforeInstantiation()  ← 可短路，返回代理
   ├── 构造器/工厂方法创建实例
   └── InstantiationAwareBPP.postProcessAfterInstantiation()   ← 可跳过属性填充

② 属性填充（Population）
   ├── autowireByName/ByType（XML 配置的自动注入模式）
   └── InstantiationAwareBPP.postProcessProperties()           ← @Autowired/@Value/@Resource

③ 初始化（Initialization）
   ├── BeanNameAware.setBeanName()
   ├── BeanClassLoaderAware.setBeanClassLoader()
   ├── BeanFactoryAware.setBeanFactory()
   ├── ApplicationContextAware.setApplicationContext()（通过 ApplicationContextAwareProcessor）
   ├── BPP.postProcessBeforeInitialization()                   ← @PostConstruct
   ├── InitializingBean.afterPropertiesSet()
   ├── 自定义 init-method
   └── BPP.postProcessAfterInitialization()                    ← ★ AOP 代理创建

④ 使用（In Use）

⑤ 销毁（Destruction）
   ├── BPP.postProcessBeforeDestruction()
   ├── @PreDestroy 方法（CommonAnnotationBeanPostProcessor）
   ├── DisposableBean.destroy()
   └── 自定义 destroy-method
```

---

## 八、相关源码文件

- [[01、BeanFactory体系结构]]
- [[02、ApplicationContext启动流程（refresh方法）]]
- [[../03_AOP源码/01、AOP代理创建源码]]
- [[../07_扩展点源码/02、BeanPostProcessor源码]]
