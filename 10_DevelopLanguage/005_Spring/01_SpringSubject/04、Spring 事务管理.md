###### 1. 什么是事务？事务的 ACID 特性是什么？

事务是数据库操作的最小工作单元，里面的一组操作要么全部成功提交，要么全部失败回滚，绝对不允许只执行一半。银行转账是最经典的例子：A 扣钱和 B 加钱必须绑在一起，不能出现 A 扣了钱、B 却没收到的情况。

**ACID 四个特性：**

**原子性（Atomicity）**：事务里的操作是一个不可分割的整体，要么全成功，要么全回滚。底层通过 **Undo Log** 实现，事务执行时记录修改前的数据镜像，回滚时用 Undo Log 恢复。

**一致性（Consistency）**：事务执行前后，数据库必须从一种合法状态变换到另一种合法状态，不能出现数据矛盾（比如总金额不能变）。一致性是目标，原子性/隔离性/持久性是手段。

**隔离性（Isolation）**：并发事务之间相互隔离，一个事务的中间状态对其他事务不可见。通过锁机制和 MVCC（多版本并发控制）实现，不同隔离级别对应不同的隔离策略。

**持久性（Durability）**：事务一旦提交，数据修改就是永久的，即使系统崩溃也不会丢。底层通过 **Redo Log** 实现，事务提交前先把变更写入 Redo Log，崩溃恢复时通过 Redo Log 重放数据。

