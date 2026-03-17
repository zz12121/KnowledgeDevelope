# BeanFactory 体系结构源码解析

> **源码版本**：Spring Framework 6.x（兼容 5.x）  
> **核心包**：`org.springframework.beans.factory`

---

## 一、BeanFactory 接口体系总览

```
BeanFactory（顶层接口）
├── HierarchicalBeanFactory        ── 支持父子容器
│   └── ConfigurableBeanFactory    ── 可配置（设置ClassLoader、BeanPostProcessor等）
│       └── ConfigurableListableBeanFactory  ── 可列举+可配置（最完整的工厂接口）
├── ListableBeanFactory            ── 可列举所有Bean（按类型/名称批量获取）
└── AutowireCapableBeanFactory     ── 支持自动注入（创建Bean并完成依赖注入）

ApplicationContext（继承以上全部接口）
├── ClassPathXmlApplicationContext
├── AnnotationConfigApplicationContext   ← 最常用
├── GenericWebApplicationContext
└── ...
```

---

## 二、BeanFactory 核心接口源码

```java
// ===================== BeanFactory.java =====================
public interface BeanFactory {

    // FactoryBean 的前缀符号，用于区分 FactoryBean 本身与其产品
    String FACTORY_BEAN_PREFIX = "&";

    // 核心方法：按名称获取 Bean（最终都会走到 doGetBean）
    Object getBean(String name) throws BeansException;
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException;
    Object getBean(String name, Object... args) throws BeansException;

    // 获取 ObjectProvider（延迟/安全获取 Bean，Spring 5+ 推荐）
    <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

    // 判断容器中是否存在 Bean
    boolean containsBean(String name);

    // 判断是否单例/原型
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

    // 判断 Bean 类型是否匹配
    boolean isTypeMatch(String name, ResolvableType typeToMatch);
    boolean isTypeMatch(String name, Class<?> typeToMatch);

    // 获取 Bean 的类型
    @Nullable
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;

    // 获取 Bean 的所有别名
    String[] getAliases(String name);
}
```

---

## 三、DefaultListableBeanFactory —— 最核心的实现类

`DefaultListableBeanFactory` 是整个 Spring 容器的"发动机"，几乎所有容器最终都委托给它。

### 3.1 类继承体系

```
DefaultListableBeanFactory
    extends AbstractAutowireCapableBeanFactory  ← 负责 Bean 实例化和自动注入
        extends AbstractBeanFactory             ← 负责 getBean 主流程
            extends FactoryBeanRegistrySupport  ← 处理 FactoryBean
                extends DefaultSingletonBeanRegistry  ← 三级缓存（解决循环依赖）
    implements ConfigurableListableBeanFactory  ← 最完整的工厂接口
    implements BeanDefinitionRegistry          ← 负责注册 BeanDefinition
```

### 3.2 核心字段

```java
public class DefaultListableBeanFactory
        extends AbstractAutowireCapableBeanFactory
        implements ConfigurableListableBeanFactory, BeanDefinitionRegistry {

    /** BeanDefinition 注册表：beanName → BeanDefinition */
    private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);

    /** 按注册顺序保存的 beanName 列表（保证有序） */
    private volatile List<String> beanDefinitionNames = new ArrayList<>(256);

    /** 按类型缓存：type → beanName[]（提升按类型查找性能） */
    private final Map<Class<?>, String[]> allBeanNamesByType = new ConcurrentHashMap<>(64);

    /** 单例 beanName → type（冻结后的类型缓存） */
    private final Map<Class<?>, String[]> singletonBeanNamesByType = new ConcurrentHashMap<>(64);

    /** 手动注册的单例（beanName → instance）*/
    private final Map<String, Object> manualSingletonNames = new LinkedHashMap<>(16);

    /** 是否允许同名 BeanDefinition 覆盖 */
    private boolean allowBeanDefinitionOverriding = true;

    /** 是否允许循环依赖 */
    private boolean allowCircularReferences = true;
}
```

---

## 四、AbstractBeanFactory.getBean() 主流程

