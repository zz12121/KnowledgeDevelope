###### 1. 什么是事务？事务的 ACID 特性是什么？
**事务**是数据库操作的最小工作单元，包含一个或多个SQL语句，这些语句要么全部成功执行，要么全部失败回滚。最经典的例子是银行转账：A账户减100元和B账户加100元两个操作必须作为一个不可分割的整体。
**ACID特性**是事务的四个核心属性：

|特性|核心含义|实现机制|
|---|---|---|
|**原子性（Atomicity）**​|事务中的所有操作要么全部提交成功，要么全部失败回滚|通过Undo Log实现。在事务执行过程中，数据库会记录数据修改前的镜像，事务回滚时用Undo Log恢复原始数据|
|**一致性（Consistency）**​|事务执行前后，数据库必须从一种一致性状态变换到另一种一致性状态|由应用层和数据库共同保证。应用层要编写正确的事务逻辑，数据库通过原子性、隔离性和持久性来保证一致性|
|**隔离性（Isolation）**​|并发事务之间相互隔离，一个事务的执行不应影响其他事务|通过锁机制、多版本并发控制（MVCC）等技术实现。不同隔离级别采用不同的隔离策略|
|**持久性（Durability）**​|事务提交后，其对数据的修改是永久性的，即使系统故障也不会丢失|通过Redo Log实现。事务提交前，先将数据变更写入Redo Log，即使系统崩溃也能通过Redo Log恢复数据|
###### 2. Spring 如何管理事务？
Spring通过**事务管理器**（`PlatformTransactionManager`）统一管理事务，提供了**声明式事务**和**编程式事务**两种管理方式。
**核心架构组件**：
- **PlatformTransactionManager**：事务管理器的核心接口
- **TransactionDefinition**：定义事务的属性（隔离级别、传播行为、超时时间、只读性）
- **TransactionStatus**：描述事务的状态（是否新事务、是否有回滚标记等）
**Spring事务管理流程**：
1. **事务拦截**：通过AOP（声明式）或模板方法（编程式）拦截目标方法
2. **事务创建**：根据`TransactionDefinition`创建或加入现有事务
3. **业务执行**：在事务上下文中执行目标方法
4. **事务处理**：根据执行结果提交或回滚事务
**源码机制**：Spring通过`TransactionInterceptor`实现声明式事务管理。当调用`@Transactional`方法时，拦截器会调用`TransactionAspectSupport#invokeWithinTransaction()`方法，该方法内部通过`PlatformTransactionManager`管理事务的完整生命周期。
###### 3. 编程式事务和声明式事务的区别是什么？

|特性|编程式事务|声明式事务|
|---|---|---|
|**控制方式**​|代码中显式控制事务边界|通过注解或XML配置声明事务属性|
|**侵入性**​|侵入性强，业务代码与事务代码耦合|非侵入性，事务管理与业务逻辑分离|
|**易用性**​|复杂度高，需要编写模板代码|简单易用，只需添加注解|
|**灵活性**​|灵活性高，可精确控制事务边界|灵活性相对较低，边界由AOP规则确定|
|**适用场景**​|复杂事务逻辑，需要精细控制|大多数常规业务场景|
**编程式事务实现示例**：
```java
public class AccountService{
    private TransactionTemplate transactionTemplate;

    public void transfer(final String out, final String in, final Double money) {
        transactionTemplate.execute(new TransactionCallbackWithoutResult(){
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus status){
                accountDao.outMoney(out,money); // 转出
                accountDao.inMoney(in,money);  // 转入
            }
        });
    }
}
```
**声明式事务实现示例**：
```java
@Service
public class AccountService2 {
    @Autowired
    private AccountDao accountDao;

    @Transactional
    public void transfer(String out, String in, Double money) {
        accountDao.outMoney(out, money); // 转出
        accountDao.inMoney(in, money);   // 转入
    }
}
```
###### 4. @Transactional 注解的使用方法和注意事项是什么？
**使用方法**：
- **类级别**：类中所有public方法都启用事务管理
- **方法级别**：仅对标注的方法启用事务管理（优先级高于类级别）
**完整参数配置**：
```java
@Transactional(
    value = "transactionManager",       // 指定事务管理器
    propagation = Propagation.REQUIRED, // 传播行为
    isolation = Isolation.DEFAULT,      // 隔离级别
    timeout = 30,                       // 超时时间（秒）
    readOnly = false,                   // 是否只读
    rollbackFor = {Exception.class},    // 触发回滚的异常
    noRollbackFor = {BusinessException.class} // 不触发回滚的异常
)
```
**重要注意事项**：
1. **代理机制限制**：基于动态代理，只有**public方法**上的注解才有效，且**自调用**（同一类中方法A调用方法B）不会触发事务
2. **异常回滚规则**：默认只在抛出**RuntimeException**和**Error**时回滚，受检异常不会回滚。可通过`rollbackFor`自定义
3. **事务管理器选择**：多数据源时需要明确指定事务管理器bean名称
4. **超时设置**：避免长时间持有数据库连接，根据业务复杂度设置合理超时
###### 5. Spring 事务的传播行为有哪些？
传播行为定义了**多个事务方法相互调用时，事务如何传播**的策略。Spring定义了7种传播行为：