📖 [[../../../24_SpringKnowledge/03_事务管理/01、Spring事务管理#一、ACID 特性回顾]]

---

###### 2. Spring 如何管理事务？

Spring 通过 **`PlatformTransactionManager`** 统一管理事务，提供了两种管理方式：声明式事务和编程式事务。

**核心架构三件套：**

- **`PlatformTransactionManager`**：事务管理器核心接口，定义 `getTransaction()`、`commit()`、`rollback()` 三个操作。常用实现有 `DataSourceTransactionManager`（JDBC/MyBatis）和 `JpaTransactionManager`（JPA/Hibernate）
- **`TransactionDefinition`**：定义事务属性，包括传播行为、隔离级别、超时时间、是否只读
- **`TransactionStatus`**：描述当前事务状态，是否是新事务、是否已标记回滚等

**声明式事务的执行路径：** 调用 `@Transactional` 方法 → 进入 AOP 代理 → `TransactionInterceptor` 拦截 → 调用 `TransactionAspectSupport.invokeWithinTransaction()` → 获取事务管理器 → 开启/加入事务 → 执行业务方法 → 提交或回滚。

**线程绑定机制**：Spring 通过 `TransactionSynchronizationManager` 用 **ThreadLocal** 把数据库连接绑定到当前线程，保证同一线程的多个 DAO 操作使用同一个数据库连接，这是事务原子性的物理基础。

📖 [[../../../24_SpringKnowledge/03_事务管理/01、Spring事务管理#二、Spring 事务管理架构]]

---

###### 3. 编程式事务和声明式事务的区别是什么？

**编程式事务**：代码里显式控制事务边界，通过 `TransactionTemplate` 或 `PlatformTransactionManager` 手写开启、提交、回滚逻辑：

```java
@Service
public class AccountService {
    @Autowired
    private TransactionTemplate transactionTemplate;

    public void transfer(String out, String in, Double money) {
        transactionTemplate.execute(status -> {
            accountDao.outMoney(out, money);
            accountDao.inMoney(in, money);
            return null;
        });
    }
}
```

**声明式事务**：用注解声明，Spring 通过 AOP 自动处理，业务代码不沾事务代码：

```java
@Service
public class AccountService {
    @Autowired
    private AccountDao accountDao;

    @Transactional
    public void transfer(String out, String in, Double money) {
        accountDao.outMoney(out, money);
        accountDao.inMoney(in, money);
    }
}
```

**如何选择：** 绝大多数场景用声明式，简单干净。只有需要在方法内部动态控制事务边界（比如循环处理时部分提交）才用编程式。

📖 [[../../../24_SpringKnowledge/03_事务管理/01、Spring事务管理#三、声明式 vs 编程式事务]]

---

###### 4. @Transactional 注解的使用方法和注意事项是什么？

`@Transactional` 可以加在**类**（类里所有 public 方法都开启事务）或**方法**（只对当前方法）上，方法级别优先级更高。

**完整参数：**

```java
@Transactional(
    value = "transactionManager",            // 指定事务管理器（多数据源时需要）
    propagation = Propagation.REQUIRED,      // 传播行为，默认 REQUIRED
    isolation = Isolation.DEFAULT,           // 隔离级别，默认用数据库的
    timeout = 30,                            // 超时时间（秒），超时自动回滚
    readOnly = false,                        // 只读事务，可以优化查询性能
    rollbackFor = {Exception.class},         // 指定哪些异常触发回滚
    noRollbackFor = {BusinessException.class} // 指定哪些异常不触发回滚
)
```

**四大注意事项：**

1. **只对 public 方法有效**：AOP 代理只能拦截 public 方法，protected/private 方法上加了也没用

2. **自调用不生效**：同一个类里方法 A 调用方法 B，调用不经过代理，B 上的 `@Transactional` 无效

3. **默认只对 RuntimeException 回滚**：受检异常（`IOException`、`SQLException` 等）默认不回滚，生产代码建议加 `rollbackFor = Exception.class`

4. **异常不能被吞掉**：如果 catch 了异常但没有重新抛出，事务拦截器感知不到异常，不会触发回滚

📖 [[../../../24_SpringKnowledge/03_事务管理/01、Spring事务管理#四、@Transactional 参数详解]]

---

###### 5. Spring 事务的传播行为有哪些？

传播行为定义了"事务方法被另一个事务方法调用时，事务如何传播"。Spring 定义了 7 种：

**REQUIRED（默认）**：有事务就加入，没有就新建。最常用，覆盖 90% 的场景。

**REQUIRES_NEW**：不管外层有没有事务，都挂起外层事务，新建一个独立事务。用于需要独立提交的操作，比如日志记录、审计，即使主业务回滚，日志也要保存。

**NESTED**：如果当前有事务，在嵌套事务（保存点）里执行；没有就新建。回滚时只回滚到保存点，不影响外层事务。比 REQUIRES_NEW 更轻量，外层能感知嵌套事务的回滚。

**SUPPORTS**：有事务就加入，没有就以非事务方式运行。适合只读查询方法。

**MANDATORY**：有事务就加入，没有就抛异常。适合必须在事务内调用的方法。

**NOT_SUPPORTED**：以非事务方式运行，有事务就挂起它。

**NEVER**：非事务运行，有事务就抛异常。

**REQUIRES_NEW vs NESTED 的关键区别：**

- `REQUIRES_NEW` 创建完全独立的新事务，外层回滚不影响新事务
- `NESTED` 在外层事务的保存点里执行，外层回滚会把嵌套事务一起回滚

```java
@Service
public class OrderService {
    @Transactional(propagation = Propagation.REQUIRED)
    public void placeOrder(Order order) {
        orderDao.save(order);
        try {
            logService.auditLog("订单创建: " + order.getId()); // 独立事务
        } catch (Exception e) {
            // 日志失败不影响主业务
        }
    }
}

@Service
public class LogService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void auditLog(String message) {
        logDao.save(new AuditLog(message)); // 独立提交，主业务回滚也会保存
    }
}
```

📖 [[../../../24_SpringKnowledge/03_事务管理/01、Spring事务管理#五、7 种传播行为]]

---

###### 6. Spring 事务的隔离级别有哪些？

隔离级别解决并发事务带来的三种问题：脏读、不可重复读、幻读。级别越高，并发性能越低。

**脏读**：读到其他事务未提交的数据，对方回滚后你读的数据就是"脏"的

**不可重复读**：同一事务内两次读同一行数据，结果不一样（被其他事务修改了）

**幻读**：同一事务内两次查询，返回的行数不一样（被其他事务插入/删除了行）

**Spring 支持 5 种隔离级别：**

- **DEFAULT**：使用数据库默认的，MySQL InnoDB 默认是 REPEATABLE_READ
- **READ_UNCOMMITTED**：最低级别，三种问题都可能发生，几乎不用
- **READ_COMMITTED**：阻止脏读，Oracle/SQL Server 的默认级别
- **REPEATABLE_READ**：阻止脏读和不可重复读，MySQL InnoDB 的默认级别（MVCC 也基本解决了幻读）
- **SERIALIZABLE**：最高级别，三种问题全防，但串行执行，性能最差

实际项目大多数场景用数据库默认值（DEFAULT）就够了，特殊需求再单独配置。

📖 [[../../../24_SpringKnowledge/03_事务管理/01、Spring事务管理#六、5 种隔离级别]]

---

###### 7. 什么情况下 @Transactional 会失效？

`@Transactional` 失效是面试高频题，也是生产环境常见的坑：

**1. 自调用（最常见）**：同类里的方法 A 调方法 B，绕过了代理，B 上的注解不生效：

```java
@Service
public class UserService {
    public void updateUser(User user) {
        saveAuditLog(user); // 不走代理！@Transactional 无效
    }

    @Transactional
    public void saveAuditLog(User user) { ... }
}
```

解决方式：把 `saveAuditLog` 提到独立的 Bean 里，或者通过 `AopContext.currentProxy()` 拿到代理对象来调用。

**2. 方法非 public**：protected/private 方法上加了也不生效，AOP 代理无法拦截。

**3. 异常被吞掉**：catch 了异常但没有重新 throw，拦截器感知不到异常，不会触发回滚：

```java
@Transactional
public void placeOrder() {
    try {
        // 业务逻辑
    } catch (Exception e) {
        log.error("出错了", e); // 异常被吞掉，不会回滚！
    }
}
```

**4. 异常类型不匹配**：默认只对 `RuntimeException` 和 `Error` 回滚，`IOException` 这类受检异常不会触发回滚。要加 `rollbackFor = Exception.class`。

**5. 数据库引擎不支持事务**：MySQL 的 MyISAM 引擎不支持事务，只有 InnoDB 支持。

**6. 多数据源未指定事务管理器**：多数据源场景下，没有指定 `value = "xxTransactionManager"`，可能用错了事务管理器。

📖 [[../../../24_SpringKnowledge/03_事务管理/01、Spring事务管理#八、事务失效的 7 种场景]]

---

###### 8. 如何实现分布式事务？

分布式事务涉及多个服务/数据库的数据一致性，比单机事务复杂得多。常见方案：

**XA/2PC（强一致性）**：基于两阶段提交协议，Prepare 阶段所有参与者预提交，Commit 阶段统一提交。强一致，但性能差（锁持有时间长），协调者单点故障风险大。Seata 的 AT 模式就是基于类似思想但做了优化。

**TCC（Try-Confirm-Cancel）**：把每个操作拆成三步：Try（预留资源）、Confirm（确认提交）、Cancel（回滚释放）。代码侵入性强，但性能好，适合核心交易场景：

```java
// Try 阶段：冻结金额（不是真正扣款）
accountService.freezeAmount(userId, amount);
// Confirm 阶段：确认扣款
accountService.confirmFreeze(userId, amount);
// Cancel 阶段：解冻（失败时回滚）
accountService.cancelFreeze(userId, amount);
```

**消息队列 + 本地事务（最终一致性）**：把业务操作和消息发送绑在同一个本地事务里，通过消息中间件（如 RocketMQ 的事务消息）保证消息一定能发出去，下游消费后达成最终一致：

```java
@Transactional
public void placeOrder(Order order) {
    orderDao.save(order);
    eventDao.save(new OrderCreatedEvent(order.getId())); // 同一事务
    // 定时任务扫描未发送的事件，确保消息最终发出
}
```

**Seata**：阿里开源的分布式事务框架，支持 AT（自动补偿）、TCC、Saga 等多种模式，AT 模式对业务侵入最小，是目前实际项目用得最多的。

📖 [[../../../24_SpringKnowledge/03_事务管理/01、Spring事务管理#九、分布式事务方案]]

---

###### 9. 什么是事务的回滚规则？

回滚规则控制"遇到什么异常触发回滚，遇到什么不回滚"。

**默认规则**：
- 触发回滚：`RuntimeException` 及其子类、`Error` 及其子类
- 不触发回滚：受检异常（`Exception` 的子类，但不是 `RuntimeException`），比如 `IOException`、`SQLException`

**自定义规则：**

```java
// 任何 Exception 都回滚（生产代码推荐）
@Transactional(rollbackFor = Exception.class)
public void criticalOperation() throws Exception { ... }

// 自定义业务异常触发回滚
@Transactional(rollbackFor = BusinessException.class)
public void operation() throws BusinessException { ... }

// IO 异常不触发回滚（比如只是日志写入失败，不影响主业务）
@Transactional(noRollbackFor = IOException.class)
public void loggingOperation() throws IOException { ... }
```

实际项目建议统一加 `rollbackFor = Exception.class`，避免因受检异常没有回滚导致数据不一致的问题。

📖 [[../../../24_SpringKnowledge/03_事务管理/01、Spring事务管理#四、@Transactional 参数详解]]

---

###### 10. Spring 事务的源码原理是什么？

Spring 事务的核心是 **AOP + PlatformTransactionManager + ThreadLocal** 三者的协作。

**代理创建阶段**：`InfrastructureAdvisorAutoProxyCreator`（一个 `BeanPostProcessor`）在 Bean 初始化后检查该 Bean 是否有 `@Transactional` 注解，有就创建代理对象，把 `TransactionInterceptor` 织入进去。

**方法调用阶段**：

1. 调用 `@Transactional` 方法 → 进入代理
2. `TransactionInterceptor.invoke()` 拦截 → 委托给 `TransactionAspectSupport.invokeWithinTransaction()`
3. `AnnotationTransactionAttributeSource` 解析方法上的 `@Transactional` 注解，获取事务属性
4. 根据属性找到对应的 `PlatformTransactionManager`
5. `getTransaction()` 开启事务（REQUIRED 时：有活跃事务就加入，没有就新建；并把数据库连接绑定到 ThreadLocal）
6. 执行业务方法
7. 正常返回 → `commit()`；抛异常 → 检查回滚规则 → `rollback()`

**ThreadLocal 的关键作用**：`TransactionSynchronizationManager` 把 `ConnectionHolder`（包含数据库连接）绑定到当前线程的 ThreadLocal。同一线程里的所有 DAO 操作从 ThreadLocal 拿到同一个连接，保证在同一个事务里。这也是为什么事务不能跨线程：子线程有自己的 ThreadLocal，拿不到父线程的事务上下文。

📖 [[../../../24_SpringKnowledge/03_事务管理/01、Spring事务管理#十、事务源码原理]]
