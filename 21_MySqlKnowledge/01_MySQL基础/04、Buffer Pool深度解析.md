---
tags:
  - MySQL/InnoDB
  - MySQL/BufferPool
  - MySQL/内存管理
  - MySQL/性能优化
aliases:
  - Buffer Pool
  - 缓冲池
  - InnoDB内存结构
  - LRU算法
date: 2026-03-18
---

# Buffer Pool 深度解析

> Buffer Pool 是 InnoDB 存储引擎的**核心内存组件**，所有数据页的读写操作都经过这里。深入理解 Buffer Pool 的工作机制，是掌握 MySQL 性能调优和故障排查的关键。

---

## 1. Buffer Pool 全景架构

### 1.1 内存布局总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         InnoDB Buffer Pool                               │
│                         （默认 128MB，生产 8G~64G）                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                     Data Pages（数据页）                         │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │    │
│  │  │ Page 16K│ │ Page 16K│ │ Page 16K│ │ Page 16K│ │ ......  │   │    │
│  │  │ 用户数据 │ │ 用户数据 │ │ 用户数据 │ │ 用户数据 │ │         │   │    │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘   │    │
│  │                    约占 Buffer Pool 的 70%~80%                    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────┐  ┌─────────────────────────────────────┐   │
│  │    Index Pages（索引页） │  │    Insert Buffer（插入缓冲）         │   │
│  │    B+树节点页            │  │    非唯一二级索引的插入优化           │   │
│  │    约占 15%~20%          │  │    约占 5%~10%                       │   │
│  └─────────────────────────┘  └─────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────┐  ┌─────────────────────────────────────┐   │
│  │    Undo Pages（回滚页）  │  │    Adaptive Hash Index（自适应哈希）  │   │
│  │    事务回滚数据          │  │    热点页哈希加速                    │   │
│  │    约占 5%~10%           │  │    约占 1%~5%                        │   │
│  └─────────────────────────┘  └─────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    Lock Info（锁信息）                           │    │
│  │    行锁、表锁等锁结构的内存表示                                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Buffer Pool 的核心作用

| 作用 | 说明 | 性能影响 |
|------|------|---------|
| **缓存数据页** | 将磁盘数据页加载到内存，避免重复磁盘 I/O | 命中时纳秒级访问，未命中时毫秒级磁盘 I/O |
| **缓存索引页** | B+树节点常驻内存，加速索引查找 | 减少索引遍历的磁盘访问次数 |
| **延迟写优化** | 脏页不立即刷盘，批量异步写入 | 将随机 I/O 转化为顺序 I/O |
| **预读机制** | 顺序扫描时预加载后续页面 | 减少顺序访问的 I/O 等待 |

---

## 2. 数据页（Page）管理机制

### 2.1 页的基本结构

InnoDB 以 **Page（页）** 为最小存储和 I/O 单元，默认大小为 **16KB**：

