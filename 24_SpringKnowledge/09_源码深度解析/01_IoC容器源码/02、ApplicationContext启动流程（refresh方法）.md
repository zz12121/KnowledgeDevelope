# ApplicationContext 启动流程源码解析（refresh 方法）

> **核心类**：`AbstractApplicationContext`  
> **核心方法**：`refresh()`  
> Spring 容器启动的一切，都从这个方法展开。

---

## 一、refresh() 方法全貌

```java
// ===================== AbstractApplicationContext.java =====================
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {

        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

        // ① 刷新前的准备工作
        prepareRefresh();

        // ② 获取/刷新 BeanFactory（加载 BeanDefinition）
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // ③ 配置 BeanFactory（注册内置组件）
        prepareBeanFactory(beanFactory);

        try {
            // ④ 子类扩展点：BeanFactory 准备完成后的后置处理
            postProcessBeanFactory(beanFactory);

            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");

            // ⑤ 调用所有 BeanFactoryPostProcessor（修改 BeanDefinition 的最后时机）
            invokeBeanFactoryPostProcessors(beanFactory);

            // ⑥ 注册所有 BeanPostProcessor（Bean 初始化前后的拦截器）
            registerBeanPostProcessors(beanFactory);

            beanPostProcess.end();

            // ⑦ 初始化 MessageSource（国际化）
            initMessageSource();

            // ⑧ 初始化事件广播器
            initApplicationEventMulticaster();

            // ⑨ 子类扩展点（SpringBoot 在此启动内嵌 Tomcat）
            onRefresh();

            // ⑩ 注册监听器
            registerListeners();

            // ⑪ 实例化所有非懒加载的单例 Bean（最核心步骤）
            finishBeanFactoryInitialization(beanFactory);

            // ⑫ 完成刷新（发布 ContextRefreshedEvent）
            finishRefresh();
        }
        catch (BeansException ex) {
            // 销毁已创建的 Bean，重置 active 状态
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        }
        finally {
            resetCommonCaches();
            contextRefresh.end();
        }
    }
}
```

---

## 二、各步骤详解

### ① prepareRefresh() —— 刷新前准备

```java
protected void prepareRefresh() {
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);

    // 初始化 PropertySource（Web 环境中初始化 ServletContext/ServletConfig 参数）
    initPropertySources();

    // 验证必需的 Environment 属性是否存在
    getEnvironment().validateRequiredProperties();

    // 初始化早期事件集合（在广播器就绪之前发布的事件先存起来）
    if (this.earlyApplicationListeners == null) {
        this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
    }
    this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

---

### ② obtainFreshBeanFactory() —— 获取 BeanFactory

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 刷新内部 BeanFactory（XML 配置会在此解析 XML 加载 BeanDefinition）
    refreshBeanFactory();
    return getBeanFactory();
}

// AnnotationConfigApplicationContext 不重写 refreshBeanFactory，直接复用已有工厂
// ClassPathXmlApplicationContext 重写，会销毁旧工厂再新建，并解析 XML
@Override
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    beanFactory.setSerializationId(getId());
    customizeBeanFactory(beanFactory);
    loadBeanDefinitions(beanFactory);  // 解析 XML / 扫描注解，注册 BeanDefinition
    this.beanFactory = beanFactory;
}
```

---

### ③ prepareBeanFactory() —— 配置标准 BeanFactory

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 设置类加载器
    beanFactory.setBeanClassLoader(getClassLoader());

    // 注册 SpEL 表达式解析器（支持 #{...} 语法）
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(...));

    // 注册属性编辑器注册商（类型转换）
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // 注册 ApplicationContextAwareProcessor（处理 *Aware 接口回调）
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

    // 忽略这些接口的自动注入（通过 Aware 接口注入，不走依赖注入）
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    // ...

    // 注册可解析的依赖类型（BeanFactory/ResourceLoader/ApplicationContext 等可直接 @Autowired）
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // 注册 ApplicationListenerDetector（检测实现 ApplicationListener 的 Bean）
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // 注册环境相关 Bean（environment/systemProperties/systemEnvironment）
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
}
```

---

### ⑤ invokeBeanFactoryPostProcessors() —— 执行 BeanFactory 后置处理器

这是 Spring 最复杂的步骤之一，`ConfigurationClassPostProcessor` 就在此阶段完成注解扫描：

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    // 委托给 PostProcessorRegistrationDelegate
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
}

// PostProcessorRegistrationDelegate 内部执行顺序：
// 1. 先执行 BeanDefinitionRegistryPostProcessor（可以追加注册 BeanDefinition）
//    ① 手动注册的（addBeanFactoryPostProcessor 添加的）
//    ② 实现了 PriorityOrdered 的（ConfigurationClassPostProcessor 就在这里！）
//    ③ 实现了 Ordered 的
//    ④ 剩余的（支持 BDRPP 注册新的 BDRPP，循环处理直到没有新增）
// 2. 再执行 BeanFactoryPostProcessor（只能修改已有 BeanDefinition）
//    ① PriorityOrdered → ② Ordered → ③ 其余
```

**ConfigurationClassPostProcessor** 的核心逻辑：