|传播行为|值|说明|适用场景|
|---|---|---|---|
|**PROPAGATION_REQUIRED**​|0|默认。当前有事务则加入，无则新建|大多数业务场景|
|**PROPAGATION_SUPPORTS**​|1|当前有事务则加入，无则以非事务运行|查询方法，可适应事务环境|
|**PROPAGATION_MANDATORY**​|2|当前有事务则加入，无则抛异常|必须在外层事务中调用的方法|
|**PROPAGATION_REQUIRES_NEW**​|3|挂起当前事务，创建新事务|独立事务操作（如日志记录）|
|**PROPAGATION_NOT_SUPPORTED**​|4|以非事务运行，挂起当前事务|不支持事务的存储操作|
|**PROPAGATION_NEVER**​|5|以非事务运行，有事务则抛异常|不能在事务中调用的方法|
|**PROPAGATION_NESTED**​|6|当前有事务则在嵌套事务中执行，无则新建|复杂业务中的部分回滚场景|
**嵌套事务示例**：
```java
@Service
public class OrderService {
    @Transactional(propagation = Propagation.REQUIRED)
    public void placeOrder(Order order) {
        // 主业务逻辑
        orderDao.save(order);
        try {
            // 记录日志需要独立事务，即使订单失败日志也要保存
            logService.auditLog("Order created: " + order.getId());
        } catch (Exception e) {
            // 日志失败不应影响主业务
        }
    }
}

@Service
public class LogService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void auditLog(String message) {
        // 独立事务记录日志
    }
}
```
###### 6. Spring 事务的隔离级别有哪些？
隔离级别解决了并发事务可能导致的问题，Spring支持5种隔离级别：