```
┌─────────────────────────────────────────────────────────────┐
│                      InnoDB Page（16KB）                     │
├─────────────────────────────────────────────────────────────┤
│  File Header（38字节）                                        │
│  ├── Page Number（页号）                                      │
│  ├── Page Type（页类型：数据页/索引页/Undo页等）               │
│  ├── Pre/Next Page（双向链表指针）                             │
│  └── Checksum（校验和）                                       │
├─────────────────────────────────────────────────────────────┤
│  Page Header（56字节）                                        │
│  ├── Slot Number（槽数量）                                    │
│  ├── Heap Top（堆顶位置）                                     │
│  ├── Record Number（记录数）                                  │
│  └── Last Insert Position（最后插入位置）                      │
├─────────────────────────────────────────────────────────────┤
│  Infimum + Supremum Records（虚拟最小/最大记录）               │
├─────────────────────────────────────────────────────────────┤
│  User Records（用户数据记录）                                  │
│  ├── Record Header（记录头：删除标记、下一条记录偏移等）        │
│  └── Record Body（实际数据：主键、列数据、事务ID、回滚指针）    │
├─────────────────────────────────────────────────────────────┤
│  Free Space（空闲空间）                                       │
├─────────────────────────────────────────────────────────────┤
│  Page Directory（页目录）                                     │
│  └── Slot 数组（二分查找加速定位记录）                         │
├─────────────────────────────────────────────────────────────┤
│  File Trailer（8字节）                                        │
│  └── Checksum（与 File Header 校验和一致，检测页完整性）       │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 页的类型与用途

| 页类型 | 类型值 | 用途 | 占比 |
|--------|--------|------|------|
| **FIL_PAGE_INDEX** | 0x45BF | B+树索引节点页 | ~60% |
| **FIL_PAGE_DATA** | 0x0002 | 表数据页（InnoDB 表空间） | ~20% |
| **FIL_PAGE_UNDO_LOG** | 0x0002 | Undo Log 页 | ~10% |
| **FIL_PAGE_IBUF_BITMAP** | 0x0005 | Insert Buffer 位图页 | ~5% |
| **FIL_PAGE_INODE** | 0x0003 | 段信息页 | ~3% |
| **FIL_PAGE_TYPE_SYS** | 0x0006 | 系统页（数据字典等） | ~2% |

---

## 3. Buffer Pool 的 LRU 算法详解

### 3.1 传统 LRU 的问题

传统 LRU（Least Recently Used）算法存在**缓存污染**问题：

```
场景：全表扫描 100万 行数据

传统 LRU 链表：
  头部（最近使用）                    尾部（最久未使用）
    A → B → C → D → E → F → G → H → I → J
    
全表扫描后（加载大量冷数据）：
  头部                              尾部（热数据被挤出！）
    Scan1 → Scan2 → ... → ScanN → A → B → C
    
问题：一次性扫描把所有热数据都挤出缓存，后续正常查询反而大量未命中
```

### 3.2 InnoDB 的改进型 LRU（Young/Old 分区）

InnoDB 将 LRU 链表分为 **Young 区（热数据）** 和 **Old 区（冷数据）**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Buffer Pool LRU 链表结构                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   头部（最近访问）                                          尾部    │
│     │                                                        │      │
│     ▼                                                        ▼      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Young 区（5/8）        │        Old 区（3/8）                │  │
│  │  ┌─────┐ ┌─────┐       │       ┌─────┐ ┌─────┐ ┌─────┐       │  │
│  │  │ Hot │ │ Hot │  ...  │  ...  │ Cold│ │ Cold│ │ Cold│       │  │
│  │  │  A  │ │  B  │       │       │  X  │ │  Y  │ │  Z  │       │  │
│  │  └──┬──┘ └──┬──┘       │       └──┬──┘ └──┬──┘ └──┬──┘       │  │
│  │     └───────┘          │          └───────┘       │          │  │
│  │                        │                          │          │  │
│  │   热数据，长期保留       │    新加载页先进入这里      │          │  │
│  │   全表扫描不影响         │    需"考验期"才能晋升      │          │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  参数控制：innodb_old_blocks_pct = 37（默认，Old区占比37%）          │
│           innodb_old_blocks_time = 1000（默认，Old区停留1秒才晋升）  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 数据页加载与淘汰流程

```
查询数据页流程：

1. 计算页哈希值，查找 Buffer Pool 的 Hash Table
   │
   ├── 命中（Page in Buffer Pool）
   │   │
   │   ├── 页在 Young 区 → 移动到 Young 区头部（MRU）
   │   │
   │   └── 页在 Old 区 → 检查 innodb_old_blocks_time
   │       │
   │       ├── 停留时间 < 1秒 → 不移动（防止扫描污染）
   │       │
   │       └── 停留时间 >= 1秒 → 移动到 Young 区头部
   │
   └── 未命中（Page Not in Buffer Pool）
       │
       ├── 从磁盘读取页到 Old 区头部
       │
       ├── 触发预读（Read-Ahead）
       │   ├── 线性预读：顺序访问 Extent 的页时，预读下一个 Extent
       │   └── 随机预读：同一 Extent 的页被多次访问，预读整个 Extent
       │
       └── 如果 Free List 为空，从 LRU 尾部淘汰页
           │
           ├── 干净页（未修改）→ 直接移除
           │
           └── 脏页（已修改）→ 触发 Flush，刷盘后移除
