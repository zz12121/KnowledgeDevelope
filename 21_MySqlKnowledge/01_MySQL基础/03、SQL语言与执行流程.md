---
tags:
  - MySQL/基础
  - MySQL/SQL
  - MySQL/执行流程
aliases:
  - SQL语句
  - 查询执行流程
  - MySQL解析器
date: 2026-03-18
---

# SQL 语言与执行流程

> 从 SQL 分类到执行原理，建立对数据库操作语言的系统性认知。

---

## 一、SQL 语言分类

### 1.1 四大分类

**DDL（Data Definition Language，数据定义语言）**
```sql
CREATE TABLE users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

ALTER TABLE users ADD COLUMN email VARCHAR(100);
ALTER TABLE users MODIFY COLUMN name VARCHAR(100) NOT NULL;
ALTER TABLE users DROP COLUMN email;

DROP TABLE IF EXISTS temp_table;
TRUNCATE TABLE users;  -- 删除全部数据（不写 binlog 行事件，比 DELETE 快，但无法回滚）
```

**DML（Data Manipulation Language，数据操作语言）**
```sql
INSERT INTO users (name) VALUES ('Alice');
INSERT INTO users (name) VALUES ('Bob'), ('Charlie');  -- 批量插入

UPDATE users SET name = 'Alice_New' WHERE id = 1;
DELETE FROM users WHERE id = 1;

-- 注意：UPDATE/DELETE 无 WHERE 会影响全表，生产操作需要额外谨慎
```

**DQL（Data Query Language，数据查询语言）**
```sql
-- 完整的 SELECT 语法顺序
SELECT  [DISTINCT] column_list       -- ⑥ 投影
FROM    table_name                   -- ① 确定数据源
JOIN    other_table ON condition     -- ② 连接
WHERE   filter_condition             -- ③ 行过滤
GROUP BY group_columns               -- ④ 分组
HAVING  group_filter                 -- ⑤ 组过滤
ORDER BY sort_columns [ASC|DESC]     -- ⑦ 排序
LIMIT   n OFFSET m;                  -- ⑧ 分页
```

**DCL（Data Control Language，数据控制语言）**
```sql
GRANT SELECT, INSERT, UPDATE ON mydb.users TO 'app_user'@'%';
REVOKE INSERT ON mydb.users FROM 'app_user'@'%';
FLUSH PRIVILEGES;
```

**TCL（Transaction Control Language，事务控制）**
```sql
START TRANSACTION;     -- 开启事务（或 BEGIN）
SAVEPOINT sp1;         -- 设置保存点
ROLLBACK TO sp1;       -- 回滚到保存点
COMMIT;                -- 提交事务
ROLLBACK;              -- 完全回滚
```

---

## 二、SQL 执行逻辑顺序

SQL 的书写顺序和执行顺序不同。理解执行顺序对于理解为什么某些写法会报错至关重要：

```
书写顺序：SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
执行顺序：FROM → ON/JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

**实际执行顺序详解**：

1. **FROM / JOIN**：确定数据来源，生成笛卡尔积，再按 ON 条件过滤（形成虚拟表 VT1）
2. **WHERE**：对 VT1 的每一行应用过滤条件（此时聚合函数还未计算，不能用 `WHERE COUNT(*) > 5`）
3. **GROUP BY**：将 VT2 的行按指定列分组（VT3 是分组后的聚合行集）
4. **HAVING**：对分组后的聚合结果再次过滤（此时可以用 `HAVING COUNT(*) > 5`）
5. **SELECT**：从 VT4 中选择指定列，计算表达式（VT5）
6. **DISTINCT**：去除重复行（VT6）
7. **ORDER BY**：排序（VT7，**此时才能使用 SELECT 中的别名**）
8. **LIMIT / OFFSET**：返回最终结果集

**重要推论**：
```sql
-- ❌ 错误：WHERE 中不能用 SELECT 别名（WHERE 早于 SELECT 执行）
SELECT name, age * 2 AS double_age FROM users WHERE double_age > 30;

