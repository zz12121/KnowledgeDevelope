# Spring 扩展点源码解析

> 这是 Spring 框架最重要的扩展机制，理解这些扩展点是框架进阶的关键。  
> **核心包**：`org.springframework.beans.factory.config`、`org.springframework.context`

---

## 一、Spring 扩展点全景图

```
容器启动阶段扩展点：
├── BeanDefinitionRegistryPostProcessor   ← 最早：可以追加注册 BeanDefinition
├── BeanFactoryPostProcessor              ← 可以修改 BeanDefinition
└── Ordered/PriorityOrdered               ← 控制以上两者的执行顺序

Bean 生命周期扩展点：
├── InstantiationAwareBeanPostProcessor   ← 实例化前/后
├── MergedBeanDefinitionPostProcessor     ← 合并 BeanDefinition 后
├── SmartInstantiationAwareBeanPostProcessor ← 推断构造器/早期引用
├── BeanPostProcessor                     ← 初始化前/后（最常用）
└── DestructionAwareBeanPostProcessor     ← 销毁前

Spring 内置的核心扩展实现：
├── ConfigurationClassPostProcessor       ← BeanDefinitionRegistryPostProcessor，处理 @Configuration
├── AutowiredAnnotationBeanPostProcessor  ← InstantiationAwareBPP，处理 @Autowired
├── CommonAnnotationBeanPostProcessor     ← InstantiationAwareBPP，处理 @Resource/@PostConstruct
└── AbstractAutoProxyCreator              ← BeanPostProcessor，AOP 代理创建
```

---

## 二、BeanFactoryPostProcessor 源码

```java
// BeanFactoryPostProcessor.java
@FunctionalInterface
public interface BeanFactoryPostProcessor {
    /**
     * 在所有 BeanDefinition 加载完成、Bean 实例化开始之前调用
     * 可以修改任何 BeanDefinition 的属性（如修改 Bean 的 scope、添加属性值等）
     */
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}

// ★ 典型应用：PropertySourcesPlaceholderConfigurer（解析 ${} 占位符）
public class PropertySourcesPlaceholderConfigurer
        extends PlaceholderConfigurerSupport implements EnvironmentAware {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // 将所有 BeanDefinition 中的 ${...} 占位符替换为实际值
        processProperties(beanFactory, new PropertySourcesPropertyResolver(this.propertySources));
    }
}
```

### BeanDefinitionRegistryPostProcessor（升级版）

```java
// BeanDefinitionRegistryPostProcessor.java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
    /**
     * 在 BFPP 之前调用，可以向注册表追加 BeanDefinition
     * 应用：ConfigurationClassPostProcessor、MapperScannerConfigurer（MyBatis）
     */
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}

// 自定义示例：动态注册 BeanDefinition
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        // 动态注册一个 Bean
        GenericBeanDefinition bd = new GenericBeanDefinition();
        bd.setBeanClass(MyDynamicService.class);
        bd.setScope(BeanDefinition.SCOPE_SINGLETON);
        registry.registerBeanDefinition("myDynamicService", bd);
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // 可以在此修改 BeanDefinition
    }
}
```

---

## 三、BeanPostProcessor 源码

```java
// BeanPostProcessor.java
public interface BeanPostProcessor {
    /**
     * 在 Bean 初始化之前调用（在 @PostConstruct 和 afterPropertiesSet 之前）
     * 返回值可以替换原始 Bean（如返回包装对象）
     */
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    /**
     * 在 Bean 初始化之后调用（在 init-method 之后）
     * ★ AOP 代理在此创建
     * 返回值可以替换原始 Bean（代理对象就是在此替换的）
     */
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

### 自定义 BeanPostProcessor 示例

```java
// 示例：为所有 Service Bean 添加性能日志
@Component
public class PerformanceLogBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean.getClass().isAnnotationPresent(Service.class)) {
            // 创建 JDK 动态代理
            return Proxy.newProxyInstance(
                    bean.getClass().getClassLoader(),
                    bean.getClass().getInterfaces(),
                    (proxy, method, args) -> {
                        long start = System.currentTimeMillis();
                        Object result = method.invoke(bean, args);
                        System.out.printf("[PERF] %s.%s: %dms%n",
                                beanName, method.getName(), System.currentTimeMillis() - start);
                        return result;
                    }
            );
        }
        return bean;
    }
}
```

### InstantiationAwareBeanPostProcessor（进阶）

```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {

    // 实例化前：返回非 null 可短路 Bean 创建（如返回代理对象直接替换）
    @Nullable
    default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        return null;
    }

    // 实例化后：返回 false 可跳过属性填充
    default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        return true;
    }

    // ★ 属性填充：替代传统的 postProcessPropertyValues（Spring 5.1+ 推荐）
    @Nullable
    default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
            throws BeansException {
        return pvs;
    }
}
```

---

## 四、Aware 接口体系源码

```java
// Aware 是一个标记接口，Spring 通过 ApplicationContextAwareProcessor 处理
// 实现对应接口，Bean 就能获取 Spring 容器内部组件

// ① BeanNameAware：获取自身 beanName
public interface BeanNameAware extends Aware {
    void setBeanName(String name);
}