```

### 3.4 关键配置参数

```sql
-- Buffer Pool 大小（生产环境建议设置为物理内存的 50%~75%）
-- MySQL 5.7+ 支持动态调整
SET GLOBAL innodb_buffer_pool_size = 8589934592;  -- 8GB

-- Old 区占比（默认 37%，即 3/8）
SET GLOBAL innodb_old_blocks_pct = 37;

-- Old 区晋升到 Young 区的最小停留时间（毫秒，默认 1000）
SET GLOBAL innodb_old_blocks_time = 1000;

-- 查看 Buffer Pool 状态
SHOW ENGINE INNODB STATUS\G
-- 关注：Buffer pool hit rate（命中率，应 > 99%）

-- 查看 Buffer Pool 详细统计
SELECT 
    POOL_ID,
    POOL_SIZE,           -- 总页数
    FREE_BUFFERS,        -- 空闲页数
    DATABASE_PAGES,      -- 数据页数
    OLD_DATABASE_PAGES,  -- Old 区页数
    PAGES_MADE_YOUNG,    -- 晋升到 Young 区的次数
    PAGES_NOT_MADE_YOUNG -- 未晋升次数（扫描导致）
FROM information_schema.INNODB_BUFFER_POOL_STATS;
```

---

## 4. 脏页（Dirty Page）管理

### 4.1 脏页的产生与危害

```
事务执行 UPDATE 操作：

1. 读取数据页到 Buffer Pool（如果不在内存）
2. 在 Buffer Pool 中修改数据页内容
3. 生成 Redo Log 并写入 Log Buffer
4. 事务提交时，Redo Log 刷盘（保证持久性）
5. 数据页标记为 Dirty（与磁盘不一致）

┌─────────────────────────────────────────────────────────────┐
│  Buffer Pool 中的页                                          │
│  ┌─────────────┐                                            │
│  │  Data Page  │  内存版本：已修改（脏页）                    │
│  │   (Dirty)   │  磁盘版本：旧数据                            │
│  └─────────────┘                                            │
│                                                             │
│  此时如果系统崩溃：                                          │
│  - 内存中的脏页丢失                                          │
│  - 但 Redo Log 已持久化                                      │
│  - 重启时通过 Redo Log 恢复                                  │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 脏页刷盘策略

InnoDB 采用**多线程异步刷盘**，避免阻塞用户线程：

```
┌─────────────────────────────────────────────────────────────────┐
│                    脏页刷盘机制                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐ │
│  │  Page Cleaner   │    │  Page Cleaner   │    │   Flush     │ │
│  │    Thread 1     │    │    Thread 2     │    │   Thread    │ │
│  │  （刷新脏页）    │    │  （刷新脏页）    │    │（批量刷盘）  │ │
│  └────────┬────────┘    └────────┬────────┘    └──────┬──────┘ │
│           │                      │                     │        │
│           └──────────────────────┼─────────────────────┘        │
│                                  ▼                              │
│                    ┌─────────────────────────┐                  │
│                    │      Flush List         │                  │
│                    │  （按最早修改时间排序）   │                  │
│                    │                         │                  │
│                    │  Page A → Page B → ...  │                  │
│                    │  (LSN=1000)  (LSN=1200) │                  │
│                    └─────────────────────────┘                  │
│                                  │                              │
│                                  ▼                              │
│                         ┌────────────────┐                      │
│                         │   磁盘数据文件   │                      │
│                         └────────────────┘                      │
│                                                                  │
│  触发条件：                                                       │
│  1. LRU 淘汰时遇到脏页                                            │
│  2. Redo Log 快满（write pos 接近 checkpoint）                    │
│  3. 定时刷新（默认每秒）                                          │
│  4. 脏页比例超过阈值（innodb_max_dirty_pages_pct=75%）            │
│  5. 系统空闲时主动刷新                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 刷盘的关键参数

```sql
-- Page Cleaner 线程数（默认 4，SSD 可增大）
SET GLOBAL innodb_page_cleaners = 8;