-- ✅ 正确：用原始表达式
SELECT name, age * 2 AS double_age FROM users WHERE age * 2 > 30;

-- ✅ 正确：ORDER BY 中可以用别名（ORDER BY 晚于 SELECT 执行）
SELECT name, age * 2 AS double_age FROM users ORDER BY double_age DESC;
```

---

## 三、JOIN 类型与原理

### 3.1 JOIN 类型

```sql
-- INNER JOIN：只返回两表都有匹配的行
SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT OUTER JOIN：返回左表所有行，右表无匹配则 NULL
SELECT u.name, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- RIGHT OUTER JOIN：返回右表所有行，左表无匹配则 NULL
-- （可以用 LEFT JOIN + 交换表顺序替代，实践中较少用）

-- CROSS JOIN：笛卡尔积，m×n 行（无 ON 条件）
SELECT a.name, b.name FROM colors a CROSS JOIN sizes b;
```

### 3.2 JOIN 算法（InnoDB 实现）

MySQL 主要使用三种 JOIN 算法：

**① Nested-Loop Join (NLJ)**：最基础的算法
```
for each row r1 in 驱动表（outer table）:
    for each row r2 in 被驱动表（inner table）:
        if join_condition(r1, r2):
            emit(r1, r2)
```
- 复杂度 O(m × n)，被驱动表每次都要扫描
- 如果被驱动表 JOIN 字段有索引，内层循环变成 O(log n)，效率大幅提升
- **结论：确保 JOIN 字段有索引，小表驱动大表**

**② Block Nested-Loop Join (BNL)**：处理没有索引的情况
- 将外层表的多行读入 `join_buffer`（大小由 `join_buffer_size` 控制），一次性与内层表比较
- 减少内层表的扫描次数（变成 `ceil(m/buffer_size)` 次而不是 m 次）
- 仍然是全表扫描，代价高，应尽量通过添加索引来避免

**③ Hash Join（MySQL 8.0.18+）**：
- 先将小表的 JOIN 列构建哈希表，再扫描大表用哈希查找匹配
- 对等值连接效率极高，尤其在无索引情况下
- MySQL 8.0 会自动选择

### 3.3 驱动表的选择原则

**优化器通常会自动选择最优驱动表**，但可以通过以下方式干预：

```sql
-- STRAIGHT_JOIN 强制指定驱动顺序（左表驱动右表）
SELECT STRAIGHT_JOIN u.name, o.amount
FROM users u  -- 驱动表
STRAIGHT_JOIN orders o ON u.id = o.user_id;
```

一般原则：**小结果集驱动大结果集**（经过 WHERE 过滤后）

---

## 四、子查询与优化

### 4.1 子查询类型

```sql
-- 标量子查询：返回单个值
SELECT name, (SELECT COUNT(*) FROM orders WHERE user_id = u.id) AS order_count
FROM users u;

-- IN 子查询
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE amount > 100);

-- EXISTS 子查询（通常比 IN 更高效，尤其子查询结果集大时）
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.amount > 100);

-- 派生表（FROM 子句中的子查询）
SELECT dept_name, avg_salary
FROM (
    SELECT dept, AVG(salary) AS avg_salary FROM employees GROUP BY dept
) AS dept_stats
WHERE avg_salary > 50000;
```

### 4.2 子查询优化

MySQL 优化器会自动尝试将子查询转换为 JOIN（semi-join 优化），但有些情况需要手动干预：

```sql
-- ❌ 低效：相关子查询，每行都执行一次子查询
SELECT * FROM users u
WHERE (SELECT COUNT(*) FROM orders WHERE user_id = u.id) > 5;

