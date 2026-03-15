###### 1. JDBC 有几个步骤？
[[22_MybatisKnowledge/01_JDBC与MyBatis基础/01、JDBC核心机制#1. JDBC 七步流程|📖]] JDBC操作数据库包含**七个核心步骤**，这是所有JDBC应用程序的基础框架：
1. **加载数据库驱动**：通过`Class.forName()`动态加载并注册特定的JDBC驱动类
2. **建立数据库连接**：使用`DriverManager.getConnection()`获取Connection对象，包含URL、用户名、密码
3. **创建Statement对象**：通过Connection对象创建Statement、PreparedStatement或CallableStatement
4. **执行SQL语句**：使用Statement对象执行查询（executeQuery）或更新（executeUpdate）
5. **处理结果集**：对ResultSet进行遍历，提取查询结果数据
6. **关闭结果集**：显式关闭ResultSet释放资源
7. **关闭Statement和Connection**：按创建顺序的逆序关闭所有数据库资源

从源码角度看，`DriverManager.getConnection()`内部会遍历所有已注册的驱动（通过`Class.forName()`加载时自动注册），依次尝试建立连接。建立连接的本质是 TCP 三次握手加上数据库认证握手协议，所以这是一个很"重"的操作，这也是连接池存在的根本原因。

###### 2. JDBC 操作数据库的核心对象有哪些？各自的作用是什么？
[[22_MybatisKnowledge/01_JDBC与MyBatis基础/01、JDBC核心机制#2. 核心对象|📖]] JDBC API 的核心对象构成了一个完整的数据访问层级体系。**DriverManager** 是驱动管理器，负责加载驱动和建立连接，内部维护一个 `registeredDrivers` 列表，根据 JDBC URL 的协议前缀（如 `jdbc:mysql:`）匹配合适的驱动实现。

**Connection** 代表一次数据库会话，底层对应一条物理 TCP 连接，是事务操作的基石——`commit()`、`rollback()`、`setAutoCommit()` 都是 Connection 的方法。

**Statement** 是静态 SQL 执行器，每次执行都要重新编译，且参数通过字符串拼接传入，存在 SQL 注入风险。**PreparedStatement** 是预编译 SQL 执行器，调用 `connection.prepareStatement(sql)` 时就把 SQL 发送到数据库进行编译并缓存执行计划，后续调用 `setXxx()` 传入的参数只作为纯数据处理，不会改变 SQL 语义，这是它防 SQL 注入的根本原理。**CallableStatement** 继承自 PreparedStatement，专门用于调用存储过程，支持 IN/OUT 参数。

**ResultSet** 是结果集游标，初始位置在第一行之前，通过 `next()` 向下移动，`getXxx()` 获取当前行的字段值。

整个 JDBC API 采用的是**桥接模式**，抽象层（JDBC 接口）与实现层（各数据库驱动）解耦，应用程序无需关心底层是 MySQL 还是 Oracle。

###### 3. JDBC 中 Statement 和 PreparedStatement 的区别是什么？
[[22_MybatisKnowledge/01_JDBC与MyBatis基础/01、JDBC核心机制#3. PreparedStatement 预编译|📖]] 两者最本质的区别是**有没有预编译**。

**Statement** 每次执行都把完整 SQL 字符串发给数据库，数据库每次都要重新解析、编译、执行，效率低。更危险的是，如果参数通过字符串拼接传入，用户可以构造 `' OR '1'='1` 这样的字符串改变 SQL 语义，产生 SQL 注入漏洞。

**PreparedStatement** 在创建时（`connection.prepareStatement(sql)`）就把带占位符 `?` 的 SQL 模板发送到数据库预编译并缓存执行计划。后续只需调用 `setXxx()` 传入参数值，参数值被视为纯数据，即使包含单引号等特殊字符也会被转义处理，不可能改变已经确定的 SQL 结构。相同 SQL 多次执行时，可以复用已编译的执行计划，性能显著高于 Statement。

**最佳实践**：生产环境一律使用 PreparedStatement，只有执行 DDL（CREATE TABLE、ALTER TABLE 等）或确定安全的静态 SQL 时才考虑 Statement。

###### 4. JDBC 如何处理事务？
[[22_MybatisKnowledge/01_JDBC与MyBatis基础/01、JDBC核心机制#4. JDBC 事务管理|📖]] JDBC 事务通过 **Connection** 对象控制。

默认情况下，JDBC 是自动提交模式（`autoCommit=true`），每条 SQL 执行后立即提交。要手动管理事务，需要先调用 `connection.setAutoCommit(false)` 关闭自动提交，然后执行一系列 SQL，最后 `commit()` 提交或 `rollback()` 回滚。

```java
connection.setAutoCommit(false);
try {
    statement.executeUpdate("UPDATE accounts SET balance=balance-100 WHERE id=1");
    statement.executeUpdate("UPDATE accounts SET balance=balance+100 WHERE id=2");
    connection.commit();  // 两步操作全部成功，一起提交
} catch (SQLException e) {
    connection.rollback();  // 任何一步失败，全部撤销
} finally {
    connection.setAutoCommit(true);  // 恢复自动提交模式
}
```

对于复杂事务，还可以用**保存点（Savepoint）** 实现部分回滚——在事务中间 `setSavepoint()`，出错时只回滚到保存点而不是回滚整个事务，保存点之前的操作仍然可以提交。

事务隔离级别通过 `connection.setTransactionIsolation()` 设置，从低到高依次是 READ_UNCOMMITTED（可能脏读）、READ_COMMITTED（避免脏读）、REPEATABLE_READ（避免不可重复读）、SERIALIZABLE（完全串行化）。

从源码角度看，MySQL Connector/J 驱动中，`setAutoCommit(false)` 会向数据库发送 `BEGIN` 命令，`commit()`/`rollback()` 分别对应数据库的 `COMMIT`/`ROLLBACK` 命令。

###### 5. JDBC 连接池的作用是什么？常见的连接池有哪些？
[[22_MybatisKnowledge/01_JDBC与MyBatis基础/01、JDBC核心机制#5. 连接池原理|📖]] 数据库连接的创建是一个"昂贵操作"——需要经过 TCP 三次握手、数据库认证、分配服务器上下文等一系列步骤，耗时通常在几毫秒到几十毫秒级别。如果每次 SQL 请求都新建连接、用完就关，在高并发场景下系统会被这些开销拖垮。

连接池的核心思路是：**提前创建一批连接放在"池"里，用的时候取出来，用完归还而不是关闭**。本质上是**包装器模式**，连接池将真实的 Connection 包装成代理对象，重写了 `close()` 方法，调用 `close()` 时实际上是将连接归还到池中。

主流连接池中，**HikariCP** 是 Spring Boot 的默认选择，以极致性能著称，代码非常精简（约130KB），底层通过字节码优化和无锁设计实现极低的连接获取延迟。**Druid** 是阿里巴巴开源的连接池，最大特点是内置了完善的监控（SQL统计、慢SQL分析）、防 SQL 注入（Wall Filter）等功能，在国内企业项目中使用非常广泛。追求极致性能选 HikariCP，需要强大监控和安全防护选 Druid。