-- 脏页占 Buffer Pool 比例达到多少开始刷盘（默认 75%）
SET GLOBAL innodb_max_dirty_pages_pct = 75.0;

-- 脏页达到此比例强制刷盘（默认 90%，避免 Redo Log 满）
SET GLOBAL innodb_max_dirty_pages_pct_lwm = 10.0;

-- 每秒刷盘页数上限（默认 200，IO 能力强可增大）
SET GLOBAL innodb_io_capacity = 2000;

-- 紧急情况下每秒刷盘页数上限（默认 2000）
SET GLOBAL innodb_io_capacity_max = 4000;

-- 自适应刷盘（根据脏页产生速度动态调整）
SET GLOBAL innodb_adaptive_flushing = ON;
```

---

## 5. Change Buffer（变更缓冲）

### 5.1 Change Buffer 的作用

Change Buffer 是 Buffer Pool 的一部分，用于**优化非唯一二级索引的插入、更新、删除操作**：

```
场景：向表中插入一行数据

┌─────────────────────────────────────────────────────────────────┐
│  无 Change Buffer（传统方式）                                     │
│  ────────────────────────────                                    │
│  1. 读取聚簇索引页到 Buffer Pool                                  │
│  2. 读取二级索引页 A 到 Buffer Pool（可能触发磁盘 I/O）            │
│  3. 读取二级索引页 B 到 Buffer Pool（可能触发磁盘 I/O）            │
│  4. 修改聚簇索引页                                                │
│  5. 修改二级索引页 A                                              │
│  6. 修改二级索引页 B                                              │
│  7. 生成 Redo Log，事务提交                                        │
│                                                                  │
│  问题：随机 I/O 多，每个二级索引页都要读磁盘                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  有 Change Buffer（优化方式）                                     │
│  ───────────────────────────                                     │
│  1. 读取聚簇索引页到 Buffer Pool                                  │
│  2. 修改聚簇索引页                                                │
│  3. 将二级索引变更暂存到 Change Buffer（内存操作，无磁盘 I/O）      │
│  4. 生成 Redo Log，事务提交                                        │
│                                                                  │
│  5. 后续读取二级索引页 A 时（Merge 操作）                          │
│     - 从磁盘读取二级索引页 A                                       │
│     - 将 Change Buffer 中关于页 A 的变更合并到页 A                 │
│     - 现在页 A 才是最新状态                                        │
│                                                                  │
│  优势：批量合并减少随机 I/O，提升写入性能                           │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Change Buffer 的限制

| 适用场景 | 不适用场景 |
|---------|-----------|
| 非唯一二级索引的 INSERT | 唯一索引（需要检查唯一性，必须读页） |
| 非唯一二级索引的 UPDATE | 聚簇索引（必须立即修改） |
| 非唯一二级索引的 DELETE | 空间索引（R-tree） |

### 5.3 Change Buffer 配置

```sql
-- 是否启用 Change Buffer（默认 ALL，全部启用）
-- NONE：禁用  INSERTS：仅插入  DELETES：仅删除  CHANGES：插入+删除  ALL：全部
SET GLOBAL innodb_change_buffering = 'ALL';

-- Change Buffer 占 Buffer Pool 比例（默认 25%，最大 50%）
SET GLOBAL innodb_change_buffer_max_size = 25;

-- 查看 Change Buffer 状态
SHOW ENGINE INNODB STATUS\G
-- 关注：merged operations 和 discarded operations

-- MySQL 8.0 支持 Change Buffer 持久化到系统表空间
-- 重启后无需重新构建
```

---

## 6. Adaptive Hash Index（自适应哈希索引）

### 6.1 AHI 的原理

InnoDB 会监控 B+树索引的访问模式，对**热点页**自动建立哈希索引：

