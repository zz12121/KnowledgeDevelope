# @Autowired 依赖注入源码解析

> **核心类**：`AutowiredAnnotationBeanPostProcessor`  
> **核心包**：`org.springframework.beans.factory.annotation`

---

## 一、@Autowired 的处理时机

```
Bean 生命周期中 @Autowired 的两个关键节点：

① 收集阶段（applyMergedBeanDefinitionPostProcessors）
   └── AutowiredAnnotationBeanPostProcessor.postProcessMergedBeanDefinition()
       └── 扫描 Bean 的所有字段/方法，找到 @Autowired/@Value/@Inject 并缓存

② 注入阶段（populateBean）
   └── AutowiredAnnotationBeanPostProcessor.postProcessProperties()
       └── InjectionMetadata.inject()
           ├── AutowiredFieldElement.inject()    ← 字段注入
           └── AutowiredMethodElement.inject()   ← 方法注入
```

---

## 二、收集阶段源码

```java
// AutowiredAnnotationBeanPostProcessor.java
@Override
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition,
        Class<?> beanType, String beanName) {
    // 查找并缓存注入元数据
    InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
    metadata.checkConfigMembers(beanDefinition);
}

private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
    String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
    // 从缓存获取（同一个 Bean 不需要重复扫描）
    InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
    if (InjectionMetadata.needsRefresh(metadata, clazz)) {
        synchronized (this.injectionMetadataCache) {
            metadata = this.injectionMetadataCache.get(cacheKey);
            if (InjectionMetadata.needsRefresh(metadata, clazz)) {
                metadata = buildAutowiringMetadata(clazz);  // 真正扫描
                this.injectionMetadataCache.put(cacheKey, metadata);
            }
        }
    }
    return metadata;
}

private InjectionMetadata buildAutowiringMetadata(Class<?> clazz) {
    List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
    Class<?> targetClass = clazz;

    do {
        final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();

        // 扫描所有字段（包括父类的字段）
        ReflectionUtils.doWithLocalFields(targetClass, field -> {
            MergedAnnotation<?> ann = findAutowiredAnnotation(field);
            if (ann != null) {
                if (Modifier.isStatic(field.getModifiers())) {
                    return;  // 忽略静态字段
                }
                boolean required = determineRequiredStatus(ann);
                currElements.add(new AutowiredFieldElement(field, required));
            }
        });

        // 扫描所有方法
        ReflectionUtils.doWithLocalMethods(targetClass, method -> {
            Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
            MergedAnnotation<?> ann = findAutowiredAnnotation(bridgedMethod);
            if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                if (Modifier.isStatic(method.getModifiers())) {
                    return;  // 忽略静态方法
                }
                boolean required = determineRequiredStatus(ann);
                PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                currElements.add(new AutowiredMethodElement(method, required, pd));
            }
        });

        elements.addAll(0, currElements);
        targetClass = targetClass.getSuperclass();
    }
    while (targetClass != null && targetClass != Object.class);

    return InjectionMetadata.forElements(elements, clazz);
}
```

---

## 三、注入阶段源码（字段注入）

```java
// AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement
@Override
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs)
        throws Throwable {

    Field field = (Field) this.member;
    Object value;
    
    // 先查缓存
    if (this.cached) {
        try {
            value = resolveCachedArgument(beanName, this.cachedFieldValue);
        } catch (NoSuchBeanDefinitionException ex) {
            value = resolveFieldValue(field, bean, beanName);
        }
    } else {
        // ★ 核心：解析字段的注入值
        value = resolveFieldValue(field, bean, beanName);
    }

    if (value != null) {
        // 设置字段可访问（处理 private 字段）
        ReflectionUtils.makeAccessible(field);
        // 通过反射给字段赋值
        field.set(bean, value);
    }
}

private Object resolveFieldValue(Field field, Object bean, @Nullable String beanName) {
    // 构建 DependencyDescriptor（封装了字段的类型信息）
    DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
    desc.setContainingClass(bean.getClass());

    Set<String> autowiredBeanNames = new LinkedHashSet<>(2);
    TypeConverter typeConverter = beanFactory.getTypeConverter();

    // ★ 通过 DefaultListableBeanFactory.resolveDependency() 解析依赖
    Object value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);

    // 缓存解析结果
    synchronized (this) {
        if (!this.cached) {
            // ... 缓存逻辑
            this.cached = true;
        }
    }
    return value;
}
```

---

## 四、resolveDependency() —— 依赖解析核心

