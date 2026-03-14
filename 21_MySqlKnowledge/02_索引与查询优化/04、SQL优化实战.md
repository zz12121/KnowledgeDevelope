# SQL 优化实战

> 把索引理论和执行计划知识落地到实际 SQL 优化场景，掌握分页、大表查询、JOIN、COUNT 等高频问题的最佳实践。

---

## 一、常见 SQL 优化技巧总览

### 1.1 基本原则

```sql
-- ❌ 避免 SELECT *：浪费带宽，阻止覆盖索引优化
SELECT * FROM users WHERE city = 'Beijing';

-- ✅ 只查需要的字段
SELECT id, name, phone FROM users WHERE city = 'Beijing';

-- ❌ 避免在 WHERE 中对索引列做运算
WHERE DATE(created_at) = '2024-01-01'
WHERE YEAR(created_at) = 2024

-- ✅ 改为范围查询（可以利用索引）
WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02'
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'

-- ❌ 隐式类型转换导致索引失效
WHERE phone = 13800138000   -- phone 是 VARCHAR

-- ✅ 类型匹配
WHERE phone = '13800138000'
```

### 1.2 UNION ALL vs UNION

```sql
-- ❌ UNION 会去重，需要额外排序操作
SELECT id FROM orders_2024 WHERE status = 1
UNION
SELECT id FROM orders_2025 WHERE status = 1;

-- ✅ 如果业务允许重复（或确定不重复），用 UNION ALL
SELECT id FROM orders_2024 WHERE status = 1
UNION ALL
SELECT id FROM orders_2025 WHERE status = 1;
```

---

## 二、分页查询优化

### 2.1 大偏移量分页的问题

```sql
-- 表有 1000 万行，查第 100 万页
SELECT id, name FROM users ORDER BY id LIMIT 10000000, 20;
-- 问题：MySQL 需要扫描 10,000,020 行，丢弃前 10,000,000 行，只返回 20 行
-- 即使有索引也需要大量回表，性能极差
```

### 2.2 延迟关联（推荐）

```sql
-- 先用覆盖索引快速定位 ID，再回表
SELECT u.* FROM users u
INNER JOIN (
    SELECT id FROM users ORDER BY id LIMIT 10000000, 20
) t ON u.id = t.id;
-- 子查询走覆盖索引（只查 id），大偏移量跳过的行无需回表
-- 性能提升显著
```

### 2.3 游标分页（最推荐）

```sql
-- 记录上一页最后一条记录的 id，下一次查询从这个 id 之后开始
-- 第一页
SELECT id, name FROM users ORDER BY id LIMIT 20;

-- 第二页（假设上一页最后一条 id = 100）
SELECT id, name FROM users WHERE id > 100 ORDER BY id LIMIT 20;

-- 优势：
-- ✅ 无论翻到第几页，查询性能一致
-- ✅ 利用主键索引，查询极快
-- 劣势：
-- ❌ 只能顺序翻页（不能跳页）
-- ❌ 需要业务层记录游标
```

### 2.4 COUNT 优化

```sql
-- 方案1：利用覆盖索引，选择最小的索引统计
SELECT COUNT(*) FROM users WHERE status = 1;
-- 建 (status) 或 (status, 其他字段) 索引，COUNT 走索引无需回表

-- 方案2：使用近似值（大表统计不需要精确值时）
SELECT TABLE_ROWS FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'users';

-- 方案3：维护计数器表（需要精确值）
CREATE TABLE user_counters (
    status TINYINT PRIMARY KEY,
    cnt BIGINT NOT NULL DEFAULT 0
);
-- INSERT/DELETE 时同步更新计数器（配合事务）
```

---

## 三、JOIN 查询优化

### 3.1 小表驱动大表

```sql
-- 假设 departments（100行）和 employees（100万行）
-- ✅ 小表 departments 驱动大表 employees
SELECT d.name, COUNT(e.id)
FROM departments d    -- 驱动表（小表）
LEFT JOIN employees e ON e.dept_id = d.id
GROUP BY d.id;

-- 确保被驱动表的 JOIN 字段有索引
CREATE INDEX idx_dept_id ON employees(dept_id);
```

### 3.2 避免 Using join buffer

```sql
-- EXPLAIN 中出现 "Using join buffer" 表示被驱动表没有走索引
-- 解决方案：给被驱动表的 JOIN 字段加索引
ALTER TABLE orders ADD INDEX idx_user_id (user_id);
```

### 3.3 多表 JOIN 的顺序控制

```sql
-- 复杂多表 JOIN，优化器可能选错顺序
-- 可用 EXPLAIN 确认，必要时用 STRAIGHT_JOIN 强制顺序
SELECT STRAIGHT_JOIN u.name, o.amount, p.product_name
FROM (
    SELECT id, user_id FROM orders WHERE status = 1 AND created_at > '2024-01-01'
) o
JOIN users u ON u.id = o.user_id       -- 先把 orders 结果集缩小再 JOIN
JOIN products p ON p.id = o.product_id;
```

