# Spring 事务管理

## 1 事务与 ACID 特性

**事务**是数据库操作的最小工作单元，包含一个或多个 SQL 语句，这些语句要么全部成功，要么全部回滚。经典例子：银行转账中"A 扣款"和"B 加款"必须作为不可分割的整体。

**ACID 四大特性**：

**原子性（Atomicity）**：事务所有操作要么全部提交，要么全部回滚。底层通过 **Undo Log** 实现——执行过程中记录数据修改前的镜像，回滚时用 Undo Log 恢复。

**一致性（Consistency）**：事务执行前后，数据库必须从一种一致性状态变到另一种一致性状态。由应用层逻辑正确性和数据库原子性/隔离性/持久性共同保证。

**隔离性（Isolation）**：并发事务相互隔离，一个事务的中间状态对其他事务不可见。通过锁机制和 MVCC（多版本并发控制）实现。

**持久性（Durability）**：事务提交后的修改是永久的，系统故障不会丢失。底层通过 **Redo Log** 实现——事务提交前先将变更写入 Redo Log，故障恢复时重放。

## 2 Spring 事务管理架构

Spring 通过 `PlatformTransactionManager` 接口统一管理事务，提供两种管理方式。

**核心组件**：
- `PlatformTransactionManager`：核心接口，定义 `getTransaction()` / `commit()` / `rollback()`
- `TransactionDefinition`：描述事务属性（隔离级别、传播行为、超时、只读性）
- `TransactionStatus`：描述事务当前状态（是否新事务、是否有回滚标记）

**常用实现**：
- `DataSourceTransactionManager`：JDBC / MyBatis 场景
- `JpaTransactionManager`：JPA / Hibernate 场景
- `JtaTransactionManager`：分布式事务场景

**声明式事务的底层机制**：`TransactionInterceptor`（一个 AOP MethodInterceptor）拦截 `@Transactional` 方法，委托给 `TransactionAspectSupport.invokeWithinTransaction()` 方法，由该方法驱动 `PlatformTransactionManager` 完成事务生命周期管理。

## 3 编程式事务 vs 声明式事务

**编程式事务**：在代码中显式控制事务边界，侵入性强，但可以精确控制：

```java
@Service
public class AccountService {
    @Autowired
    private TransactionTemplate transactionTemplate;

    public void transfer(String from, String to, Double amount) {
        transactionTemplate.execute(status -> {
            accountDao.deduct(from, amount);
            accountDao.add(to, amount);
            return null;
        });
    }
}
```

**声明式事务**：通过注解声明，AOP 自动处理，业务代码与事务逻辑完全分离，是现代 Spring 开发的标准选择：

```java
@Service
public class AccountService {
    @Transactional
    public void transfer(String from, String to, Double amount) {
        accountDao.deduct(from, amount);
        accountDao.add(to, amount);
    }
}
```

**选择原则**：优先用声明式（`@Transactional`），只有在需要细粒度控制事务边界时才考虑编程式。

## 4 @Transactional 详解

**完整参数**：

```java
@Transactional(
    value = "transactionManager",        // 指定事务管理器 Bean 名称（多数据源场景）
    propagation = Propagation.REQUIRED,  // 传播行为，默认 REQUIRED
    isolation = Isolation.DEFAULT,       // 隔离级别，默认跟随数据库
    timeout = 30,                        // 超时时间（秒），-1 表示不限
    readOnly = false,                    // 只读事务可优化查询性能
    rollbackFor = {Exception.class},     // 触发回滚的异常类型
    noRollbackFor = {BusinessException.class} // 不触发回滚的异常类型
)
```

**关键注意事项**：

**代理限制**：`@Transactional` 基于 AOP 代理实现，因此只对 `public` 方法有效；同一类内部调用（自调用）会绕过代理，导致注解失效。

**默认回滚规则**：只有 `RuntimeException` 及其子类、`Error` 才触发回滚；受检异常（checked exception）默认**不回滚**。建议关键业务都加 `rollbackFor = Exception.class`。

**只读事务**：`readOnly = true` 不是"强制只读"，而是给数据库和 ORM 框架的优化提示，可以跳过一些写相关的开销。

## 5 事务传播行为（7 种）

传播行为定义了**多个事务方法相互调用时，事务如何传播**的策略，是 Spring 事务最复杂的部分：

**REQUIRED（默认）**：有事务就加入，没有就新建。99% 的业务方法用这个就够了。

**REQUIRES_NEW**：无论当前是否有事务，都创建一个全新的独立事务，当前事务（如果有）会被挂起。新事务和外层事务互相不影响——新事务回滚不影响外层，外层回滚也不影响已提交的新事务。**典型场景：记录操作日志，无论主业务成败都要保存日志。**

**NESTED**：如果有外层事务，则在**嵌套事务**（保存点机制）中执行。嵌套事务回滚只回滚到保存点，不影响外层事务；但外层事务回滚会导致嵌套事务也一起回滚。**注意与 REQUIRES_NEW 的区别**：NESTED 共享外层事务连接，REQUIRES_NEW 是独立连接。

**SUPPORTS**：有事务就加入，没有就以非事务方式执行。适合不强依赖事务的查询方法。

