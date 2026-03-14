# 事务与 ACID

> 事务是数据库保证数据完整性的核心机制。理解 ACID 的实现原理（不只是定义），才能在面试和实战中游刃有余。

---

## 一、什么是事务

**事务（Transaction）** 是一组 SQL 操作的逻辑执行单元。这些操作要么**全部成功**，要么**全部回滚**，不存在中间状态。

```sql
-- 经典场景：银行转账
START TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 扣款
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- 存款

-- 检查业务约束
-- 如果余额不足，主动回滚
COMMIT;   -- 成功则提交
-- 或
ROLLBACK; -- 失败则回滚
```

---

## 二、ACID 特性与实现原理

ACID 是四个特性的首字母缩写，**每个特性在 InnoDB 中都有对应的实现机制**：

### 2.1 原子性（Atomicity）

**定义**：事务中的所有操作，要么全部执行，要么全部不执行。

**实现：Undo Log（回滚日志）**

```
事务执行过程：
┌─────────────────────────────────────────────────────────┐
│  BEGIN                                                   │
│  UPDATE users SET balance = 900 WHERE id = 1           │
│      → 同时写 Undo Log："UPDATE users SET balance = 1000 WHERE id = 1"
│  UPDATE users SET balance = 1100 WHERE id = 2          │
│      → 同时写 Undo Log："UPDATE users SET balance = 1000 WHERE id = 2"
│  ROLLBACK                                               │
│      → 读取 Undo Log，按逆序执行反向操作，恢复原始数据    │
└─────────────────────────────────────────────────────────┘
```

**Undo Log 的逻辑**：
- INSERT → Undo 记录主键，回滚时 DELETE
- DELETE → Undo 记录完整行，回滚时 INSERT
- UPDATE → Undo 记录修改前的旧值，回滚时执行反向 UPDATE

**注意**：Undo Log 是**逻辑日志**，记录 SQL 的反向操作，而不是数据页的物理差异。

### 2.2 一致性（Consistency）

**定义**：事务执行前后，数据库必须从一个一致性状态变换到另一个一致性状态。

**实现**：
- **数据库层面**：各种约束（主键约束、唯一约束、外键约束、CHECK 约束、数据类型约束）
- **原子性、隔离性、持久性的共同保证**：这三个特性是一致性的前提
- **业务层面**：应用代码中的业务规则（如余额不能为负）

**重要理解**：一致性是最终目标，原子性、隔离性、持久性是手段。如果业务逻辑本身有缺陷（比如没有检查余额是否充足就允许扣款），即使 ACID 全部满足，数据也可能"一致地错误"。

### 2.3 隔离性（Isolation）

**定义**：并发执行的事务互相隔离，一个事务的中间状态对其他事务不可见。

**实现：锁机制 + MVCC**（详见独立章节）

- **锁**：解决写-写冲突（两个事务同时修改同一行）
- **MVCC**：解决读-写冲突（一个事务读，另一个事务同时写）

### 2.4 持久性（Durability）

**定义**：事务一旦提交，对数据的修改就是永久性的，即使系统故障也不会丢失。

**实现：Redo Log + WAL 策略**

**为什么需要 Redo Log？**

InnoDB 使用 Buffer Pool 缓存数据页，数据修改先在内存中进行，再由后台线程异步刷到磁盘。如果事务提交后、刷盘前 MySQL 宕机，内存中的修改就会丢失。

**WAL（Write-Ahead Logging，预写式日志）**：

```
事务提交流程：
1. 修改 Buffer Pool 中的数据页（产生脏页）
2. 将修改写入 Redo Log Buffer（内存）
3. 事务提交 → 将 Redo Log Buffer fsync 到磁盘（Redo Log 文件）
   ↑ 这一步保证了持久性！
4. 后台线程择机将脏页刷到数据文件（异步）

崩溃恢复：
- 如果步骤 3 完成，步骤 4 未完成时宕机
- 重启后，InnoDB 读取 Redo Log，"重做"已提交的修改
- 数据页恢复到提交时的状态
```

**Redo Log 的物理特性**：
- **物理日志**：记录"在某个数据页的某个偏移量做了什么修改"
- **循环写入**：固定大小（如 4 个 1GB 的文件），写满后覆盖最旧的内容
- **顺序写入**：比数据文件的随机 I/O 快得多
- 通过 `innodb_flush_log_at_trx_commit` 控制刷盘时机

---

## 三、事务隔离级别

### 3.1 四种并发问题

| 问题 | 描述 | 举例 |
|------|------|------|
| **脏读（Dirty Read）** | 读到了另一个事务**未提交**的数据 | A 修改余额 100→50 未提交，B 读到 50，A 回滚，B 读到了无效数据 |
| **不可重复读（Non-repeatable Read）** | 同一事务内，同一行被读了两次，结果不同（另一事务在两次读之间**提交了修改**）| A 两次读取余额，第一次 100，B 转账后提交，第二次变成 50 |
| **幻读（Phantom Read）** | 同一事务内，同一范围查询了两次，结果集的**行数**不同（另一事务在两次查询之间**INSERT/DELETE 了数据**）| A 统计用户数 100，B 新增用户提交，A 再统计变成 101 |

