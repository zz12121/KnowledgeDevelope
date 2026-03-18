---
tags:
  - MySQL/日志
  - MySQL/RedoLog
  - MySQL/Binlog
aliases:
  - Redo Log
  - Undo Log
  - Binlog
  - 崩溃恢复
date: 2026-03-18
---

# 日志系统（Redo Log / Undo Log / Binlog）

> MySQL 的三大核心日志各司其职：**Redo Log** 保证崩溃恢复（持久性），**Undo Log** 支持回滚与 MVCC（原子性），**Binlog** 服务于主从复制和数据归档。三者配合，构成 InnoDB 事务可靠性的基石。

---

## 1. 日志系统全景

```
MySQL 日志体系
├── InnoDB 引擎层（事务相关）
│   ├── Redo Log   物理日志，崩溃恢复，循环写入
│   └── Undo Log   逻辑日志，回滚 + MVCC，Purge 清理
│
└── MySQL Server 层（通用）
    ├── Binlog        逻辑日志，主从复制 + 时间点恢复
    ├── Slow Query Log 慢查询，性能调优
    ├── Error Log      错误诊断
    └── General Log    全量审计（生产慎用）
```

---

## 2. Redo Log（重做日志）

### 2.1 核心作用与 WAL

**Redo Log 是 WAL（Write-Ahead Logging，预写式日志）** 机制的实现载体。

**为什么需要 Redo Log？**

InnoDB 使用 Buffer Pool 缓存数据页，修改操作先在内存中进行，异步刷盘。如果在脏页刷盘前系统崩溃，内存中的修改会丢失。直接每次修改都同步刷盘（随机 I/O）又太慢。

**WAL 的解决思路**：
```
修改数据 → 记录到 Redo Log（顺序 I/O，快）→ 事务提交
                    ↓
         后台线程择机将脏页刷回磁盘（随机 I/O，异步）
```

顺序写 > 随机写，所以先写 Redo Log 再刷脏页，既保证了持久性，又避免了每次修改都随机 I/O 的性能问题。

### 2.2 物理结构

Redo Log 由**固定大小的文件组**构成，默认为 `ib_logfile0` 和 `ib_logfile1`，循环写入：

```
┌─────────────────────────────────────────────────────────┐
│                    Redo Log 文件（环形）                  │
│                                                         │
│  ib_logfile0          ib_logfile1                       │
│  ┌───────────────┐    ┌───────────────┐                │
│  │▓▓▓▓▓▓▓░░░░░░░│───▶│░░░░░░░░░░░░░░░│──┐              │
│  └───────────────┘    └───────────────┘  │              │
│        ↑                                 │              │
│    checkpoint                            │              │
│    （已刷盘，可覆盖）                        │              │
│                                          ↓              │
│                           write pos（当前写入位置）      │
└─────────────────────────────────────────────────────────┘

▓ = 已写入待 checkpoint 推进   ░ = 空闲可写入
```

**关键指针**：
- **write pos**：当前 Redo Log 的写入位置，向前推进
- **checkpoint**：已刷脏页到磁盘的位置，write pos 追上 checkpoint 时必须先推进 checkpoint（刷脏页）才能继续写

### 2.3 刷盘策略（innodb_flush_log_at_trx_commit）

| 值 | 行为 | 性能 | 安全性 | 适用场景 |
|----|------|------|--------|---------|
| `1`（默认） | 每次事务提交，**同步刷盘**（`fsync`） | 最低 | 最高（零丢失） | 金融、支付等强一致性场景 |
| `2` | 写入 OS 缓存，每秒 `fsync` | 中 | 较高（宕机最多丢1秒） | 多数业务的推荐设置 |
| `0` | 每秒写缓存 + 刷盘 | 最高 | 最低（MySQL崩溃可能丢1秒） | 可接受少量丢失的日志类场景 |

> **生产建议**：核心交易系统用 `=1`；日志流水等可用 `=2`，性能提升约 3-5 倍。

### 2.4 Crash Recovery（崩溃恢复）

```
MySQL 重启流程：
1. 扫描 Redo Log，找到最后一个 checkpoint 点
2. 从 checkpoint 开始，重放（redo）所有已提交事务的操作
3. 对于处于 Prepare 状态（两阶段提交中间态）的事务：
   - 检查 Binlog 中是否有对应的完整事务记录
   - 有 → 提交（恢复该事务）
   - 无 → 回滚（丢弃该事务）
```

### 2.5 关键配置参数

