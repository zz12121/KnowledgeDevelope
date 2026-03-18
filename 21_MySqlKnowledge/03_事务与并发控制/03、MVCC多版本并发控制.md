---
tags:
  - MySQL/MVCC
  - MySQL/并发控制
  - MySQL/事务
aliases:
  - 多版本并发控制
  - ReadView
  - 事务版本链
  - 快照读
date: 2026-03-18
---

# MVCC 多版本并发控制

> MVCC（Multi-Version Concurrency Control）是 InnoDB 实现高并发读写的核心机制，本质是**用空间换时间**——保存数据的多个历史版本，让读操作不阻塞写操作，写操作不阻塞读操作。

---

## 1. 核心设计思想

| 问题 | 传统锁方案 | MVCC 方案 |
|------|-----------|-----------|
| 读-写冲突 | 读加共享锁，写等读释放 | 读取历史快照，无需加锁 |
| 写-读冲突 | 写加排他锁，读等写释放 | 读不阻塞写 |
| 写-写冲突 | **无法避免，仍需加锁** | **无法避免，仍需加锁** |

**MVCC 只解决读-写冲突，写-写冲突依然依赖行锁。**

MVCC 工作在 **READ COMMITTED** 和 **REPEATABLE READ** 两个隔离级别下：
- `READ UNCOMMITTED`：直接读最新数据，不需要版本机制
- `SERIALIZABLE`：全程加锁串行化，不用快照

---

## 2. 三大核心组件

```
MVCC 实现原理
├── 隐藏字段        ← 每行数据的版本元信息
│   ├── DB_TRX_ID     最近修改该行的事务ID
│   ├── DB_ROLL_PTR   指向上一版本的回滚指针
│   └── DB_ROW_ID     无主键时自动生成的行ID
│
├── Undo Log        ← 版本链的存储介质
│   ├── Insert Undo Log   仅用于回滚，提交后可立即清理
│   └── Update Undo Log   用于回滚 + MVCC，需 Purge 线程清理
│
└── Read View       ← 可见性判断的"时间基准"
    ├── m_ids           生成时活跃事务ID集合
    ├── min_trx_id      活跃事务中最小ID
    ├── max_trx_id      下一个将分配的事务ID（当前最大+1）
    └── creator_trx_id  创建本 Read View 的事务自身ID
```

---

## 3. 隐藏字段详解

InnoDB 为每行数据自动添加 **3 个隐藏字段**，用户不可见但至关重要：

| 字段 | 大小 | 说明 |
|------|------|------|
| `DB_TRX_ID` | 6 字节 | 最近一次插入或更新该行的**事务 ID**。DELETE 在 InnoDB 内部被实现为一种特殊的 UPDATE（打删除标记） |
| `DB_ROLL_PTR` | 7 字节 | **回滚指针**，指向该行上一个历史版本在 Undo Log 中的地址，是构成版本链的关键 |
| `DB_ROW_ID` | 6 字节 | 当表**没有显式主键**时，InnoDB 用此字段自动创建聚簇索引；有主键时通常不存在 |

---

## 4. Undo Log 与版本链

### 4.1 Undo Log 的两种类型

| 类型 | 产生于 | 生命周期 | 说明 |
|------|--------|----------|------|
| **Insert Undo Log** | `INSERT` | 事务提交后**立即清理** | 只用于回滚，其他事务不会读取插入前的状态 |
| **Update Undo Log** | `UPDATE` / `DELETE` | 无活跃事务引用后，由 **Purge 线程**异步清理 | 既用于回滚，也供 MVCC 快照读使用 |

> ⚠️ **长事务的危害**：长事务的 Read View 长期存活，导致其引用的 Undo Log 版本无法清理，造成**表空间膨胀（undo segment 膨胀）**。这是生产中 `ibdata1` 文件不断增大的常见原因。

### 4.2 版本链的形成过程

假设事务 A（trx_id=10）插入一行数据，随后事务 B（trx_id=20）、事务 C（trx_id=30）相继更新：