```java
// ===================== AbstractBeanFactory.java =====================
@Override
public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
    return doGetBean(name, requiredType, null, false);
}

protected <T> T doGetBean(
        String name, @Nullable Class<T> requiredType,
        @Nullable Object[] args, boolean typeCheckOnly) throws BeansException {

    // ① 规范化 beanName（处理别名、去掉 FactoryBean 前缀 "&"）
    String beanName = transformedBeanName(name);
    Object beanInstance;

    // ② 先从三级缓存查单例（解决循环依赖的关键）
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // 如果是 FactoryBean，需要进一步调用 getObject()
        beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        // ③ 如果当前线程正在创建该 prototype Bean → 循环依赖，直接报错
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // ④ 检查父容器（如果本容器没有，委托给父容器）
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            return parentBeanFactory.getBean(...);
        }

        // ⑤ 获取 BeanDefinition（合并父子 BeanDefinition）
        RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

        // ⑥ 先初始化当前 Bean 依赖的其他 Bean（dependsOn）
        String[] dependsOn = mbd.getDependsOn();
        if (dependsOn != null) {
            for (String dep : dependsOn) {
                registerDependentBean(dep, beanName);
                getBean(dep);  // 先创建依赖 Bean
            }
        }

        // ⑦ 根据 scope 分流创建
        if (mbd.isSingleton()) {
            // 单例：通过 getSingleton(beanName, ObjectFactory) 确保全局唯一
            sharedInstance = getSingleton(beanName, () -> {
                return createBean(beanName, mbd, args);  // 核心创建逻辑
            });
            beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        } else if (mbd.isPrototype()) {
            // 原型：每次都新建
            Object prototypeInstance = createBean(beanName, mbd, args);
            beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
        } else {
            // 自定义 Scope（如 request、session）
            String scopeName = mbd.getScope();
            Scope scope = this.scopes.get(scopeName);
            Object scopedInstance = scope.get(beanName, () -> createBean(beanName, mbd, args));
            beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
        }
    }

    // ⑧ 类型检查/适配
    return adaptBeanInstance(name, beanInstance, requiredType);
}
```

---

## 五、三级缓存（DefaultSingletonBeanRegistry）

三级缓存是 Spring 解决**单例 Bean 循环依赖**的核心机制：

```java
public class DefaultSingletonBeanRegistry {

    /** 一级缓存：完整的单例 Bean（成品）*/
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    /** 二级缓存：早期暴露的 Bean（已实例化但未完成属性填充的半成品）*/
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

    /** 三级缓存：Bean 工厂（存放 ObjectFactory，用于生成早期引用）*/
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

    /** 正在创建中的 Bean 集合 */
    private final Set<String> singletonsCurrentlyInCreation = Collections.newSetFromMap(new ConcurrentHashMap<>(16));


    @Nullable
    protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        // ① 先查一级缓存（最快路径）
        Object singletonObject = this.singletonObjects.get(beanName);

        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            // ② 查二级缓存（已暴露的早期引用）
            singletonObject = this.earlySingletonObjects.get(beanName);

            if (singletonObject == null && allowEarlyReference) {
                synchronized (this.singletonObjects) {
                    // ③ 查三级缓存，调用 ObjectFactory.getObject() 获取早期引用
                    //    （此处会触发 AOP 代理的提前创建）
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        singletonObject = singletonFactory.getObject();
                        // 升级到二级缓存，避免重复调用 AOP 代理
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return singletonObject;
    }
}
```

### 循环依赖解决示意

```
A 依赖 B，B 依赖 A

① 创建 A：实例化 → 放入三级缓存（ObjectFactory<A>）→ 填充属性（需要 B）
② 创建 B：实例化 → 放入三级缓存（ObjectFactory<B>）→ 填充属性（需要 A）
③ 获取 A：命中三级缓存 → 调用 ObjectFactory.getObject() 得到早期 A 引用
           → 放入二级缓存
④ B 完成初始化 → 放入一级缓存
⑤ A 完成属性填充 → 完成初始化 → 放入一级缓存
```

> ⚠️ **构造器注入无法解决循环依赖**：因为实例化阶段就需要依赖方已存在，此时尚未放入任何缓存。

---

## 六、BeanDefinitionRegistry —— Bean 注册

```java
public interface BeanDefinitionRegistry extends AliasRegistry {

    // 注册 BeanDefinition
    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException;

    // 移除 BeanDefinition
    void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

    // 获取 BeanDefinition
    BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

    // 是否包含
    boolean containsBeanDefinition(String beanName);

    // 获取所有注册的 beanName
    String[] getBeanDefinitionNames();

    int getBeanDefinitionCount();
}
```

---

## 七、常见面试问题

| 问题 | 关键源码位置 |
|------|------------|
| BeanFactory 和 ApplicationContext 的区别？ | `AbstractApplicationContext#refresh()` 额外做了很多初始化工作 |
| 三级缓存为什么需要第三级？ | 为了让 AOP 代理对象能在循环依赖中被提前创建，且只创建一次 |
| 为何不直接用二级缓存解决循环依赖？ | AOP 代理需要 `ObjectFactory` 延迟创建，直接存对象则无法延迟 |
| getBean 线程安全吗？ | 单例加了 `synchronized`，多级缓存用 `ConcurrentHashMap`，整体安全 |

---

## 八、相关源码文件

- [[02、ApplicationContext启动流程（refresh方法）]]
- [[../02_Bean生命周期源码/01、Bean实例化源码]]
- [[../07_扩展点源码/01、BeanFactoryPostProcessor源码]]
