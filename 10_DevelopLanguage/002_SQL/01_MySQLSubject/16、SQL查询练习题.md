## 练习数据表结构

以下所有题目基于同一套数据表，请先熟悉表结构再作答。

```sql
-- 部门表
CREATE TABLE departments (
    dept_id   INT PRIMARY KEY AUTO_INCREMENT,
    dept_name VARCHAR(50) NOT NULL,
    location  VARCHAR(100)
);

-- 员工表
CREATE TABLE employees (
    emp_id      INT PRIMARY KEY AUTO_INCREMENT,
    emp_name    VARCHAR(50) NOT NULL,
    dept_id     INT,
    manager_id  INT,                          -- 上级主管 emp_id，顶层为 NULL
    salary      DECIMAL(10, 2),
    hire_date   DATE,
    gender      TINYINT(1),                   -- 1=男 0=女
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
);

-- 订单表
CREATE TABLE orders (
    order_id    INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT NOT NULL,
    emp_id      INT,                          -- 负责员工
    order_date  DATE NOT NULL,
    total_amount DECIMAL(12, 2),
    status      VARCHAR(20)                   -- 'pending','paid','cancelled'
);

-- 订单明细表
CREATE TABLE order_items (
    item_id    INT PRIMARY KEY AUTO_INCREMENT,
    order_id   INT NOT NULL,
    product_id INT NOT NULL,
    quantity   INT NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- 产品表
CREATE TABLE products (
    product_id   INT PRIMARY KEY AUTO_INCREMENT,
    product_name VARCHAR(100) NOT NULL,
    category     VARCHAR(50),
    price        DECIMAL(10, 2),
    stock        INT DEFAULT 0
);

-- 客户表
CREATE TABLE customers (
    customer_id INT PRIMARY KEY AUTO_INCREMENT,
    cust_name   VARCHAR(50),
    city        VARCHAR(50),
    register_date DATE
);

-- 用户操作日志表（大数据量）
CREATE TABLE user_logs (
    log_id      BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id     INT NOT NULL,
    action      VARCHAR(50),
    created_at  DATETIME NOT NULL,
    INDEX idx_user_time (user_id, created_at)
);
```

---

## 基础查询（第 1-10 题）

###### 1. 查询所有员工的姓名和薪资，按薪资从高到低排列

```sql
SELECT emp_name, salary
FROM employees
ORDER BY salary DESC;
```

###### 2. 查询薪资在 8000 到 15000 之间的员工信息

```sql
SELECT emp_id, emp_name, salary
FROM employees
WHERE salary BETWEEN 8000 AND 15000;
```

###### 3. 查询所有部门名称，去掉重复值

```sql
SELECT DISTINCT dept_name
FROM departments;
```

###### 4. 查询姓名中包含"张"字的员工

```sql
SELECT emp_id, emp_name
FROM employees
WHERE emp_name LIKE '%张%';
```

###### 5. 查询没有分配部门的员工（dept_id 为 NULL）

```sql
SELECT emp_id, emp_name
FROM employees
WHERE dept_id IS NULL;
```

###### 6. 查询 2023 年入职的员工数量

```sql
SELECT COUNT(*) AS emp_count
FROM employees
WHERE YEAR(hire_date) = 2023;
```

###### 7. 查询每个部门的员工人数

```sql
SELECT dept_id, COUNT(*) AS emp_count
FROM employees
GROUP BY dept_id;
```

###### 8. 查询员工人数超过 5 人的部门

```sql
SELECT dept_id, COUNT(*) AS emp_count
FROM employees
GROUP BY dept_id
HAVING COUNT(*) > 5;
```

###### 9. 查询薪资最高的前 3 名员工

```sql
SELECT emp_id, emp_name, salary
FROM employees
ORDER BY salary DESC
LIMIT 3;
```

###### 10. 查询每个部门的平均薪资，保留 2 位小数

