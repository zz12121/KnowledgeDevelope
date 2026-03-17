# @Transactional 源码解析

> **核心类**：`TransactionInterceptor`、`AbstractPlatformTransactionManager`  
> **核心包**：`org.springframework.transaction.interceptor`

---

## 一、事务整体架构

```
@EnableTransactionManagement
└── 注册 InfrastructureAdvisorAutoProxyCreator（BeanPostProcessor）
    + 注册 BeanFactoryTransactionAttributeSourceAdvisor（Advisor）
        ├── TransactionAttributeSourcePointcut（切点：识别 @Transactional 方法）
        └── TransactionInterceptor（通知：执行事务逻辑）

运行时：
调用 @Transactional 方法
└── AOP 代理拦截
    └── TransactionInterceptor.invoke()
        └── invokeWithinTransaction()
            ├── 获取事务属性（@Transactional 的配置）
            ├── 获取 TransactionManager
            ├── 开启事务（doBegin）
            ├── 调用目标方法
            └── 提交/回滚事务
```

---

## 二、@EnableTransactionManagement 注册流程

```java
// @EnableTransactionManagement → @Import(TransactionManagementConfigurationSelector.class)
public class TransactionManagementConfigurationSelector
        extends AdviceModeImportSelector<EnableTransactionManagement> {

    @Override
    protected String[] selectImports(AdviceMode adviceMode) {
        return switch (adviceMode) {
            case PROXY -> new String[] {
                AutoProxyRegistrar.class.getName(),                      // ← 注册 AutoProxyCreator
                ProxyTransactionManagementConfiguration.class.getName()  // ← 注册事务 Advisor
            };
            case ASPECTJ -> new String[] { ... };
        };
    }
}

// ProxyTransactionManagementConfiguration.java
@Configuration(proxyBeanMethods = false)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

    // 注册 Advisor：TransactionAttributeSourceAdvisor
    @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(
            TransactionAttributeSource transactionAttributeSource,
            TransactionInterceptor transactionInterceptor) {
        BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
        advisor.setTransactionAttributeSource(transactionAttributeSource);
        advisor.setAdvice(transactionInterceptor);    // TransactionInterceptor 就是 Advice
        return advisor;
    }

    // 注册 TransactionAttributeSource（解析 @Transactional 属性）
    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public TransactionAttributeSource transactionAttributeSource() {
        return new AnnotationTransactionAttributeSource();
    }

    // 注册 TransactionInterceptor（核心拦截器）
    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public TransactionInterceptor transactionInterceptor(
            TransactionAttributeSource transactionAttributeSource) {
        TransactionInterceptor interceptor = new TransactionInterceptor();
        interceptor.setTransactionAttributeSource(transactionAttributeSource);
        return interceptor;
    }
}
```

---

## 三、TransactionInterceptor.invoke() —— 事务拦截核心

```java
// TransactionInterceptor.java
@Override
@Nullable
public Object invoke(MethodInvocation invocation) throws Throwable {
    Class<?> targetClass = (invocation.getThis() != null
            ? AopUtils.getTargetClass(invocation.getThis()) : null);

    // ★ 委托给 invokeWithinTransaction
    return invokeWithinTransaction(invocation.getMethod(), targetClass, new CoroutinesInvocationCallback() {
        @Override
        public Object proceedWithInvocation() throws Throwable {
            return invocation.proceed();  // 调用目标方法
        }
        @Override
        public Object getTarget() { return invocation.getThis(); }
        @Override
        public Object[] getArguments() { return invocation.getArguments(); }
    });
}
```

---

## 四、invokeWithinTransaction() —— 事务管理核心逻辑

```java
// TransactionAspectSupport.java
@Nullable
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
        final InvocationCallback invocation) throws Throwable {

    // ① 获取事务属性源（解析 @Transactional）
    TransactionAttributeSource tas = getTransactionAttributeSource();
    final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);

    // ② 获取 TransactionManager（从容器中找合适的 TM）
    final TransactionManager tm = determineTransactionManager(txAttr);

    // ③ 处理响应式事务（略）
    if (this.reactiveAdapterRegistry != null && tm instanceof ReactiveTransactionManager rtm) {
        // ...
    }

    PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
    final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

    if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
        // ④ 普通事务处理（声明式事务）

        // 开启或加入事务
        TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

        Object retVal;
        try {
            // ⑤ 调用目标方法（链式递归执行）
            retVal = invocation.proceedWithInvocation();
        } catch (Throwable ex) {
            // ⑥ 异常处理：判断是否需要回滚
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        } finally {
            // ⑦ 清理事务信息
            cleanupTransactionInfo(txInfo);
        }

        // ⑧ 处理 @Transactional 的 afterCompletion（响应式相关）
        if (retVal != null && txAttr != null) {
            // ...
        }

        // ⑨ 提交事务
        commitTransactionAfterReturning(txInfo);
        return retVal;
    }
    // ...
}
```