|隔离级别|值|脏读|不可重复读|幻读|性能影响|
|---|---|---|---|---|---|
|**ISOLATION_DEFAULT**​|-1|取决于数据库默认|同左|同左|平衡|
|**ISOLATION_READ_UNCOMMITTED**​|1|可能|可能|可能|最低|
|**ISOLATION_READ_COMMITTED**​|2|阻止|可能|可能|较低|
|**ISOLATION_REPEATABLE_READ**​|4|阻止|阻止|可能|中等|
|**ISOLATION_SERIALIZABLE**​|8|阻止|阻止|阻止|最高|
**并发问题说明**：
- **脏读**：读取到其他事务未提交的数据
- **不可重复读**：同一事务内多次读取同一数据结果不一致（被其他事务修改）
- **幻读**：同一事务内多次查询返回的记录数不同（被其他事务插入/删除）
###### 7. 什么情况下 @Transactional 会失效？
**常见失效场景及解决方案**：
1. **自调用问题**（最常见）
```java
    @Service
    public class UserService {
        public void updateUser(User user) {
            saveAuditLog(user); // 自调用，@Transactional失效！
        }
    
        @Transactional
        public void saveAuditLog(User user) {
            // 不会在事务中执行
        }
    }
```
**解决方案**：注入自身代理或提取方法到其他类
2. **方法可见性非public**
```java
    @Service
    public class UserService {
        @Transactional
        protected void internalOperation() { // 失效！
        }
    }
```
3. **异常被捕获未抛出**
```java
    @Service
    public class OrderService {
        @Transactional
        public void placeOrder() {
            try {
                // 业务逻辑
            } catch (Exception e) {
                log.error("Error occurred", e); // 异常被吃掉，事务不会回滚
            }
        }
    }
```
4. **数据库引擎不支持事务**（如MyISAM）
5. **多数据源未指定事务管理器**
###### 8. 如何实现分布式事务？
分布式事务涉及多个独立资源（数据库、消息队列等）的事务一致性，常用方案：
**强一致性方案**：
- **XA协议/JTA**：基于两阶段提交（2PC），需要支持XA的资源和事务管理器
**最终一致性方案**：
- **TCC模式**（Try-Confirm-Cancel）
```java
    public class TccOrderService {
        @Transactional
        public void tryPhase() {
            // 预留资源：冻结金额、预扣库存
            accountService.freezeAmount(amount);
            inventoryService.prepareDecrease(productId, quantity);
        }
    
        public void confirmPhase() {
            // 确认操作：实际扣款、扣减库存
            accountService.confirmFreeze(amount);
            inventoryService.confirmDecrease(productId, quantity);
        }
    
        public void cancelPhase() {
            // 取消操作：解冻金额、恢复库存
            accountService.cancelFreeze(amount);
            inventoryService.cancelDecrease(productId, quantity);
        }
    }
```
- **消息队列+本地事件表**
```java
    @Transactional
    public void placeOrder(Order order) {
        // 1. 订单数据+事件记录在同一个数据库事务中
        orderDao.save(order);
        eventDao.saveEvent(new OrderCreatedEvent(order.getId()));
        // 2. 定时任务发送事件消息（确保最终一致）
    }
```
###### 9. 什么是事务的回滚规则？
回滚规则定义了**在什么情况下事务应该回滚**的策略配置。
**默认回滚规则**：
- 回滚：`RuntimeException`及其子类、`Error`及其子类
- 不回滚：受检异常（Exception子类但不是RuntimeException）
**自定义回滚规则**：
```java
@Service
public class BusinessService {
    // 遇到BusinessException时回滚
    @Transactional(rollbackFor = BusinessException.class)
    public void operation1() throws BusinessException {
    }
    
    // 遇到IOException时不回滚
    @Transactional(noRollbackFor = IOException.class)
    public void operation2() throws IOException {
    }
    
    // 所有异常都回滚
    @Transactional(rollbackFor = Exception.class)
    public void criticalOperation() throws Exception {
    }
}
```
###### 10. Spring事务实现的源码原理
Spring事务的实现原理是一个融合了**AOP（面向切面编程）**、**代理模式**​ 和**平台抽象**​ 的复杂机制，其核心目标是将事务管理这一横磨关注点与业务逻辑代码解耦。下面我们从核心架构、关键流程和源码细节三个层面进行剖析。
🔧 核心架构与组件
Spring事务的底层实现依赖于几个关键组件，它们协同工作，将声明式注解（如`@Transactional`)转化为具体的事务操作。

|组件|核心职责|关键实现类/接口|
|---|---|---|
|**事务管理器**​ (`PlatformTransactionManager`)|统一事务管理的核心接口，定义了事务的**获取、提交、回滚**行为。|`DataSourceTransactionManager`(JDBC), `HibernateTransactionManager`(Hibernate)|
|**事务属性源**​ (`TransactionAttributeSource`)|负责解析事务方法（如`@Transactional`）上的**元数据**，包括传播行为、隔离级别等。|`AnnotationTransactionAttributeSource`|
|**事务拦截器**​ (`TransactionInterceptor`)|一个AOP **MethodInterceptor**，是事务执行的**核心入口**，将事务属性转化为具体的事务操作。|`TransactionInterceptor`|
|**事务切面**​ (`BeanFactoryTransactionAttributeSourceAdvisor`)|一个AOP **Advisor**，它将**切入点**（匹配带有`@Transactional`的方法）和**通知**（`TransactionInterceptor`）组合成一个完整的切面。|`BeanFactoryTransactionAttribute|
|SourceAdvisor`|
|**自动代理创建器**​|在Bean生命周期中，负责扫描Bean，并为那些|
**需要事务增强的Bean创建代理对象**。 | `InfrastructureAdvisorAutoProxyCreator`|
这些组件的协作关系，可以用以下时序图来直观展示其核心工作流程：
```mermaid
flowchart TD
    A[@Transactional 方法被调用] --> B(调用 代理对象)
    B --> C{事务拦截器拦截调用}
    D[TransactionInterceptor] --> E
    E[PlatformTransactionManager] --> F
    F[DataSourceTransaction]
    C --> D
    subgraph 事务处理核心流程
        E
        F
    end
```
🚀 关键流程与源码解析
整个事务处理流程可以清晰地分为两个主要阶段：**代理创建**和**事务执行**。
 1. 代理创建与织入