```sql
SELECT dept_id, ROUND(AVG(salary), 2) AS avg_salary
FROM employees
GROUP BY dept_id;
```

---

## 多表关联（第 11-20 题）

###### 11. 查询每个员工的姓名及其所在部门名称（INNER JOIN）

```sql
SELECT e.emp_name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;
```

###### 12. 查询所有员工及其部门名称，包括没有分配部门的员工（LEFT JOIN）

```sql
SELECT e.emp_name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id;
```

###### 13. 查询没有任何员工的部门（反向 LEFT JOIN）

```sql
SELECT d.dept_id, d.dept_name
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
WHERE e.emp_id IS NULL;
```

###### 14. 查询每笔订单的客户名称、负责员工姓名和订单金额

```sql
SELECT
    c.cust_name,
    e.emp_name,
    o.total_amount
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
LEFT JOIN employees e ON o.emp_id = e.emp_id;
```

###### 15. 查询购买过产品 ID 为 101 的所有客户名称（使用 JOIN）

```sql
SELECT DISTINCT c.cust_name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE oi.product_id = 101;
```

###### 16. 查询每个部门薪资最高的员工姓名和薪资

```sql
-- 方法一：子查询
SELECT e.emp_name, e.dept_id, e.salary
FROM employees e
WHERE e.salary = (
    SELECT MAX(salary)
    FROM employees
    WHERE dept_id = e.dept_id
);

-- 方法二：窗口函数（MySQL 8.0+）
SELECT emp_name, dept_id, salary
FROM (
    SELECT emp_name, dept_id, salary,
           RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rk
    FROM employees
) t
WHERE rk = 1;
```

###### 17. 查询每个产品的总销售额（单价 × 数量之和）

```sql
SELECT
    p.product_id,
    p.product_name,
    SUM(oi.unit_price * oi.quantity) AS total_sales
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name;
```

###### 18. 查询下单金额排名前 10 的客户，显示客户名称和总消费金额

```sql
SELECT
    c.cust_name,
    SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'paid'
GROUP BY c.customer_id, c.cust_name
ORDER BY total_spent DESC
LIMIT 10;
```

###### 19. 查询同时存在于 orders 表和 customers 表中的 customer_id（使用 EXISTS）

```sql
SELECT customer_id, cust_name
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

###### 20. 查询从未下过订单的客户（NOT EXISTS / NOT IN 两种写法）

```sql
-- NOT EXISTS 写法（推荐，性能更优）
SELECT customer_id, cust_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.customer_id
);

-- NOT IN 写法（注意子查询结果中不能有 NULL）
SELECT customer_id, cust_name
FROM customers
WHERE customer_id NOT IN (
    SELECT DISTINCT customer_id FROM orders
);
```

---

## 聚合与分组进阶（第 21-30 题）

###### 21. 统计每个月的订单量和总金额

```sql
SELECT
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    COUNT(*)                          AS order_count,
    SUM(total_amount)                 AS total_amount
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

###### 22. 查询连续 3 天及以上都有订单的日期段

```sql
-- 思路：找出每个订单日期与其"行号差值"，相同差值的日期属于同一连续段
WITH date_rn AS (
    SELECT DISTINCT order_date,
           DATE_SUB(order_date, INTERVAL ROW_NUMBER() OVER (ORDER BY order_date) DAY) AS grp
    FROM orders
)
SELECT MIN(order_date) AS start_date, MAX(order_date) AS end_date,
       DATEDIFF(MAX(order_date), MIN(order_date)) + 1 AS days
FROM date_rn
GROUP BY grp
HAVING days >= 3;
```

###### 23. 查询每个部门中，薪资高于本部门平均薪资的员工

```sql
SELECT e.emp_name, e.dept_id, e.salary
FROM employees e
JOIN (
    SELECT dept_id, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY dept_id
) dept_avg ON e.dept_id = dept_avg.dept_id
WHERE e.salary > dept_avg.avg_sal;
```