-- ✅ 高效：转为 JOIN + GROUP BY
SELECT u.*
FROM users u
INNER JOIN (
    SELECT user_id, COUNT(*) cnt FROM orders GROUP BY user_id HAVING cnt > 5
) o ON u.id = o.user_id;
```

---

## 五、聚合函数与窗口函数

### 5.1 聚合函数

```sql
SELECT
    COUNT(*)          AS total_rows,     -- 统计总行数（包含 NULL 行）
    COUNT(1)          AS count_1,        -- 同 COUNT(*)，无区别
    COUNT(email)      AS has_email,      -- 统计非 NULL 的 email 行数
    COUNT(DISTINCT dept) AS dept_count,  -- 统计不重复的部门数
    SUM(salary)       AS total_salary,
    AVG(salary)       AS avg_salary,
    MAX(salary)       AS max_salary,
    MIN(hire_date)    AS earliest_hire
FROM employees
WHERE dept = 'Engineering';
```

**COUNT(*) vs COUNT(1) vs COUNT(字段)**：
- `COUNT(*)` 和 `COUNT(1)` 性能完全相同（MySQL 8.0 内部等价）
- `COUNT(字段)` 不统计 NULL 值，语义不同，会额外判断 NULL

### 5.2 窗口函数（MySQL 8.0+）

窗口函数在 GROUP BY 无法满足需求时（既需要聚合值，又需要每行原始数据）非常有用：

```sql
-- ROW_NUMBER：每组内的行号
SELECT name, dept, salary,
    ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS rank_in_dept
FROM employees;

-- RANK / DENSE_RANK：排名（RANK 有跳跃，DENSE_RANK 无跳跃）
SELECT name, salary,
    RANK() OVER (ORDER BY salary DESC) AS salary_rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;

-- LAG / LEAD：前后行的值
SELECT name, salary,
    LAG(salary, 1) OVER (PARTITION BY dept ORDER BY hire_date) AS prev_salary,
    LEAD(salary, 1) OVER (PARTITION BY dept ORDER BY hire_date) AS next_salary
FROM employees;

-- SUM 累计求和
SELECT name, salary,
    SUM(salary) OVER (PARTITION BY dept ORDER BY hire_date ROWS UNBOUNDED PRECEDING) AS cumulative_salary
FROM employees;
```

---

## 六、EXPLAIN 执行计划关键字段速览

```sql
EXPLAIN SELECT * FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.age > 18;
```

| 字段 | 重要性 | 关键值说明 |
|------|-------|-----------|
| `type` | ⭐⭐⭐ | `system`>`const`>`eq_ref`>`ref`>`range`>`index`>`ALL`，ALL 需优化 |
| `key` | ⭐⭐⭐ | 实际使用的索引，NULL 表示未走索引 |
| `rows` | ⭐⭐ | 预计扫描行数，越小越好 |
| `Extra` | ⭐⭐ | `Using index`（覆盖索引，好）；`Using filesort`/`Using temporary`（需关注） |
| `key_len` | ⭐ | 索引使用长度，可判断联合索引是否被充分利用 |

---

## 七、面试要点速查

| 问题 | 核心答案 |
|------|---------|
| SQL 的执行顺序？ | FROM→JOIN→WHERE→GROUP BY→HAVING→SELECT→ORDER BY→LIMIT |
| WHERE 和 HAVING 的区别？ | WHERE 在分组前过滤行；HAVING 在分组后过滤聚合结果 |
| COUNT(*) vs COUNT(字段)？ | COUNT(*) 统计所有行；COUNT(字段) 不统计 NULL |
| IN vs EXISTS 哪个更快？ | 无绝对结论；子查询结果集小用 IN，大用 EXISTS；现代优化器差距很小 |
| TRUNCATE 和 DELETE 的区别？ | TRUNCATE 不可回滚、不触发触发器、速度快；DELETE 可回滚、写 binlog、可加 WHERE |

---

**相关面试题** → [[../../10_Developlanguage/002_SQL/01_MySQLSubject/01、MySQL 基础概念]]、[[../../10_Developlanguage/002_SQL/01_MySQLSubject/08、查询优化]]