```
初始状态（事务A INSERT）：
┌─────────────────────────────────────┐
│  name="Alice"  age=25               │
│  DB_TRX_ID = 10                     │
│  DB_ROLL_PTR = NULL                 │
└─────────────────────────────────────┘

事务B UPDATE（name="Bob"）：
                                    Undo Log
┌─────────────────────────────────┐  ┌─────────────────────────────────┐
│  name="Bob"   age=25            │  │  name="Alice"  age=25           │
│  DB_TRX_ID = 20                 │  │  DB_TRX_ID = 10                 │
│  DB_ROLL_PTR ──────────────────▶│  │  DB_ROLL_PTR = NULL             │
└─────────────────────────────────┘  └─────────────────────────────────┘
           （当前版本）                        （V1 历史版本）

事务C UPDATE（age=30）：
                  当前版本             V2（Undo Log）         V1（Undo Log）
┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐
│ name="Bob"   │  │ name="Bob"   │  │ name="Alice"  age=25 │
│ age=30       │  │ age=25       │  │ DB_TRX_ID=10         │
│ DB_TRX_ID=30 │  │ DB_TRX_ID=20 │  │ DB_ROLL_PTR=NULL     │
│ DB_ROLL_PTR─▶│  │ DB_ROLL_PTR─▶│  └──────────────────────┘
└──────────────┘  └──────────────┘
```

**版本链特点：**
- 链头 = 最新版本（当前数据页中）
- 链尾 = 最旧版本（最深的 Undo Log 中）
- 通过 `DB_ROLL_PTR` 单向链接

---

## 5. Read View 详解

### 5.1 四个关键字段

| 字段 | 含义 |
|------|------|
| `m_ids` | 生成 Read View 瞬间，**系统中所有活跃（未提交）事务的 ID 集合** |
| `min_trx_id` | `m_ids` 中的**最小值**（最早开始的活跃事务） |
| `max_trx_id` | 系统**即将分配的下一个事务 ID**（= 当前已分配最大值 + 1） |
| `creator_trx_id` | **创建本 Read View 的事务自身 ID** |

**直观理解：**
```
事务ID 时间轴：
──────────────────────────────────────────────────────▶
 1  2  3  4  5  6  7  8  9  10  11  12  ...
          [已提交]  [活跃: m_ids={5,7,9}]  [未来]
                    ↑                      ↑
               min_trx_id=5           max_trx_id=10
```

### 5.2 可见性判断算法（5步）

设某数据版本的事务 ID 为 `T`：

```
步骤1: T == creator_trx_id ?
    是 → 自己修改的，可见 ✓
    否 → 步骤2

步骤2: T < min_trx_id ?
    是 → 该事务在 Read View 创建前已提交，可见 ✓
    否 → 步骤3

步骤3: T >= max_trx_id ?
    是 → 该事务在 Read View 创建后才开启，不可见 ✗
    否 → 步骤4

步骤4: T 在 m_ids 中 ?
    是 → 创建 Read View 时该事务仍活跃（未提交），不可见 ✗
    否 → 步骤5

步骤5: T 不在 m_ids 中，且 min_trx_id ≤ T < max_trx_id
    → 该事务在 Read View 创建前已提交，可见 ✓
```

**如果当前版本不可见**：沿 `DB_ROLL_PTR` 找到上一个版本，重复判断，直到找到可见版本或链遍历完毕（返回空）。

---

## 6. RC vs RR：Read View 生成时机的核心差异

| 隔离级别 | Read View 生成策略 | 效果 |
|----------|-------------------|------|
| **READ COMMITTED (RC)** | **每次快照读都生成新的 Read View** | 能看到其他事务**最新提交**的数据 → 不可重复读 |
| **REPEATABLE READ (RR)** | **仅在事务第一次快照读时生成，后续复用** | 整个事务看到同一快照 → 可重复读 |

### 具体示例对比

```
时间轴：
T1: BEGIN                  ← 事务1开始
T2:   UPDATE name="Bob" WHERE id=1; COMMIT  ← 事务2提交
T3: SELECT name FROM t WHERE id=1;   ← 事务1读取
T4:   UPDATE name="Carol" WHERE id=1; COMMIT ← 事务3提交
T5: SELECT name FROM t WHERE id=1;   ← 事务1再次读取
T6: COMMIT

RC 下：
  T3 生成 Read View1 → 看到 "Bob"（T2已提交）
  T5 生成 Read View2 → 看到 "Carol"（T3已提交）
  → 两次读取结果不同：不可重复读 ✗

RR 下：
  T3 生成 Read View1 → 看到 "Bob"
  T5 复用 Read View1 → 仍然看到 "Bob"（T3提交的修改被隐藏）
  → 两次读取结果相同：可重复读 ✓
```