###### 24. 查询每个类别销售额最高的产品（每类只取第一名）

```sql
SELECT product_id, product_name, category, total_sales
FROM (
    SELECT
        p.product_id,
        p.product_name,
        p.category,
        SUM(oi.unit_price * oi.quantity) AS total_sales,
        RANK() OVER (PARTITION BY p.category ORDER BY SUM(oi.unit_price * oi.quantity) DESC) AS rk
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.product_id, p.product_name, p.category
) t
WHERE rk = 1;
```

###### 25. 统计每个客户的累计消费金额（使用窗口函数滚动求和）

```sql
SELECT
    customer_id,
    order_date,
    total_amount,
    SUM(total_amount) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_amount
FROM orders
WHERE status = 'paid'
ORDER BY customer_id, order_date;
```

###### 26. 查询每个员工本月与上月的订单量对比（LAG 函数）

```sql
WITH monthly_orders AS (
    SELECT
        emp_id,
        DATE_FORMAT(order_date, '%Y-%m') AS month,
        COUNT(*) AS order_count
    FROM orders
    GROUP BY emp_id, DATE_FORMAT(order_date, '%Y-%m')
)
SELECT
    emp_id,
    month,
    order_count,
    LAG(order_count, 1, 0) OVER (PARTITION BY emp_id ORDER BY month) AS last_month_count,
    order_count - LAG(order_count, 1, 0) OVER (PARTITION BY emp_id ORDER BY month) AS diff
FROM monthly_orders;
```

###### 27. 查询购买了全部产品类别的客户

```sql
-- 思路：客户购买的类别数 = 产品表中总类别数
SELECT o.customer_id
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY o.customer_id
HAVING COUNT(DISTINCT p.category) = (SELECT COUNT(DISTINCT category) FROM products);
```

###### 28. 查询各部门薪资的中位数

```sql
-- MySQL 8.0+ 无内置 MEDIAN，用百分位近似实现
WITH ranked AS (
    SELECT
        dept_id,
        salary,
        ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary) AS rn,
        COUNT(*) OVER (PARTITION BY dept_id) AS total
    FROM employees
)
SELECT dept_id, AVG(salary) AS median_salary
FROM ranked
WHERE rn IN (FLOOR((total + 1) / 2), CEIL((total + 1) / 2))
GROUP BY dept_id;
```

###### 29. 统计每天新注册用户数和当天累计注册用户总数

```sql
WITH daily_reg AS (
    SELECT register_date, COUNT(*) AS new_users
    FROM customers
    GROUP BY register_date
)
SELECT
    register_date,
    new_users,
    SUM(new_users) OVER (ORDER BY register_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS total_users
FROM daily_reg
ORDER BY register_date;
```

###### 30. 查询连续登录超过 7 天的用户

```sql
-- 利用"日期 - 行号 = 常数"判断连续登录
WITH login_days AS (
    SELECT DISTINCT user_id, DATE(created_at) AS login_date
    FROM user_logs
    WHERE action = 'login'
),
grouped AS (
    SELECT user_id, login_date,
           DATE_SUB(login_date, INTERVAL ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) DAY) AS grp
    FROM login_days
)
SELECT user_id, MIN(login_date) AS start_date, MAX(login_date) AS end_date,
       COUNT(*) AS consecutive_days
FROM grouped
GROUP BY user_id, grp
HAVING consecutive_days >= 7;
```

---

## 子查询与 CTE（第 31-38 题）

###### 31. 查询薪资高于全公司平均薪资的员工

