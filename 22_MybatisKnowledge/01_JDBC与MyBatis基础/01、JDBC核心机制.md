# JDBC 核心机制

## 一、JDBC 的七个步骤

JDBC 操作数据库有固定的七步流程，是所有数据库访问的基础骨架：

1. **加载数据库驱动**：通过 `Class.forName()` 动态加载并注册驱动类（JDBC 4.0+ 已支持 SPI 自动加载，但显式写出可提升可读性）
2. **建立数据库连接**：调用 `DriverManager.getConnection(url, user, password)` 建立连接（TCP 握手 + 数据库认证）
3. **创建 Statement 对象**：根据需要创建 Statement、PreparedStatement 或 CallableStatement
4. **执行 SQL 语句**：使用 `executeQuery()` 执行查询，`executeUpdate()` 执行增删改
5. **处理结果集**：遍历 ResultSet 提取数据
6. **关闭结果集**：显式调用 `resultSet.close()`
7. **关闭 Statement 和 Connection**：按创建顺序逆序关闭所有资源

**资源关闭要用 try-with-resources 或 finally 块确保执行**，否则会造成连接泄漏。

---

## 二、JDBC 核心对象

JDBC API 采用**桥接模式**，将 JDBC 接口与具体数据库驱动实现分离，应用代码无需感知底层数据库。核心对象各司其职：

**DriverManager** 是驱动管理器，维护一个 `registeredDrivers` 列表，`getConnection()` 时根据 URL 协议匹配合适的 Driver 实现建立连接。

**Connection** 代表一次数据库会话，底层对应一条 TCP 连接，是事务操作的基石。`setAutoCommit(false)` 开启事务，`commit()` / `rollback()` 控制事务边界。

**Statement** 是静态 SQL 执行器，每次执行都需要发送完整 SQL 字符串让数据库重新编译，**存在 SQL 注入风险**，一般只用于执行 DDL 或无参数的静态 SQL。

**PreparedStatement** 是预编译 SQL 执行器，调用 `connection.prepareStatement(sql)` 时会将 SQL 模板发送给数据库预编译并缓存执行计划，后续通过 `setXxx()` 设置的参数作为纯数据处理，无法改变 SQL 结构，从根本上**防止 SQL 注入**。同一 SQL 多次执行只需编译一次，性能更高。**生产环境应一律使用 PreparedStatement**。

**CallableStatement** 继承自 PreparedStatement，用于调用数据库存储过程，通过 `registerOutParameter()` 处理 OUT 参数。

**ResultSet** 是结果集游标，初始位置在第一行之前，调用 `next()` 向前移动，通过 `getXxx(columnName)` 获取当前行数据。

---

## 三、JDBC 事务管理

JDBC 通过 Connection 对象控制事务，默认是自动提交模式（每条 SQL 执行后立即提交）。

**基本事务流程**：
```java
connection.setAutoCommit(false);  // 关闭自动提交，开启事务
try {
    statement.executeUpdate("UPDATE accounts SET balance=balance-100 WHERE id=1");
    statement.executeUpdate("UPDATE accounts SET balance=balance+100 WHERE id=2");
    connection.commit();  // 全部成功则提交
} catch (SQLException e) {
    connection.rollback();  // 任何异常则回滚
} finally {
    connection.setAutoCommit(true);  // 恢复自动提交
}
```

**Savepoint 部分回滚**：对于复杂事务，可以在中间设置保存点，异常时只回滚到保存点，保存点之前的操作仍然提交：
```java
Savepoint sp = connection.setSavepoint("SAVEPOINT_1");
// 出错时：connection.rollback(sp) 只回滚到保存点
```

**隔离级别**：通过 `connection.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED)` 设置，对应 MySQL 的四个隔离级别。

在 MySQL Connector/J 源码中，`setAutoCommit(false)` 向数据库发送 `BEGIN` 命令，`commit()` 发送 `COMMIT`，`rollback()` 发送 `ROLLBACK`。

---

## 四、连接池原理与选型

数据库连接创建涉及 **TCP 三次握手 + MySQL 认证 + 上下文初始化**，单次耗时约 1-5ms。高并发场景下频繁创建销毁连接会成为性能瓶颈，**连接池**通过复用连接解决这个问题。

连接池的本质是**包装器模式**：真实 Connection 被代理包装，`close()` 方法被重写为"归还到池中"而非真正关闭。应用代码无感知地复用已有连接。

**主流连接池对比**：

**HikariCP** 是目前性能最佳的连接池，代码精简（约 130KB），Spring Boot 默认选用。核心设计是无锁的 ConcurrentBag 数据结构，减少线程竞争。关键参数：`maximumPoolSize`（最大连接数，建议 10-20）、`connectionTimeout`（获取连接等待超时，建议 30s）、`idleTimeout`（空闲连接超时，建议 10min）。

**Druid** 是阿里开源的企业级连接池，功能最全面：内置 SQL 监控、Wall 防 SQL 注入、加密连接串等，提供 Web 监控控制台。适合对运维监控有要求的企业项目。

**C3P0** 和 **DBCP** 较老，性能和功能不如前两者，新项目一般不选用。

---

**相关面试题** → [[../../../10_DevelopLanguage/003_ORM/01_MyBatisSubject/01、JDBC 基础|JDBC 基础面试题]]