---

## 四、大表查询优化

### 4.1 分批处理（避免锁表和大事务）

```sql
-- ❌ 一次更新 1000 万行（长事务、锁时间长）
UPDATE orders SET status = 3 WHERE status = 1 AND created_at < '2020-01-01';

-- ✅ 分批更新（每批 1000 行）
REPEAT
    UPDATE orders SET status = 3 
    WHERE status = 1 AND created_at < '2020-01-01'
    LIMIT 1000;
    
    SELECT SLEEP(0.1);  -- 适当暂停，给主库喘息时间
    
UNTIL ROW_COUNT() = 0 END REPEAT;
```

### 4.2 避免全表扫描的 DELETE

```sql
-- ❌ 删除大量历史数据，一次 DELETE 过多行
DELETE FROM logs WHERE created_at < '2023-01-01';

-- ✅ 分批删除
DELETE FROM logs WHERE created_at < '2023-01-01' LIMIT 10000;
-- 循环执行，直到 affected rows = 0

-- ✅ 更优：先用新表 + RENAME 替换（对大表）
CREATE TABLE logs_new LIKE logs;
INSERT INTO logs_new SELECT * FROM logs WHERE created_at >= '2023-01-01';
RENAME TABLE logs TO logs_archive, logs_new TO logs;
```

### 4.3 表分区（Partitioning）

```sql
-- 按时间范围分区，大幅提升时间范围查询性能
CREATE TABLE orders (
    id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)  -- 分区键必须包含在主键中
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- 查询只扫描对应分区（分区裁剪）
SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
-- EXPLAIN 中 partitions 字段会显示只访问了 p2024

-- 删除历史分区（极快，直接删除文件）
ALTER TABLE orders DROP PARTITION p2022;
```

---

## 五、慢查询定位与优化流程

### 5.1 开启慢查询日志

```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 动态开启（重启失效）
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;           -- 超过 1 秒记录
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未走索引的查询

-- 持久化（写入 my.cnf）
[mysqld]
slow_query_log = 1
long_query_time = 1
slow_query_log_file = /var/log/mysql/slow.log
log_queries_not_using_indexes = 1
```

### 5.2 分析慢查询日志

```bash
# 使用 mysqldumpslow（MySQL 内置）
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
# -s t：按查询时间排序
# -t 10：显示前 10 条

# 使用 pt-query-digest（更强大，Percona Toolkit）
pt-query-digest /var/log/mysql/slow.log | head -200
# 输出按总耗时排名的 TOP SQL，包含平均时间、执行次数等
```

### 5.3 优化流程

```
1. 慢查询日志 / Performance Schema 发现慢 SQL
         ↓
2. EXPLAIN 分析执行计划
   - type = ALL？→ 缺索引
   - Using filesort？→ 排序未走索引
   - Using temporary？→ GROUP BY 未走索引
   - rows 过大？→ 索引选择性差或缺索引
         ↓
3. 针对性优化
   - 添加合适索引
   - 改写 SQL（避免函数、改分页策略等）
   - 优化表结构（数据类型、分区等）
         ↓
4. EXPLAIN ANALYZE 验证优化效果
         ↓
5. 上线后监控慢查询
```

---

## 六、索引合并（Index Merge）

某些情况下 MySQL 会同时使用多个索引并合并结果：

```sql
-- 假设 idx_name 和 idx_city 分别独立存在
-- OR 查询时，优化器可能用 Index Merge Union
SELECT * FROM users WHERE name = 'Alice' OR city = 'Beijing';
-- EXPLAIN: type=index_merge, Extra=Using union(idx_name, idx_city)

-- 通常不如建一个联合索引效率高
-- 如果频繁出现此查询，考虑建 (name, city) 联合索引或拆成 UNION ALL
```

---

## 七、面试要点速查

| 场景 | 优化方案 |
|------|---------|
| 大偏移量分页慢 | 延迟关联（先覆盖索引定位ID）或游标分页（WHERE id > last_id） |
| COUNT(*) 慢 | 覆盖索引；维护计数器；用 information_schema 近似值 |
| JOIN 慢 | 小表驱动大表；被驱动表 JOIN 字段必须有索引 |
| 大批量 DELETE/UPDATE | 分批处理（LIMIT + 循环），避免长事务和大锁 |
| 时间范围查询慢 | 表分区（RANGE/LIST PARTITION），查询时自动分区裁剪 |
| 慢查询定位流程 | 慢查询日志 → EXPLAIN 分析 → 添加索引/改写SQL → 验证 |

---

**相关面试题** → [[../../10_Developlanguage/002_SQL/01_MySQLSubject/08、查询优化]]、[[../../10_Developlanguage/002_SQL/01_MySQLSubject/13、性能优化]]