```sql
SELECT emp_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

###### 32. 用 CTE 查询每个部门薪资排名前 2 的员工

```sql
WITH ranked_emp AS (
    SELECT
        emp_id, emp_name, dept_id, salary,
        DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dr
    FROM employees
)
SELECT emp_id, emp_name, dept_id, salary
FROM ranked_emp
WHERE dr <= 2;
```

###### 33. 递归 CTE：查询某员工的完整汇报链路（自顶向下）

```sql
-- 查询从 CEO（manager_id IS NULL）到 emp_id=100 的层级路径
WITH RECURSIVE org_tree AS (
    -- 锚点：找出顶层（无上级）的员工
    SELECT emp_id, emp_name, manager_id, 1 AS level, CAST(emp_name AS CHAR(500)) AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- 递归：向下展开
    SELECT e.emp_id, e.emp_name, e.manager_id, ot.level + 1,
           CONCAT(ot.path, ' -> ', e.emp_name)
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.emp_id
)
SELECT emp_id, emp_name, level, path
FROM org_tree
ORDER BY level, emp_id;
```

###### 34. 查询每个客户最近一次下单的订单信息

```sql
-- 方法一：关联子查询
SELECT o.*
FROM orders o
WHERE o.order_date = (
    SELECT MAX(order_date)
    FROM orders
    WHERE customer_id = o.customer_id
);

-- 方法二：窗口函数（性能更好）
SELECT *
FROM (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
) t
WHERE rn = 1;
```

###### 35. 查询下了订单但订单全部被取消的客户

```sql
SELECT customer_id
FROM customers
WHERE customer_id IN (SELECT DISTINCT customer_id FROM orders)
  AND customer_id NOT IN (
      SELECT DISTINCT customer_id
      FROM orders
      WHERE status != 'cancelled'
  );
```

###### 36. 查询产品销量排名（考虑并列名次，使用 RANK 和 DENSE_RANK）

```sql
SELECT
    product_id,
    product_name,
    total_qty,
    RANK()       OVER (ORDER BY total_qty DESC) AS rank_with_gap,
    DENSE_RANK() OVER (ORDER BY total_qty DESC) AS dense_rank_no_gap
FROM (
    SELECT p.product_id, p.product_name, SUM(oi.quantity) AS total_qty
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.product_id, p.product_name
) t;
```

###### 37. 用 CTE 计算 2023 年每个季度的环比增长率

```sql
WITH quarterly AS (
    SELECT
        QUARTER(order_date) AS quarter,
        SUM(total_amount)   AS revenue
    FROM orders
    WHERE YEAR(order_date) = 2023
      AND status = 'paid'
    GROUP BY QUARTER(order_date)
)
SELECT
    quarter,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY quarter) AS prev_revenue,
    ROUND(
        (revenue - LAG(revenue, 1) OVER (ORDER BY quarter))
        / LAG(revenue, 1) OVER (ORDER BY quarter) * 100,
        2
    ) AS growth_rate_pct
FROM quarterly;
```

###### 38. 查询至少下过 3 笔订单且每笔金额都超过 1000 的客户

```sql
SELECT customer_id
FROM orders
GROUP BY customer_id
HAVING COUNT(*) >= 3
   AND MIN(total_amount) > 1000;
```

---

## 复杂场景（第 39-50 题）

###### 39. 行转列：统计每个部门男女员工人数（CASE WHEN 实现 PIVOT）

```sql
SELECT
    dept_id,
    SUM(CASE WHEN gender = 1 THEN 1 ELSE 0 END) AS male_count,
    SUM(CASE WHEN gender = 0 THEN 1 ELSE 0 END) AS female_count
FROM employees
GROUP BY dept_id;
```

###### 40. 列转行：将多列薪资区间数据展开为行（UNION ALL 实现 UNPIVOT）

```sql
-- 假设有汇总表 salary_stats(dept_id, low_salary, mid_salary, high_salary)
-- 将三列展开为 (dept_id, salary_level, amount)
SELECT dept_id, 'low'  AS salary_level, low_salary  AS amount FROM salary_stats
UNION ALL
SELECT dept_id, 'mid'  AS salary_level, mid_salary  AS amount FROM salary_stats
UNION ALL
SELECT dept_id, 'high' AS salary_level, high_salary AS amount FROM salary_stats
ORDER BY dept_id, salary_level;
```

###### 41. 分页查询：第 3 页数据（每页 10 条），使用延迟关联优化大偏移量

```sql
-- 普通写法（深分页时性能差，offset 很大时扫描量大）
SELECT * FROM orders ORDER BY order_id LIMIT 20, 10;