```java
// ConfigurationClassPostProcessor.java（简化）
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    // ① 找到所有 @Configuration 类
    List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
    for (String beanName : registry.getBeanDefinitionNames()) {
        BeanDefinition beanDef = registry.getBeanDefinition(beanName);
        if (ConfigurationClassUtils.isConfigurationCandidate(beanDef)) {
            configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
        }
    }

    // ② 用 ConfigurationClassParser 解析每个配置类
    ConfigurationClassParser parser = new ConfigurationClassParser(...);
    parser.parse(configCandidates);  // 处理 @ComponentScan/@Import/@Bean 等

    // ③ 用 ConfigurationClassBeanDefinitionReader 把解析结果注册成 BeanDefinition
    this.reader.loadBeanDefinitions(configClasses);
}
```

---

### ⑥ registerBeanPostProcessors() —— 注册 BeanPostProcessor

```java
// 执行顺序和 BFPP 类似：PriorityOrdered → Ordered → 其余 → MergedBeanDefinitionPostProcessor
// 注意：这里只是注册（addBeanPostProcessor），并不执行
// BeanPostProcessor 的执行在每个 Bean 的 initializeBean() 阶段

// 常见内置 BeanPostProcessor：
// AutowiredAnnotationBeanPostProcessor  ← 处理 @Autowired/@Value
// CommonAnnotationBeanPostProcessor     ← 处理 @Resource/@PostConstruct/@PreDestroy
// AbstractAdvisorAutoProxyCreator       ← AOP 代理创建器（AnnotationAwareAspectJAutoProxyCreator）
```

---

### ⑪ finishBeanFactoryInitialization() —— 实例化所有单例 Bean

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 初始化 ConversionService（类型转换服务）
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // 注册默认的嵌入值解析器（@Value 注解的 ${...} 解析）
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // 初始化 LoadTimeWeaverAware Bean（AspectJ 编译期/类加载期织入）
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // 冻结 BeanDefinition（之后不允许再注册/修改 BeanDefinition）
    beanFactory.freezeConfiguration();

    // ★ 核心：预实例化所有非懒加载的单例 Bean
    beanFactory.preInstantiateSingletons();
}

// DefaultListableBeanFactory.preInstantiateSingletons()
@Override
public void preInstantiateSingletons() throws BeansException {
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 非抽象、非懒加载、单例 → 触发 getBean（第一次获取时创建）
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                // FactoryBean：先获取工厂本身，再决定是否提前初始化产品
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof SmartFactoryBean<?> smartFactoryBean
                        && smartFactoryBean.isEagerInit()) {
                    getBean(beanName);
                }
            } else {
                getBean(beanName);  // ← 触发 doGetBean → createBean → 完整生命周期
            }
        }
    }

    // 所有单例 Bean 创建完毕，回调 SmartInitializingSingleton.afterSingletonsInstantiated()
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton smartSingleton) {
            smartSingleton.afterSingletonsInstantiated();
        }
    }
}
```

---

### ⑫ finishRefresh() —— 完成刷新

```java
protected void finishRefresh() {
    // 清理资源缓存（如 ASM 的元数据缓存）
    clearResourceCaches();

    // 初始化 LifecycleProcessor（管理 Bean 的生命周期：start/stop）
    initLifecycleProcessor();

    // 触发 Lifecycle Bean 的 start()
    getLifecycleProcessor().onRefresh();

    // ★ 发布 ContextRefreshedEvent（所有监听器可以在此做最终初始化）
    publishEvent(new ContextRefreshedEvent(this));
}
```

---

## 三、refresh() 12步总结图

```
refresh()
├── ① prepareRefresh()           → 设置启动时间、active=true、初始化 PropertySources
├── ② obtainFreshBeanFactory()   → 加载 XML 或注解，注册 BeanDefinition
├── ③ prepareBeanFactory()       → 配置 ClassLoader、注册 Aware 处理器、注册可解析依赖
├── ④ postProcessBeanFactory()   → 子类扩展（Web 环境注册 Web 相关 Scope）
├── ⑤ invokeBeanFactoryPostProcessors() → ★ 执行 BFPP（ConfigurationClassPostProcessor 在此完成扫描）
├── ⑥ registerBeanPostProcessors()      → 注册所有 BPP（AutowiredAnnotationBeanPostProcessor等）
├── ⑦ initMessageSource()        → 初始化国际化
├── ⑧ initApplicationEventMulticaster() → 初始化事件广播器
├── ⑨ onRefresh()                → 子类扩展（SpringBoot 启动 Tomcat）
├── ⑩ registerListeners()        → 注册 ApplicationListener
├── ⑪ finishBeanFactoryInitialization() → ★★ 实例化所有非懒加载单例（调用每个 Bean 的完整生命周期）
└── ⑫ finishRefresh()            → 发布 ContextRefreshedEvent，启动 Lifecycle
```

---

## 四、常见面试问题

| 问题 | 答案要点 |
|------|---------|
| Spring 容器的启动流程？ | 12 步 refresh()，重点是 ⑤⑥⑪ |
| @ComponentScan 是什么时候扫描的？ | 第 ⑤ 步，ConfigurationClassPostProcessor 解析 |
| @Autowired 是什么时候注入的？ | 第 ⑪ 步，Bean 创建时由 AutowiredAnnotationBeanPostProcessor 处理 |
| Spring 发布了哪些容器事件？ | ContextRefreshedEvent / ContextClosedEvent / ContextStartedEvent / ContextStoppedEvent |

---

## 五、相关源码文件

- [[01、BeanFactory体系结构]]
- [[../02_Bean生命周期源码/01、Bean实例化源码]]
- [[../07_扩展点源码/01、BeanFactoryPostProcessor源码]]
- [[../06_SpringBoot自动配置源码/01、SpringApplication启动流程]]