```java
// DefaultListableBeanFactory.java
@Override
@Nullable
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
        @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

    descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());

    // 特殊类型处理
    if (Optional.class == descriptor.getDependencyType()) {
        return createOptionalDependency(descriptor, requestingBeanName);
    } else if (ObjectFactory.class == descriptor.getDependencyType()
            || ObjectProvider.class == descriptor.getDependencyType()) {
        return new DependencyObjectProvider(descriptor, requestingBeanName);
    } else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
        return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
    } else {
        // 懒加载代理（@Lazy）
        Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
                descriptor, requestingBeanName);
        if (result == null) {
            // ★ 正常注入：按类型查找 Bean
            result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
        }
        return result;
    }
}

@Nullable
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
        @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

    // ① 处理 @Value
    Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
    if (value != null) {
        if (value instanceof String strValue) {
            // 解析 ${...} 占位符和 #{...} SpEL 表达式
            String resolvedValue = resolveEmbeddedValue(strValue);
            BeanDefinition bd = beanName != null && containsBean(beanName)
                    ? getMergedBeanDefinition(beanName) : null;
            value = evaluateBeanDefinitionString(resolvedValue, bd);
        }
        TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
        return converter.convertIfNecessary(value, type, descriptor.getTypeDescriptor());
    }

    // ② 处理集合类型（List/Map/Set/数组）
    Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
    if (multipleBeans != null) {
        return multipleBeans;
    }

    // ③ 按类型查找候选 Bean（findAutowireCandidates）
    Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
    if (matchingBeans.isEmpty()) {
        if (isRequired(descriptor)) {
            raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
        }
        return null;
    }

    String autowiredBeanName;
    Object instanceCandidate;

    if (matchingBeans.size() > 1) {
        // ④ 多个候选者时：@Primary → @Priority → 按字段名匹配
        autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
        if (autowiredBeanName == null) {
            if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
                return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
            }
            return null;
        }
        instanceCandidate = matchingBeans.get(autowiredBeanName);
    } else {
        Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
        autowiredBeanName = entry.getKey();
        instanceCandidate = entry.getValue();
    }

    // ⑤ 将候选 beanName 加入 autowiredBeanNames（用于注册依赖关系）
    if (autowiredBeanNames != null) {
        autowiredBeanNames.add(autowiredBeanName);
    }

    // ⑥ 如果候选是 Class 对象（延迟加载），调用 getBean 实例化
    if (instanceCandidate instanceof Class) {
        instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
    }

    return instanceCandidate;
}
```

---

## 五、@Autowired 注入优先级规则

```
按类型查找到多个候选 Bean 时的优先级：

① @Primary 注解 → 优先选择
② @Priority 注解 → 值越小优先级越高
③ 字段名/参数名与 beanName 匹配 → 按名称精确匹配
④ 都不满足 → 抛 NoUniqueBeanDefinitionException
```

---

## 六、@Qualifier 源码

```java
// QualifierAnnotationAutowireCandidateResolver.java
@Override
public boolean isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
    // 先判断基本条件（非懒加载等）
    if (!super.isAutowireCandidate(bdHolder, descriptor)) {
        return false;
    }
    // 检查 @Qualifier 是否匹配
    return checkQualifiers(bdHolder, descriptor.getAnnotations());
}

protected boolean checkQualifiers(BeanDefinitionHolder bdHolder, Annotation[] annotationsToSearch) {
    for (Annotation annotation : annotationsToSearch) {
        Class<? extends Annotation> type = annotation.annotationType();
        if (isQualifier(type)) {
            // 比较 @Qualifier 的 value 和 BeanDefinition 上的 qualifier
            if (!checkQualifier(bdHolder, annotation, typeConverter)) {
                return false;
            }
        }
    }
    return true;
}
```

---

## 七、常见问题与源码对应

| 问题 | 源码位置 |
|------|---------|
| @Autowired 是如何找到对应 Bean 的？ | `doResolveDependency` → `findAutowireCandidates` |
| 多个同类型 Bean 如何选择？ | `determineAutowireCandidate`：@Primary → @Priority → 按名 |
| @Autowired(required=false) 如何处理找不到的情况？ | `isRequired` 判断，找不到返回 null |
| @Value 的 ${} 和 #{} 有何区别？ | `${}` 走 `resolveEmbeddedValue`（PropertyPlaceholder），`#{}` 走 SpEL |
| @Resource 和 @Autowired 的区别？ | @Resource 由 `CommonAnnotationBeanPostProcessor` 处理，先按名找 |

---

## 八、相关源码文件

- [[01、Bean实例化源码]]
- [[../07_扩展点源码/02、BeanPostProcessor源码]]
- [[../01_IoC容器源码/01、BeanFactory体系结构]]