---

## 7. MVCC 解决了哪些并发问题

| 并发问题 | RR 级别 | RC 级别 | 说明 |
|----------|---------|---------|------|
| **脏读** | ✅ 解决 | ✅ 解决 | Read View 只能看到已提交版本 |
| **不可重复读** | ✅ 解决 | ❌ 不能解决 | RR 复用 Read View，RC 每次刷新 |
| **快照读幻读** | ✅ 解决 | ❌ 不能解决 | RR 复用 Read View，看不到新插入行 |
| **当前读幻读** | ❌ 不能解决 | ❌ 不能解决 | 需依赖 **Next-Key Lock** |

### 当前读 vs 快照读

| | 快照读（Snapshot Read） | 当前读（Current Read） |
|--|------------------------|----------------------|
| **触发语句** | 普通 `SELECT` | `SELECT ... FOR UPDATE`、`SELECT ... LOCK IN SHARE MODE`、`INSERT`、`UPDATE`、`DELETE` |
| **读取内容** | 历史快照（Read View） | 最新版本（加锁） |
| **MVCC 参与** | ✅ 参与 | ❌ 不参与，使用锁 |
| **幻读防护** | MVCC（RR 下） | Next-Key Lock |

> **RR 下幻读的完整解决方案**：
> - 快照读（普通 SELECT）→ 复用 Read View，天然防止幻读
> - 当前读（FOR UPDATE 等）→ Next-Key Lock 锁定范围，阻止插入

---

## 8. MVCC 优缺点与生产注意事项

### 优点

- **读不阻塞写，写不阻塞读**：高并发场景性能显著优于纯锁方案
- **降低死锁风险**：读操作无需加锁
- **一致性快照**：适合生成报表、数据分析等需要稳定视图的场景

### 缺点与注意事项

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **Undo Log 膨胀** | 长事务导致 Purge 线程无法清理旧版本 | 避免长事务，监控 `information_schema.INNODB_TRX` |
| **版本链过长** | 频繁更新热点行导致版本链很深 | 减少大事务中的热点更新 |
| **无法解决所有幻读** | 当前读不走 MVCC | 结合 Next-Key Lock，或业务层处理 |
| **写-写冲突** | MVCC 不处理写冲突 | 行锁/乐观锁处理 |

### 生产监控

```sql
-- 查看当前活跃事务（排查长事务）
SELECT trx_id, trx_started, trx_state, 
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) as seconds,
       trx_query
FROM information_schema.INNODB_TRX
ORDER BY trx_started ASC;

-- 查看 Undo Log 使用情况
SELECT name, subsystem, comment 
FROM information_schema.INNODB_METRICS
WHERE name LIKE 'trx_rseg%';

-- 查看 Purge 进度
SHOW ENGINE INNODB STATUS\G  -- 关注 History list length
```

> **经验法则**：`SHOW ENGINE INNODB STATUS` 中的 `History list length` 代表待 Purge 的 Undo Log 版本数，正常应 < 1000；若持续增长超过 10000，说明存在长事务阻塞 Purge。

---

## 9. 面试高频考点总结

| 考点 | 核心答案 |
|------|----------|
| MVCC 三要素 | 隐藏字段（DB_TRX_ID/DB_ROLL_PTR）+ Undo Log 版本链 + Read View |
| RC 和 RR 区别 | RC 每次快照读重新生成 Read View；RR 第一次生成后复用 |
| 可见性判断核心逻辑 | `T < min_trx_id` → 可见；`T >= max_trx_id` → 不可见；`T ∈ m_ids` → 不可见 |
| 为什么 RR 能防幻读（快照读） | 复用同一 Read View，新插入行的 trx_id 必然 >= max_trx_id，不可见 |
| 长事务的危害 | Read View 存活期间 Undo Log 无法清理，表空间膨胀 |

---

**相关面试题** → [[../../10_Developlanguage/002_SQL/01_MySQLSubject/06、MVCC（多版本并发控制）|📖]]

**相关知识点** → [[01、事务与ACID|事务与ACID]] | [[02、锁机制深度解析|锁机制]] | [[04、隔离级别与并发问题|隔离级别]]