```
场景：频繁通过主键查询某张表

B+树索引查找路径（3 层 B+树）：
  Root Page → Branch Page → Leaf Page → Record
  需要 3 次内存访问（如果页都在 Buffer Pool）

AHI 优化后：
  Hash Table[primary_key] → Leaf Page → Record
  只需 1 次哈希查找 + 1 次内存访问

┌─────────────────────────────────────────────────────────────────┐
│                    Adaptive Hash Index                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  哈希表结构（分区锁，默认 8 个分区）：                              │
│                                                                  │
│  ┌─────────┐    ┌──────────────────────────────────────────┐    │
│  │ Hash Key│ →  │ (table_id, index_id, prefix_bytes)       │    │
│  └─────────┘    └──────────────────────────────────────────┘    │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────┐    ┌──────────────────────────────────────────┐    │
│  │ Hash Val│ →  │ (page_address, record_offset)            │    │
│  └─────────┘    └──────────────────────────────────────────┘    │
│                                                                  │
│  触发条件（同时满足）：                                            │
│  1. 某索引页被连续访问 N 次（默认 100）                            │
│  2. 该页的记录访问模式相同（如都通过主键查）                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 AHI 的优缺点

| 优点 | 缺点 |
|------|------|
| 热点查询从 O(logN) 降到 O(1) | 占用 Buffer Pool 内存（约 1%~5%） |
| 自动建立，无需人工干预 | 高并发写入时，AHI 维护成为瓶颈 |
| 对点查场景性能提升明显 | 范围查询无法使用 AHI |

### 6.3 AHI 配置

```sql
-- 是否启用 AHI（默认 ON）
SET GLOBAL innodb_adaptive_hash_index = ON;

-- AHI 分区数（默认 8，高并发可增大减少锁竞争）
SET GLOBAL innodb_adaptive_hash_index_parts = 8;

-- 查看 AHI 使用情况
SHOW ENGINE INNODB STATUS\G
-- 关注：
-- - Hash searches/s：哈希命中次数
-- - Non-hash searches/s：B+树搜索次数
-- - 比例 > 50% 说明 AHI 有效
```

---

## 7. Buffer Pool 预热与恢复

### 7.1 问题：重启后缓存全失

MySQL 重启后，Buffer Pool 为空，所有查询都要走磁盘，性能断崖式下降：

```
重启前：
  Buffer Pool 命中率 99% → 平均查询响应 1ms
  
重启后：
  Buffer Pool 命中率 0% → 平均查询响应 50ms（磁盘 I/O）
  
需要数小时甚至数天才能恢复到原来的热数据状态
```

### 7.2 MySQL 5.6+ 的 Buffer Pool Dump/Load

```sql
-- 正常关闭时自动 Dump Buffer Pool 的页号列表到磁盘
-- 启动时自动 Load 这些页回 Buffer Pool

-- 查看 Dump 文件
-- ib_buffer_pool 文件位于数据目录

-- 手动 Dump
SET GLOBAL innodb_buffer_pool_dump_now = ON;

-- 手动 Load
SET GLOBAL innodb_buffer_pool_load_now = ON;

-- 查看 Dump/Load 进度
SHOW STATUS LIKE 'Innodb_buffer_pool_dump_status';
SHOW STATUS LIKE 'Innodb_buffer_pool_load_status';

-- 配置参数
-- 关闭时 Dump 的最近使用页比例（默认 25%）
SET GLOBAL innodb_buffer_pool_dump_pct = 40;

-- 启动后自动 Load
SET GLOBAL innodb_buffer_pool_load_at_startup = ON;

-- 关闭时自动 Dump
SET GLOBAL innodb_buffer_pool_dump_at_shutdown = ON;
```

---

## 8. 多 Buffer Pool 实例

### 8.1 为什么要多实例

单个大 Buffer Pool 在高并发下存在**锁竞争**问题：

```
单实例问题：
  所有线程竞争同一个 Buffer Pool 的 Mutex
  高并发时，获取/释放锁成为瓶颈

多实例解决方案：
  将大 Buffer Pool 拆分为多个独立实例
  不同线程访问不同实例，减少锁竞争
```

### 8.2 配置多实例

```sql
-- Buffer Pool 实例数（默认 1，大内存建议 4~8）
-- 必须是 innodb_buffer_pool_size / 1GB 的约数
-- 例如：8GB Buffer Pool，可设为 8 个实例，每个 1GB

[mysqld]
innodb_buffer_pool_size = 8589934592  -- 8GB
innodb_buffer_pool_instances = 8       -- 8 个实例