在Spring容器启动时，通过自动代理机制完成事务增强：
- **自动代理创建器**​ (`InfrastructureAdvisorAutoProxyCreator`) 作为一个`BeanPostProcessor`，在Bean的**初始化后**阶段，会检查当前Bean是否**匹配**
**事务切面**（即`BeanFactoryTransactionAttributeSourceAdvisor`）。
- 如果匹配，则会为该Bean创建一个**代理对象**（**JDK动态代理**或**CGLIB代理**），并将`TransactionInterceptor`植入代理逻辑中。这使得后续对该Bean方法的调用会先经过`TransactionInterceptor`。
 2. 事务执行流程
当调用代理对象的方法时，核心逻辑进入`TransactionInterceptor`，它又委托给其父类`TransactionAspectSupport`的`invokeWithinTransaction`方法。此方法是事务执行的**大脑**，其核心步骤如下：
- **Step 1: 获取事务属性**
    通过`TransactionAttributeSource`解析当前被调用方法上的`@Transactional`注解，获取传播行为、隔离级别等属性，并封装为`TransactionAttribute`。
- **Step 2: 获取事务管理器**
    根据`TransactionAttribute`中的限定符或其它规则，确定使用哪个`PlatformTransactionManager`，这对于多数据源场景至关重要。
- **Step 3: 创建或加入事务（核心）**
    这是最复杂的一步，由`PlatformTransactionManager.getTransaction(TransactionDefinition)`方法实现，其内部处理逻辑如下：
	**1. 检查****当前线程是否存在活跃事务
	**2. 根据****传播行为****决定后续动作
	**3. 执行事务创建或挂**
	**4. 将资源绑定到当前线程**
- **Step 4: 执行业务逻辑**
    在已建立的事务上下文中，通过`MethodInvocation.proceed()`调用链，最终执行用户的业务代码。
- **Step 5: 提交或回滚**
    - 如果业务逻辑执行成功且未抛出异常，则调用`commit()`提交事务。
    - 如果抛**异常**，**则根据回滚规则**（默认对`RuntimeException`和`Error`回滚）决定是否调用`rollback()`。开发者可以通过`@Transactional(rollbackFor = Exception.class)`来定制回滚规则。
💡 核心机制深度解析
 传播行为的实现
传播行为决定了事务方法在调用时如何与现有事务交互。其核心实现机制如下：[citation1]
- **PROPAGATION_REQUIRED (默认)**: 如果当前存在事务，则加入该事务，否则创建一个新事务。
- **PROPRichATION_REQUIRES_NEW**: 无论当前是否存在事务，都创建一个新事务。如果**当前有事务**则将其挂起**，新事务独立提交或回滚，与挂起的事务无关。这是通过`TransactionSuspension`机制实现的。
- **PROPAGATION_NESTED**: 如果当前存在事务，则在**嵌套事务**中执行。它会在当前事务内设置一个**保存点**， 回滚时只回滚到该保存点，而不会影响整个外部事务。需要数据库支持保存点。
资源同步与线程绑定
Spring通过`TransactionSynchronizationManager`类，使用**ThreadLocal**将数据库连接（`ConnectionHolder`）等资源**绑定到当前线程**，确保在同一线程的多个事务操作（如多个DAO方法调用）中，使用的是同一个连接，**从而保证事务的原子性**。
⚠️ 常见失效场景与原理
理解源码后，可以轻松解释常见的`@Transactional`失效场景：
- **自调用**：在同一个类中非事务方法调用`@Transactional`方法，**会绕过代理对象**，直接调用目标方法，导致事务拦截器不生效。
- **方法非public**：`@Transactional`默认只对public方法生效，因为Spring的AOP代理机制无法代理非public方法。
- **异常被捕获**：事务回滚依赖于异常**被抛出**到拦截器，如果在方法内部被捕获并处理，拦截器将无法感知异常，**从而不会触发回滚**。
 📚 总结
Spring事务的源码实现是一个精妙的设计，它通过AOP和代理模式将**复杂的事务管理逻辑**封装**在`TransactionInterceptor`和`PlatformTransactionManager`中，**使得开发者可以通过简单的注解**享受强大而灵活的事务管理能力。 理解其底层原理，**不仅有助于避免常见的坑**，更能让我们在遇到复杂问题时能快速定位和解决。