```sql
-- 查看 Redo Log 文件大小（默认 48MB，生产建议 1-4GB）
SHOW VARIABLES LIKE 'innodb_log_file_size';

-- 查看 Redo Log 文件数
SHOW VARIABLES LIKE 'innodb_log_files_in_group';

-- 查看刷盘策略
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';

-- 监控 Checkpoint 与写入差距（差距过大说明刷盘跟不上写入）
SHOW ENGINE INNODB STATUS\G  -- 关注 Log sequence number 和 Log flushed up to 的差值
```

---

## 3. Undo Log（回滚日志）

### 3.1 双重职责

| 职责 | 触发场景 | 说明 |
|------|---------|------|
| **事务回滚（原子性）** | `ROLLBACK` 或异常 | 利用逆向操作恢复数据 |
| **MVCC 版本链** | 快照读（普通 SELECT） | 提供历史版本供其他事务读取 |

### 3.2 逻辑日志的工作原理

Undo Log 是**逻辑日志**，记录的是操作的**逆操作**：

| 原始操作 | Undo Log 记录 |
|---------|--------------|
| `INSERT INTO t VALUES(1, 'a')` | `DELETE FROM t WHERE pk=1` |
| `DELETE FROM t WHERE pk=1` | `INSERT INTO t VALUES(1, 'a')（完整数据）` |
| `UPDATE t SET name='b' WHERE pk=1` | `UPDATE t SET name='a' WHERE pk=1（旧值）` |

### 3.3 两种 Undo Log 的生命周期

```
INSERT Undo Log:
  事务执行 INSERT ──→ 生成 Insert Undo Log ──→ 事务提交 ──→ 立即清理 ✓
  
Update Undo Log:
  事务执行 UPDATE/DELETE ──→ 生成 Update Undo Log
     └→ 事务提交（不能立即清理！）
          └→ 等待所有引用此版本的 Read View 消失
               └→ Purge 线程异步清理 ✓
```

### 3.4 存储位置

- MySQL 5.6 之前：Undo Log 存储在系统表空间（`ibdata1`），无法缩小
- MySQL 5.6+：支持独立 Undo 表空间（`innodb_undo_tablespaces`）
- MySQL 8.0：默认使用独立 Undo 表空间，支持自动 truncate

```sql
-- 查看 Undo Log 相关配置
SHOW VARIABLES LIKE 'innodb_undo%';

-- 查看回滚段状态
SELECT * FROM information_schema.INNODB_TABLESPACES 
WHERE NAME LIKE 'innodb_undo%';
```

---

## 4. Binlog（二进制日志）

### 4.1 与 Redo Log 的本质区别

| 维度 | Redo Log | Binlog |
|------|----------|--------|
| **所属层** | InnoDB 引擎层 | MySQL Server 层（引擎无关） |
| **日志性质** | **物理日志**（页偏移量级别的修改） | **逻辑日志**（SQL 或行变更逻辑） |
| **写入方式** | **循环写入**（固定大小，覆盖旧日志） | **追加写入**（不断新增文件，可归档） |
| **主要用途** | 崩溃恢复（本机） | 主从复制、时间点恢复、数据订阅 |
| **事务感知** | 与 InnoDB 事务强绑定 | Server 层记录，不直接感知引擎事务 |

### 4.2 三种 Binlog 格式

| 格式 | 记录内容 | 优点 | 缺点 | 适用场景 |
|------|---------|------|------|---------|
| **Statement (SBR)** | 原始 SQL 语句 | 日志体积小 | 非确定性函数（`NOW()`/`RAND()`）可能导致主从不一致 | 简单 SQL，逻辑一致的场景 |
| **Row (RBR)** | 每行数据修改前后的完整镜像 | 绝对精准，无不一致风险 | 批量操作（百万行 UPDATE）日志体积极大 | 金融交易、要求绝对一致 |
| **Mixed (MBR)** | 默认 Statement，遇非确定性自动切换 Row | 兼顾性能与安全 | 少量不确定性边界 | 通用推荐 |

> **MySQL 5.7.7+ 默认格式为 Row**，结合 `binlog_row_image=MINIMAL`（只记录变更列）可降低日志量。

```sql
-- 查看当前 Binlog 格式
SHOW VARIABLES LIKE 'binlog_format';

-- 查看 Binlog 文件列表
SHOW BINARY LOGS;

-- 查看 Binlog 内容（指定文件）
SHOW BINLOG EVENTS IN 'mysql-bin.000001' LIMIT 20;

-- 命令行解析 Binlog
mysqlbinlog --base64-output=DECODE-ROWS -v mysql-bin.000001 | head -100
```

### 4.3 Binlog 的生产应用

**数据恢复（时间点恢复）**：

```bash
# 从全量备份 + Binlog 恢复到指定时间点
# 1. 先恢复全量备份
mysqldump --all-databases > full_backup.sql
mysql < full_backup.sql

# 2. 应用增量 Binlog（恢复到2026-03-14 22:00:00）
mysqlbinlog --start-datetime="2026-03-14 20:00:00" \
            --stop-datetime="2026-03-14 22:00:00" \
            mysql-bin.000001 mysql-bin.000002 | mysql -u root -p
```