---

## 五、createTransactionIfNecessary() —— 事务传播行为实现

```java
// TransactionAspectSupport.java
protected TransactionInfo createTransactionIfNecessary(
        @Nullable PlatformTransactionManager tm, @Nullable TransactionAttribute txAttr,
        final String joinpointIdentification) {

    if (txAttr != null && txAttr.getName() == null) {
        txAttr = new DelegatingTransactionAttribute(txAttr) {
            @Override
            public String getName() { return joinpointIdentification; }
        };
    }

    TransactionStatus status = null;
    if (txAttr != null) {
        if (tm != null) {
            // ★ 获取事务状态（传播行为在这里处理）
            status = tm.getTransaction(txAttr);
        }
    }
    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

---

## 六、AbstractPlatformTransactionManager.getTransaction() —— 传播行为源码

```java
// AbstractPlatformTransactionManager.java
@Override
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException {
    TransactionDefinition def = (definition != null ? definition : TransactionDefinition.withDefaults());

    // ① 获取当前事务对象（数据库连接）
    Object transaction = doGetTransaction();

    boolean debugEnabled = logger.isDebugEnabled();

    // ② 检查是否已有现有事务
    if (isExistingTransaction(transaction)) {
        // 当前线程已有事务 → 根据传播行为处理
        return handleExistingTransaction(def, transaction, debugEnabled);
    }

    // 当前线程没有事务的情况
    if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", def.getTimeout());
    }

    // ③ MANDATORY：必须在现有事务中，没有则报错
    if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException("No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    // ④ REQUIRED/REQUIRES_NEW/NESTED：开启新事务
    else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED
            || def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW
            || def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        // 挂起 null（当前没有事务可挂起）
        SuspendedResourcesHolder suspendedResources = suspend(null);
        try {
            // 开启新事务
            return startTransaction(def, transaction, debugEnabled, suspendedResources);
        } catch (RuntimeException | Error ex) {
            resume(null, suspendedResources);
            throw ex;
        }
    }
    // ⑤ 其他传播行为：以无事务状态运行（SUPPORTS/NOT_SUPPORTED/NEVER）
    else {
        boolean newSynchronization = (getTransactionSynchronizationName() != SYNCHRONIZATION_NEVER);
        return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
    }
}

// 已有事务时的处理
private TransactionStatus handleExistingTransaction(TransactionDefinition definition,
        Object transaction, boolean debugEnabled) throws TransactionException {

    // NEVER：已有事务则报错
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException("Existing transaction found for transaction marked with propagation 'never'");
    }
    // NOT_SUPPORTED：挂起现有事务，以无事务状态运行
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
        Object suspendedResources = suspend(transaction);
        boolean newSynchronization = (getTransactionSynchronizationName() != SYNCHRONIZATION_NEVER);
        return prepareTransactionStatus(definition, null, false, newSynchronization, debugEnabled, suspendedResources);
    }
    // REQUIRES_NEW：挂起现有事务，开启全新事务
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
        SuspendedResourcesHolder suspendedResources = suspend(transaction);
        try {
            return startTransaction(definition, transaction, debugEnabled, suspendedResources);
        } catch (RuntimeException | Error beginEx) {
            resumeAfterBeginException(transaction, suspendedResources, beginEx);
            throw beginEx;
        }
    }
    // NESTED：使用 savepoint 建立嵌套事务（JDBC 不支持则模拟）
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        if (useSavepointForNestedTransaction()) {
            DefaultTransactionStatus status = prepareTransactionStatus(definition, transaction,
                    false, false, debugEnabled, null);
            status.createAndHoldSavepoint();  // ← 创建 JDBC savepoint
            return status;
        }
        // 不支持 savepoint，开启新事务
        return startTransaction(definition, transaction, debugEnabled, null);
    }
    // SUPPORTS/REQUIRED：加入现有事务（直接复用）
    boolean newSynchronization = (getTransactionSynchronizationName() != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}