-- 延迟关联优化（先用覆盖索引取 ID，再回表取数据）
SELECT o.*
FROM orders o
JOIN (
    SELECT order_id FROM orders ORDER BY order_id LIMIT 20, 10
) t ON o.order_id = t.order_id;

-- 游标分页（已知上一页最后一条 ID，性能最优）
SELECT * FROM orders
WHERE order_id > 12345  -- 上一页最后一条的 order_id
ORDER BY order_id
LIMIT 10;
```

###### 42. 查询近 7 天每天的活跃用户数（含没有数据的日期补零）

```sql
-- 先生成近 7 天的日期序列（MySQL 8.0 递归 CTE）
WITH RECURSIVE date_series AS (
    SELECT CURDATE() - INTERVAL 6 DAY AS dt
    UNION ALL
    SELECT dt + INTERVAL 1 DAY FROM date_series WHERE dt < CURDATE()
)
SELECT
    ds.dt,
    COUNT(DISTINCT ul.user_id) AS active_users
FROM date_series ds
LEFT JOIN user_logs ul
    ON DATE(ul.created_at) = ds.dt
    AND ul.action = 'login'
GROUP BY ds.dt
ORDER BY ds.dt;
```

###### 43. 找出重复数据：查询 employees 表中 emp_name 相同的记录

```sql
-- 方法一：找出重复的姓名
SELECT emp_name, COUNT(*) AS cnt
FROM employees
GROUP BY emp_name
HAVING cnt > 1;

-- 方法二：查出完整的重复行（保留所有重复记录）
SELECT *
FROM employees
WHERE emp_name IN (
    SELECT emp_name FROM employees
    GROUP BY emp_name HAVING COUNT(*) > 1
)
ORDER BY emp_name;
```

###### 44. 删除重复数据，只保留 emp_id 最小的一条

```sql
-- 先查出要删除的记录（验证）
SELECT emp_id FROM employees
WHERE emp_id NOT IN (
    SELECT MIN(emp_id) FROM employees GROUP BY emp_name
);

-- 删除（MySQL 不支持在同一语句 DELETE 和 SELECT 同一张表，需要子查询包一层）
DELETE FROM employees
WHERE emp_id NOT IN (
    SELECT min_id FROM (
        SELECT MIN(emp_id) AS min_id FROM employees GROUP BY emp_name
    ) t
);
```

###### 45. 查询间隔超过 30 天下单的客户（找出下单时间间隔较大的记录）

```sql
-- 用 LAG 获取每个客户的上一次下单日期，计算间隔
SELECT customer_id, order_date, prev_date,
       DATEDIFF(order_date, prev_date) AS gap_days
FROM (
    SELECT customer_id, order_date,
           LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_date
    FROM orders
) t
WHERE DATEDIFF(order_date, prev_date) > 30;
```

###### 46. 更新订单金额：将金额低于 100 的 pending 订单更新为标记 status='cancelled'

```sql
UPDATE orders
SET status = 'cancelled'
WHERE status = 'pending'
  AND total_amount < 100;

-- 如果需要关联其他表条件（多表 UPDATE）
UPDATE orders o
JOIN customers c ON o.customer_id = c.customer_id
SET o.status = 'cancelled'
WHERE o.status = 'pending'
  AND o.total_amount < 100
  AND c.city = '北京';
```

###### 47. 用单条 INSERT ... ON DUPLICATE KEY UPDATE 实现 upsert（插入或更新）

```sql
-- 如果 product_id 已存在则更新库存，否则插入新记录
INSERT INTO products (product_id, product_name, stock)
VALUES (101, '商品A', 50)
ON DUPLICATE KEY UPDATE stock = stock + VALUES(stock);