**不可重复读 vs 幻读的区别**：
- 不可重复读：关注的是**同一行数据**被修改（UPDATE/DELETE）
- 幻读：关注的是**结果集的行数**变化（INSERT/DELETE）

### 3.2 四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | MySQL 实现方式 |
|---------|------|----------|------|--------------|
| **Read Uncommitted（读未提交）** | ❌可能 | ❌可能 | ❌可能 | 不加读锁，读最新版本 |
| **Read Committed（读已提交，RC）** | ✅避免 | ❌可能 | ❌可能 | MVCC（每次快照读生成新 ReadView）|
| **Repeatable Read（可重复读，RR）** | ✅避免 | ✅避免 | ⚠️部分避免 | MVCC（复用第一次的 ReadView）+ Next-Key Lock |
| **Serializable（串行化）** | ✅避免 | ✅避免 | ✅避免 | 全部加锁，读写串行 |

**MySQL 默认隔离级别是 RR（Repeatable Read）**，比 SQL 标准要求（通常是 RC）更严格。

### 3.3 RR 级别下幻读的解决

InnoDB 在 RR 级别下通过**两种机制**解决幻读：

**① 快照读（普通 SELECT）：MVCC 解决**
- 同一事务内复用第一次生成的 ReadView
- 后续其他事务的 INSERT 对当前事务不可见
- 有效避免幻读

**② 当前读（SELECT ... FOR UPDATE / UPDATE / DELETE）：Next-Key Lock 解决**
- 当前读读取最新版本数据，MVCC 不起作用
- 通过 Next-Key Lock（记录锁 + 间隙锁）锁住范围，阻止其他事务插入

```sql
-- 事务 A 执行范围查询（当前读）
BEGIN;
SELECT * FROM users WHERE age BETWEEN 20 AND 30 FOR UPDATE;
-- InnoDB 锁住：age=20 的行（Record Lock）+ age (20,30] 的区间（Gap Lock）
-- 其他事务无法在 age 20~30 范围内插入新行

-- 事务 B 尝试插入
INSERT INTO users (name, age) VALUES ('Charlie', 25);
-- ⏳ 等待，被 Gap Lock 阻塞！直到事务 A 提交或回滚
```

### 3.4 操作隔离级别

```sql
-- 查看当前会话隔离级别
SELECT @@transaction_isolation;

-- 查看全局隔离级别
SELECT @@global.transaction_isolation;

-- 设置当前会话隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Java Spring 中设置
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void transferMoney(Account from, Account to, BigDecimal amount) { ... }
```

---

## 四、当前读与快照读

| 读取方式 | 触发条件 | 读取内容 | 机制 |
|---------|---------|---------|------|
| **快照读（Snapshot Read）** | 普通 `SELECT` | 某个历史版本 | MVCC + ReadView |
| **当前读（Current Read）** | `SELECT ... FOR UPDATE`、`SELECT ... LOCK IN SHARE MODE`、`UPDATE`、`DELETE`、`INSERT` | 最新提交版本 | 加锁（行锁/Next-Key Lock）|

**理解关键点**：
- 同一个 `SELECT` 语句在 RR 级别下：
  - 普通 `SELECT` = 快照读，看到事务开始时的数据
  - 加 `FOR UPDATE` = 当前读，看到最新数据，并加锁

---

## 五、面试要点速查

| 问题 | 核心答案 |
|------|---------|
| 原子性如何实现？ | Undo Log，事务失败时通过逻辑反向操作恢复 |
| 持久性如何实现？ | Redo Log + WAL：提交时先 fsync Redo Log，后台异步刷数据页 |
| 一致性如何保证？ | 原子性+隔离性+持久性共同保证，加上数据库约束和业务逻辑 |
| MySQL 默认隔离级别？ | Repeatable Read（可重复读） |
| RR 和 RC 的区别？ | RR：第一次快照读生成 ReadView 并复用（不可重复读问题消失）；RC：每次快照读生成新 ReadView |
| RR 级别下如何解决幻读？ | 快照读用 MVCC，当前读用 Next-Key Lock（间隙锁+行锁）|
| Redo Log 和 Undo Log 的区别？ | Redo Log 是物理日志，保证持久性（崩溃恢复）；Undo Log 是逻辑日志，保证原子性（回滚）和 MVCC |
| 当前读和快照读的区别？ | 快照读（普通SELECT）走 MVCC 读历史版本；当前读（FOR UPDATE等）读最新版本并加锁 |

---

**相关面试题** → [[../../10_Developlanguage/002_SQL/01_MySQLSubject/04、事务]]