**NOT_SUPPORTED**：以非事务方式执行，有事务则挂起。适合不支持事务的操作。

**MANDATORY**：必须在现有事务中执行，没有外层事务则抛异常。适合"必须由调用方保证事务"的方法。

**NEVER**：不能在事务中执行，有事务则抛异常。

```java
@Service
public class LogService {
    // 独立事务记录日志，不受主业务事务影响
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void auditLog(String message) {
        logDao.save(message);
    }
}
```

## 6 事务隔离级别

隔离级别解决并发事务间的数据可见性问题：

**三类并发问题**：
- **脏读**：读到其他事务未提交的数据
- **不可重复读**：同一事务内两次读取同一行，值不同（被其他事务修改并提交）
- **幻读**：同一事务内两次查询返回的行数不同（被其他事务插入/删除）

**五个隔离级别**：

`DEFAULT`：跟随数据库默认（MySQL InnoDB 默认 REPEATABLE_READ）。

`READ_UNCOMMITTED`：可以读到未提交数据，脏读/不可重复读/幻读都可能发生，几乎不用。

`READ_COMMITTED`：只能读已提交数据，解决脏读，不可重复读和幻读仍可能。Oracle 默认。

`REPEATABLE_READ`：同一事务内多次读取结果一致，解决脏读和不可重复读。MySQL InnoDB 通过 MVCC 在此级别下还能防止幻读。

`SERIALIZABLE`：最高级别，完全串行化，性能最低，极少使用。

**实际选择**：Spring 默认 `DEFAULT`（跟随数据库），MySQL InnoDB 的 REPEATABLE_READ 在多数场景下已足够。

## 7 @Transactional 的常见失效场景

**自调用（最常见）**：同类方法 A 直接调用方法 B 上的 `@Transactional`，因为调用的是 `this.methodB()` 而不是代理对象，导致事务失效：

```java
// 失效！
@Service
public class UserService {
    public void updateUser(User user) {
        saveAuditLog(user);  // 直接调用，绕过代理
    }

    @Transactional
    public void saveAuditLog(User user) { }
}

// 解决方案 1：注入自身代理
@Service
public class UserService {
    @Autowired
    private UserService self;  // 注入自己的代理

    public void updateUser(User user) {
        self.saveAuditLog(user);  // 通过代理调用
    }
}
```

**方法非 public**：AOP 代理无法拦截非 public 方法。

**异常被内部捕获**：事务回滚依赖异常抛出到拦截器层，catch 住不抛就不会触发回滚：

```java
@Transactional
public void placeOrder() {
    try {
        // ...
    } catch (Exception e) {
        log.error("error", e);  // 异常被吃掉，不会回滚！
    }
}
```

**其他场景**：数据库引擎不支持事务（如 MyISAM）；多数据源未指定正确的事务管理器；`@Transactional` 加在 `final` 方法上（CGLIB 无法覆写）。

## 8 分布式事务方案

分布式事务涉及多个独立资源（不同数据库、消息队列等）的一致性保证：

**2PC（两阶段提交，XA 协议）**：强一致性，但性能差，协调者单点故障，事务长时间锁定资源，不适合高并发。

**TCC（Try-Confirm-Cancel）**：业务层实现补偿，分三个阶段：Try 预留资源 → Confirm 确认操作 → Cancel 撤销预留。性能好，但业务代码侵入性强，每个操作都要实现三个方法。

**消息队列 + 本地事件表（最终一致性）**：将业务操作和事件记录放在同一本地事务中，再由定时任务或消息中间件异步推送事件，确保最终一致。实现简单，适合对实时一致性要求不高的场景。

**Seata**：阿里开源的分布式事务框架，支持 AT/TCC/SAGA/XA 四种模式，AT 模式对业务代码零侵入，是目前主流选型。

## 9 事务源码原理（深度）

Spring 事务实现是 AOP + 代理模式 + 平台抽象的综合体。

**代理创建阶段**：`InfrastructureAdvisorAutoProxyCreator` 作为 `BeanPostProcessor`，在 Bean 初始化后检查是否匹配事务切面（`BeanFactoryTransactionAttributeSourceAdvisor`），匹配则创建代理。

**执行阶段的核心流程**（`invokeWithinTransaction` 方法）：
1. 通过 `TransactionAttributeSource` 解析 `@Transactional` 注解元数据
2. 确定使用哪个 `PlatformTransactionManager`（支持多数据源场景）
3. 调用 `getTransaction()`，根据传播行为决定创建/加入/挂起事务
4. 通过 `ThreadLocal`（`TransactionSynchronizationManager`）将数据库连接绑定到当前线程，保证同一事务内多个 DAO 操作使用同一连接
5. 执行业务逻辑
6. 成功则 `commit()`，异常则根据回滚规则决定 `rollback()`

**ThreadLocal 资源绑定**是保证事务原子性的关键——同一线程的多个 DAO 操作共享同一个 `Connection`，才能在同一事务中。

---

相关面试题 → [[../../10_DevelopLanguage/005_Spring/01_SpringSubject/04、Spring 事务管理|04、Spring事务管理]]