// ② BeanFactoryAware：获取 BeanFactory
public interface BeanFactoryAware extends Aware {
    void setBeanFactory(BeanFactory beanFactory) throws BeansException;
}

// ③ ApplicationContextAware：获取 ApplicationContext（最常用）
public interface ApplicationContextAware extends Aware {
    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}

// ④ EnvironmentAware：获取 Environment（配置属性）
public interface EnvironmentAware extends Aware {
    void setEnvironment(Environment environment);
}

// ⑤ ResourceLoaderAware：获取资源加载器
public interface ResourceLoaderAware extends Aware {
    void setResourceLoader(ResourceLoader resourceLoader);
}

// ⑥ ApplicationEventPublisherAware：获取事件发布器
public interface ApplicationEventPublisherAware extends Aware {
    void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher);
}

// ⑦ MessageSourceAware：获取国际化消息源
public interface MessageSourceAware extends Aware {
    void setMessageSource(MessageSource messageSource);
}

// Aware 接口的处理流程：
// invokeAwareMethods()：处理 BeanNameAware / BeanClassLoaderAware / BeanFactoryAware
// ApplicationContextAwareProcessor.postProcessBeforeInitialization()：
//   处理 EnvironmentAware / ApplicationContextAware 等其余 Aware 接口
```

---

## 五、Spring 事件机制源码

```java
// ===================== 事件发布者 =====================
// ApplicationEventPublisher.java
@FunctionalInterface
public interface ApplicationEventPublisher {
    default void publishEvent(ApplicationEvent event) {
        publishEvent((Object) event);
    }
    void publishEvent(Object event);
}

// AbstractApplicationContext.publishEvent()（实现类）
@Override
public void publishEvent(Object event) {
    publishEvent(event, null);
}

protected void publishEvent(Object event, @Nullable ResolvableType typeHint) {
    // 将普通对象包装成 PayloadApplicationEvent
    ApplicationEvent applicationEvent;
    if (event instanceof ApplicationEvent ae) {
        applicationEvent = ae;
    } else {
        applicationEvent = new PayloadApplicationEvent<>(this, event, typeHint);
    }

    // 如果广播器尚未初始化，先存入 earlyApplicationEvents
    if (this.earlyApplicationEvents != null) {
        this.earlyApplicationEvents.add(applicationEvent);
    } else {
        // ★ 通过 ApplicationEventMulticaster 广播事件
        getApplicationEventMulticaster().multicastEvent(applicationEvent, typeHint);
    }

    // 父容器也广播
    if (this.parent != null) {
        this.parent.publishEvent(event);
    }
}
```

### 事件广播器源码

```java
// SimpleApplicationEventMulticaster.java
@Override
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));

    // 找到所有匹配该事件类型的监听器
    Executor executor = getTaskExecutor();
    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        if (executor != null && listener.supportsAsyncExecution()) {
            // 异步执行监听器（配置了 TaskExecutor 时）
            executor.execute(() -> invokeListener(listener, event));
        } else {
            // 同步执行
            invokeListener(listener, event);
        }
    }
}

@SuppressWarnings({"rawtypes", "unchecked"})
protected void invokeListener(ApplicationListener listener, ApplicationEvent event) {
    ErrorHandler errorHandler = getErrorHandler();
    if (errorHandler != null) {
        try {
            listener.onApplicationEvent(event);
        } catch (Throwable err) {
            errorHandler.handleError(err);
        }
    } else {
        listener.onApplicationEvent(event);
    }
}
```

### @EventListener 注解处理

```java
// EventListenerMethodProcessor.java —— 处理 @EventListener 注解
// 在 afterSingletonsInstantiated() 中扫描所有 Bean 的 @EventListener 方法
// 将每个 @EventListener 方法包装成 ApplicationListenerMethodAdapter 注册到广播器

// 使用示例：
@Component
public class MyEventListener {

    // 监听指定事件
    @EventListener
    public void onUserCreated(UserCreatedEvent event) {
        System.out.println("用户创建：" + event.getUser().getName());
    }

    // 异步监听（需要配置 @EnableAsync）
    @EventListener
    @Async
    public void onOrderCreated(OrderCreatedEvent event) {
        // 异步处理
    }

    // 监听多个事件
    @EventListener({UserCreatedEvent.class, UserUpdatedEvent.class})
    public void onUserChange(ApplicationEvent event) {
        // ...
    }

    // 条件过滤（SpEL）
    @EventListener(condition = "#event.user.vip == true")
    public void onVipUserCreated(UserCreatedEvent event) {
        // 仅 VIP 用户触发
    }

    // 事件链：返回值会作为新事件发布
    @EventListener
    public OrderCreatedEvent onUserCreated2(UserCreatedEvent event) {
        return new OrderCreatedEvent(this, event.getUser());  // 触发 OrderCreatedEvent
    }
}
```

---

## 六、FactoryBean 源码

```java
// FactoryBean.java —— 工厂 Bean，用于创建复杂对象
public interface FactoryBean<T> {
    String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";

    // ★ 返回 Bean 的实例（getBean("myFactoryBean") 实际返回的是这个对象）
    @Nullable
    T getObject() throws Exception;