**数据订阅（Canal 实时同步）**：

```
MySQL Binlog ──→ Canal Server（模拟从库，解析 Binlog）
                      ↓
              Canal Client（应用消费）
                  ├── 同步到 Redis（缓存刷新）
                  ├── 同步到 ES（搜索索引更新）
                  └── 同步到消息队列（异步业务处理）
```

---

## 5. 两阶段提交（2PC）

### 5.1 为什么需要两阶段提交

Redo Log（引擎层）和 Binlog（Server 层）是两个独立的日志系统。如果不协调，任一方写入失败都会导致数据不一致：

| 崩溃场景 | 不一致后果 |
|---------|-----------|
| Redo Log 写完，Binlog 未写，崩溃 | 主库重启后有该数据（Redo 恢复），Binlog 没有 → 从库少了这条数据 |
| Binlog 写完，Redo Log 未提交，崩溃 | 主库回滚了该数据，从库有 → 主从不一致 |

### 5.2 两阶段提交流程

```
事务提交过程（两阶段提交）：

InnoDB 引擎                    MySQL Server
    │                              │
    │ 1. 写 Redo Log               │
    │    标记为 PREPARE 状态        │
    │                              │
    │──────────────────────────────▶│
    │                              │ 2. 写 Binlog
    │                              │    fsync 到磁盘
    │◀──────────────────────────────│
    │                              │
    │ 3. 写 Redo Log               │
    │    标记为 COMMIT 状态         │
    │    （事务真正提交完成）         │
```

**崩溃恢复判断逻辑**：
```
检查 Redo Log 中处于 PREPARE 的事务：
├── 找到对应 XID 在 Binlog 中有完整记录 → 提交（主库和从库都要有）
└── Binlog 中没有该 XID              → 回滚（两边都丢弃）
```

### 5.3 组提交（Group Commit）优化

两阶段提交存在多次 `fsync` 调用，高并发下性能瓶颈明显。MySQL 5.6 引入了**组提交（Group Commit）**：

```
多个并发事务按时间窗口分组，一次 fsync 刷写多个事务的日志：

事务1 ──┐
事务2 ──┤─ 组内排队 ──→ 一次 fsync 刷盘（ Binlog 或 Redo Log）
事务3 ──┘
```

相关参数：
```sql
-- 控制组提交的延迟（等待更多事务加入组，牺牲延迟换吞吐）
SHOW VARIABLES LIKE 'binlog_group_commit_sync_delay';    -- 微秒，默认0
SHOW VARIABLES LIKE 'binlog_group_commit_sync_no_delay_count'; -- 触发的最小事务数
```

---

## 6. 慢查询日志（Slow Query Log）

### 6.1 配置与启用

```sql
-- 动态开启慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;       -- 超过1秒记录（生产可设0.5）
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录全表扫描
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 查看当前配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';
```

### 6.2 分析工具

```bash
# mysqldumpslow（内置工具，简单统计）
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log    # 按总耗时排序，取前10

# pt-query-digest（Percona Toolkit，专业分析）
pt-query-digest /var/log/mysql/slow.log > slow_report.txt
```

pt-query-digest 报告关键指标：
- **Rank**：按总耗时排名
- **Response time**：总时间及占比
- **Calls**：执行次数
- **R/Call**：平均响应时间

---

## 7. 日志系统常见面试考点

| 问题 | 核心答案 |
|------|---------|
| Redo Log 和 Binlog 的区别 | 引擎层物理循环写 vs Server 层逻辑追加写；崩溃恢复 vs 主从复制 |
| 为什么 Redo Log 用循环写而 Binlog 用追加写 | Redo Log 只需保留未刷盘的增量（checkpoint 之前的可丢弃）；Binlog 需要归档，供从库和恢复使用 |
| 两阶段提交解决什么问题 | 保证 Redo Log 和 Binlog 逻辑一致，避免主从数据分叉 |
| Undo Log 为什么不能事务提交后立即删除 | Update Undo Log 需要为活跃事务的 Read View 提供历史版本（MVCC） |
| `innodb_flush_log_at_trx_commit=2` 什么情况会丢数据 | OS 崩溃（内核 Panic），MySQL 崩溃不会丢数据 |

---

**相关面试题** → [[../../10_Developlanguage/002_SQL/01_MySQLSubject/07、日志系统|📖]]

**相关知识点** → [[../03_事务与并发控制/01、事务与ACID|事务与ACID]] | [[../03_事务与并发控制/03、MVCC多版本并发控制|MVCC]]