-- 查看各实例状态
SELECT 
    POOL_ID,
    POOL_SIZE,
    DATABASE_PAGES,
    PAGES_MADE_YOUNG,
    PAGES_READ,
    PAGES_WRITTEN
FROM information_schema.INNODB_BUFFER_POOL_STATS;
```

---

## 9. Buffer Pool 性能监控与优化

### 9.1 核心监控指标

```sql
-- Buffer Pool 整体命中率（应 > 95%，理想 > 99%）
SELECT 
    (1 - (SELECT VARIABLE_VALUE 
          FROM performance_schema.global_status 
          WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
         (SELECT VARIABLE_VALUE 
          FROM performance_schema.global_status 
          WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    ) * 100 AS buffer_pool_hit_rate;

-- 详细状态
SHOW ENGINE INNODB STATUS\G

-- 关键指标解读：
-- Buffer pool hit rate: 999 / 1000（99.9% 命中率，很好）
-- Youngs/s: 1000（每秒晋升到 Young 区的页数）
-- Non-youngs/s: 100（每秒未晋升的页数，扫描导致）
-- Pages read ahead: 50（每秒预读页数）
```

### 9.2 常见问题与优化

| 问题 | 现象 | 解决方案 |
|------|------|---------|
| **命中率低** | < 90% | 增大 innodb_buffer_pool_size |
| **Young/Non-young 比例失衡** | Non-youngs 过高 | 检查是否有全表扫描，优化 SQL |
| **脏页刷盘慢** | checkpoint age 接近 max | 增大 innodb_io_capacity |
| **AHI 竞争** | 高并发下性能下降 | 增大 innodb_adaptive_hash_index_parts |
| **Change Buffer 过大** | 占用过多内存 | 减小 innodb_change_buffer_max_size |

### 9.3 生产环境配置建议

```ini
[mysqld]
# 根据内存大小配置 Buffer Pool
# 32GB 内存服务器示例：
innodb_buffer_pool_size = 24G           # 75% 物理内存
innodb_buffer_pool_instances = 8        # 8 个实例
innodb_buffer_pool_dump_at_shutdown = ON
innodb_buffer_pool_load_at_startup = ON
innodb_buffer_pool_dump_pct = 40        # Dump 40% 的热数据

# LRU 优化
innodb_old_blocks_pct = 37
innodb_old_blocks_time = 1000

# 刷盘优化（SSD 磁盘）
innodb_page_cleaners = 8
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000
innodb_max_dirty_pages_pct = 75
innodb_adaptive_flushing = ON

# AHI 优化
innodb_adaptive_hash_index = ON
innodb_adaptive_hash_index_parts = 8

# Change Buffer
innodb_change_buffer_max_size = 25
```

---

## 10. 面试要点速查

| 问题 | 核心答案 |
|------|---------|
| Buffer Pool 是什么？ | InnoDB 的内存缓存区，缓存数据和索引页，所有读写操作都经过这里 |
| 为什么 InnoDB 要用 16KB 的页？ | 平衡 I/O 效率和内存碎片，太小 I/O 次数多，太大缓存效率低 |
| LRU 算法怎么解决全表扫描污染？ | Young/Old 分区，新页先进 Old 区，停留超过 1 秒才晋升 Young 区 |
| 脏页什么时候刷盘？ | LRU 淘汰、Redo Log 快满、定时刷新、脏页比例超阈值、系统空闲时 |
| Change Buffer 有什么用？ | 缓存非唯一二级索引的变更，合并时减少随机 I/O |
| AHI 是什么？ | 自适应哈希索引，对热点页自动建立哈希索引，加速点查 |
| 重启后 Buffer Pool 怎么恢复？ | MySQL 5.6+ 支持 Dump/Load，关闭时保存页号列表，启动时加载 |

---

**相关知识点** → [[01、MySQL架构与存储引擎|MySQL架构与存储引擎]] | [[../04_日志与高可用/01、日志系统（Redo_Undo_Binlog）|日志系统]] | [[../05_架构与运维/02、性能优化与监控|性能优化与监控]]