    // 返回 Bean 的类型
    @Nullable
    Class<?> getObjectType();

    // 是否单例（默认 true）
    default boolean isSingleton() {
        return true;
    }
}

// 使用示例：SqlSessionFactoryBean（MyBatis 的核心 FactoryBean）
public class SqlSessionFactoryBean
        implements FactoryBean<SqlSessionFactory>, InitializingBean, ApplicationListener<ContextRefreshedEvent> {

    @Override
    public SqlSessionFactory getObject() throws Exception {
        if (this.sqlSessionFactory == null) {
            afterPropertiesSet();
        }
        return this.sqlSessionFactory;  // 返回 SqlSessionFactory
    }

    @Override
    public Class<?> getObjectType() {
        return this.sqlSessionFactory == null
                ? SqlSessionFactory.class : this.sqlSessionFactory.getClass();
    }
}

// 获取 FactoryBean 本身（加 & 前缀）
SqlSessionFactoryBean factoryBean = (SqlSessionFactoryBean) context.getBean("&sqlSessionFactory");
// 获取 FactoryBean 产品
SqlSessionFactory sqlSessionFactory = (SqlSessionFactory) context.getBean("sqlSessionFactory");
```

---

## 七、ImportBeanDefinitionRegistrar —— 动态注册 Bean

```java
// 由 @Import 触发，用于动态注册 BeanDefinition
// 典型应用：@MapperScan（MyBatis）、@EnableFeignClients（OpenFeign）

public interface ImportBeanDefinitionRegistrar {
    default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry, BeanNameGenerator importBeanNameGenerator) {
        registerBeanDefinitions(importingClassMetadata, registry);
    }

    default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry) {
    }
}

// 自定义示例：@EnableMyComponents → 扫描并注册指定包下的接口为 Bean
public class MyComponentRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry) {
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
                importingClassMetadata.getAnnotationAttributes(EnableMyComponents.class.getName()));
        String[] basePackages = attributes.getStringArray("basePackages");

        // 扫描包，找到所有 @MyComponent 接口
        ClassPathScanningCandidateComponentProvider scanner = new ClassPathScanningCandidateComponentProvider(false);
        scanner.addIncludeFilter(new AnnotationTypeFilter(MyComponent.class));

        for (String basePackage : basePackages) {
            Set<BeanDefinition> candidates = scanner.findCandidateComponents(basePackage);
            for (BeanDefinition candidate : candidates) {
                // 将每个接口注册为由工厂方法创建的 Bean
                BeanDefinitionBuilder builder = BeanDefinitionBuilder
                        .genericBeanDefinition(MyComponentFactoryBean.class);
                builder.addConstructorArgValue(candidate.getBeanClassName());
                registry.registerBeanDefinition(
                        candidate.getBeanClassName(), builder.getBeanDefinition());
            }
        }
    }
}
```

---

## 八、SmartLifecycle —— Bean 启动/停止控制

```java
// Lifecycle.java —— 控制 Bean 的启动和停止
public interface Lifecycle {
    void start();
    void stop();
    boolean isRunning();
}

// SmartLifecycle —— 增强版，支持自动启动和顺序控制
public interface SmartLifecycle extends Lifecycle, Phased {
    int DEFAULT_PHASE = Integer.MAX_VALUE;

    // 是否自动启动（容器 refresh 完成后自动调用 start()）
    default boolean isAutoStartup() {
        return true;
    }

    // 停止时异步回调
    default void stop(Runnable callback) {
        stop();
        callback.run();
    }

    // 启动/停止顺序（值越小越先启动，越晚停止）
    @Override
    default int getPhase() {
        return DEFAULT_PHASE;
    }
}

// 典型应用：@Scheduled 调度器在此启动，Web 服务器（Tomcat）也通过此机制启动
```

---

## 九、扩展点选择指南

| 需求 | 推荐扩展点 |
|------|----------|
| 动态注册 BeanDefinition | `BeanDefinitionRegistryPostProcessor` / `ImportBeanDefinitionRegistrar` |
| 修改已注册的 BeanDefinition | `BeanFactoryPostProcessor` |
| Bean 初始化前后统一处理 | `BeanPostProcessor` |
| 为 Bean 创建代理 | `BeanPostProcessor.postProcessAfterInitialization` |
| 跳过某 Bean 的属性注入 | `InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation` |
| 创建复杂对象 | `FactoryBean` |
| 监听容器事件/业务事件 | `@EventListener` / `ApplicationListener` |
| 应用启动后执行初始化 | `CommandLineRunner` / `ApplicationRunner` |
| 控制 Bean 的启动停止顺序 | `SmartLifecycle` |
| 获取 Spring 内部组件 | `*Aware` 接口 |

---

## 十、相关源码文件

- [[../01_IoC容器源码/02、ApplicationContext启动流程（refresh方法）]]
- [[../02_Bean生命周期源码/01、Bean实例化源码]]
- [[../02_Bean生命周期源码/02、@Autowired依赖注入源码]]
- [[../03_AOP源码/01、AOP代理创建源码]]