-- MySQL 8.0.20+ 推荐写法（VALUES() 函数已废弃）
INSERT INTO products (product_id, product_name, stock)
VALUES (101, '商品A', 50) AS new_row
ON DUPLICATE KEY UPDATE stock = products.stock + new_row.stock;
```

###### 48. 统计每个员工处理的订单总金额，并与部门平均值比较，输出超出或低于的百分比

```sql
SELECT
    e.emp_id,
    e.emp_name,
    e.dept_id,
    emp_total.total_handled,
    dept_avg.dept_avg_handled,
    ROUND(
        (emp_total.total_handled - dept_avg.dept_avg_handled)
        / dept_avg.dept_avg_handled * 100,
        2
    ) AS pct_vs_dept_avg
FROM employees e
JOIN (
    SELECT emp_id, SUM(total_amount) AS total_handled
    FROM orders
    GROUP BY emp_id
) emp_total ON e.emp_id = emp_total.emp_id
JOIN (
    SELECT e2.dept_id, AVG(emp_sum.total_handled) AS dept_avg_handled
    FROM employees e2
    JOIN (
        SELECT emp_id, SUM(total_amount) AS total_handled
        FROM orders GROUP BY emp_id
    ) emp_sum ON e2.emp_id = emp_sum.emp_id
    GROUP BY e2.dept_id
) dept_avg ON e.dept_id = dept_avg.dept_id
ORDER BY e.dept_id, pct_vs_dept_avg DESC;
```

###### 49. 用事务 + 锁保证并发场景下的库存扣减（SELECT ... FOR UPDATE）

```sql
-- 业务场景：下单时扣减库存，并发时需要防止超卖
START TRANSACTION;

-- 加行级排他锁，防止并发修改
SELECT stock FROM products WHERE product_id = 101 FOR UPDATE;

-- 检查库存是否充足（应用层判断后执行）
UPDATE products
SET stock = stock - 5
WHERE product_id = 101 AND stock >= 5;

-- 如果影响行数为 0 说明库存不足，应用层 ROLLBACK
COMMIT;
```

###### 50. 综合查询：月度销售报表（部门 + 员工 + 月份 + 金额 + 环比 + 排名）

```sql
WITH monthly_sales AS (
    SELECT
        e.dept_id,
        d.dept_name,
        o.emp_id,
        e.emp_name,
        DATE_FORMAT(o.order_date, '%Y-%m') AS month,
        SUM(o.total_amount) AS monthly_amount
    FROM orders o
    JOIN employees e ON o.emp_id = e.emp_id
    JOIN departments d ON e.dept_id = d.dept_id
    WHERE o.status = 'paid'
    GROUP BY e.dept_id, d.dept_name, o.emp_id, e.emp_name,
             DATE_FORMAT(o.order_date, '%Y-%m')
),
with_lag AS (
    SELECT *,
           LAG(monthly_amount) OVER (
               PARTITION BY emp_id ORDER BY month
           ) AS prev_month_amount
    FROM monthly_sales
)
SELECT
    dept_name,
    emp_name,
    month,
    monthly_amount,
    prev_month_amount,
    ROUND(
        (monthly_amount - prev_month_amount) / prev_month_amount * 100,
        2
    ) AS mom_growth_pct,
    RANK() OVER (PARTITION BY dept_id, month ORDER BY monthly_amount DESC) AS dept_rank
FROM with_lag
ORDER BY dept_id, month, dept_rank;
```

---

**知识库索引**
- [[../../../21_MySqlKnowledge/01_MySQL基础/03、SQL语言与执行流程|📖 SQL语言与执行流程]]
- [[../../../21_MySqlKnowledge/03_索引与查询优化/01、索引原理与设计|📖 索引原理与设计]]
- [[../../../21_MySqlKnowledge/03_索引与查询优化/02、查询优化与执行计划|📖 查询优化与执行计划]]