```

---

## 七、事务传播行为总结

| 传播行为 | 有事务时 | 无事务时 |
|---------|---------|---------|
| `REQUIRED`（默认） | 加入现有事务 | 新建事务 |
| `SUPPORTS` | 加入现有事务 | 以无事务运行 |
| `MANDATORY` | 加入现有事务 | 抛出异常 |
| `REQUIRES_NEW` | 挂起现有，新建事务 | 新建事务 |
| `NOT_SUPPORTED` | 挂起现有，无事务运行 | 以无事务运行 |
| `NEVER` | 抛出异常 | 以无事务运行 |
| `NESTED` | 创建嵌套事务（savepoint） | 新建事务 |

---

## 八、回滚机制源码

```java
// TransactionAspectSupport.java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        if (txInfo.transactionAttribute != null
                && txInfo.transactionAttribute.rollbackOn(ex)) {
            // 满足回滚条件 → 回滚
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
        } else {
            // 不满足回滚条件（如 checked exception 且未配置 rollbackFor）→ 提交
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
        }
    }
}

// RuleBasedTransactionAttribute.rollbackOn() —— 判断是否需要回滚
@Override
public boolean rollbackOn(Throwable ex) {
    RollbackRuleAttribute winner = null;
    int deepest = Integer.MAX_VALUE;

    // 遍历 @Transactional(rollbackFor/noRollbackFor) 配置的规则
    if (this.rollbackRules != null) {
        for (RollbackRuleAttribute rule : this.rollbackRules) {
            int depth = rule.getDepth(ex);
            if (depth >= 0 && depth < deepest) {
                deepest = depth;
                winner = rule;
            }
        }
    }

    // 没有匹配规则时走默认逻辑：RuntimeException 和 Error 回滚
    if (winner == null) {
        return super.rollbackOn(ex);  // ex instanceof RuntimeException || ex instanceof Error
    }

    return !(winner instanceof NoRollbackRuleAttribute);
}
```

---

## 九、事务同步（TransactionSynchronizationManager）

```java
// TransactionSynchronizationManager.java —— 基于 ThreadLocal 管理事务资源
public abstract class TransactionSynchronizationManager {

    // 当前线程的资源绑定（如数据库连接 ConnectionHolder）
    private static final ThreadLocal<Map<Object, Object>> resources =
            new NamedThreadLocal<>("Transactional resources");

    // 当前线程的同步回调（在事务提交/回滚时执行）
    private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
            new NamedThreadLocal<>("Transaction synchronizations");

    // 当前事务的名称
    private static final ThreadLocal<String> currentTransactionName = new NamedThreadLocal<>("Current transaction name");

    // 是否只读
    private static final ThreadLocal<Boolean> currentTransactionReadOnly = new NamedThreadLocal<>("Current transaction read-only status");

    // 事务隔离级别
    private static final ThreadLocal<Integer> currentTransactionIsolationLevel = new NamedThreadLocal<>("Current transaction isolation level");

    // 是否存在活跃事务
    private static final ThreadLocal<Boolean> actualTransactionActive = new NamedThreadLocal<>("Actual transaction active");

    // 绑定数据库连接到当前线程
    public static void bindResource(Object key, Object value) {
        Map<Object, Object> map = resources.get();
        if (map == null) {
            map = new HashMap<>();
            resources.set(map);
        }
        map.put(key, value);
    }

    // 获取当前线程绑定的连接（同一事务内多次 getConnection 拿到同一个）
    public static Object getResource(Object key) {
        Map<Object, Object> map = resources.get();
        if (map == null) {
            return null;
        }
        return map.get(key);
    }
}
```

---

## 十、@Transactional 失效场景（源码分析）

| 失效场景 | 原因（源码层面） |
|---------|----------------|
| 同类内部调用（this.method()） | 走的是原始对象而非代理，不经过 `TransactionInterceptor` |
| private/protected 方法 | CGLIB 无法重写，`TransactionAttributeSourcePointcut` 不匹配 |
| 非 Spring 管理的类 | 不经过 BeanPostProcessor，没有代理 |
| 多线程（子线程开启新事务） | `TransactionSynchronizationManager` 用 ThreadLocal，子线程没有事务上下文 |
| 异常被 catch 吞掉 | `completeTransactionAfterThrowing` 接收不到异常，直接走 commit |
| checked 异常未配置 rollbackFor | `DefaultTransactionAttribute.rollbackOn()` 默认只对 RuntimeException 回滚 |

---

## 十一、相关源码文件

- [[../03_AOP源码/01、AOP代理创建源码]]
- [[02、事务传播行为与隔离级别]]
- [[../07_扩展点源码/03、事件机制源码]]
