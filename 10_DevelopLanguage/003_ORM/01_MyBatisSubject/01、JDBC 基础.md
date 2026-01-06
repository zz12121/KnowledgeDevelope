###### 1. JDBC 有几个步骤？
JDBC操作数据库包含**七个核心步骤**，这是所有JDBC应用程序的基础框架：
1. **加载数据库驱动**：通过`Class.forName()`动态加载并注册特定的JDBC驱动类
2. **建立数据库连接**：使用`DriverManager.getConnection()`获取Connection对象，包含URL、用户名、密码
3. **创建Statement对象**：通过Connection对象创建Statement、PreparedStatement或CallableStatement
4. **执行SQL语句**：使用Statement对象执行查询（executeQuery）或更新（executeUpdate）
5. **处理结果集**：对ResultSet进行遍历，提取查询结果数据
6. **关闭结果集**：显式关闭ResultSet释放资源
7. **关闭Statement和Connection**：按创建顺序的逆序关闭所有数据库资源
**源码角度分析**：在`java.sql.DriverManager`的`getConnection()`方法中，会遍历所有已注册的驱动（通过`Class.forName()`加载时自动注册），尝试建立连接。建立连接的过程实质上是TCP三次握手+数据库认证握手协议。
###### 2. JDBC 操作数据库的核心对象有哪些？各自的作用是什么？
JDBC API的核心对象构成了一个完整的数据访问层级体系：

|**核心对象**​|**职责与作用**​|**重要方法**​|**源码级职责**​|
|---|---|---|---|
|**DriverManager**​|驱动管理器，负责加载和注册驱动，建立连接|`getConnection()`, `registerDriver()`|维护`registeredDrivers`列表，基于URL协议匹配选择合适的Driver实现|
|**Connection**​|数据库连接会话，代表一个TCP连接，是事务操作的基石|`createStatement()`, `prepareStatement()`, `setAutoCommit()`, `commit()`, `rollback()`|底层对应物理TCP连接，管理事务状态和隔离级别|
|**Statement**​|静态SQL执行器，用于执行不带参数的SQL|`executeQuery()`, `executeUpdate()`, `execute()`|维护SQL文本，每次执行都需要编译，易导致SQL注入|
|**PreparedStatement**​|预编译SQL执行器，**防止SQL注入**，支持参数化查询|`setXxx()`, `executeQuery()`, `executeUpdate()`|预编译SQL模板（在`connection.prepareStatement()`时完成），参数仅作为数据传递|
|**CallableStatement**​|存储过程调用器，继承自PreparedStatement|`registerOutParameter()`, `getXxx()`|处理IN/OUT参数，调用数据库存储过程|
|**ResultSet**​|结果集游标，提供对查询结果的迭代访问|`next()`, `getXxx()`, `first()`, `last()`|维护指向当前数据行的游标，初始位置在第一行之前|
**设计理念**：JDBC采用**桥接模式**，将抽象（JDBC API）与实现（具体数据库驱动）分离，使得应用程序可以独立于特定数据库。
###### 3. JDBC 中 Statement 和 PreparedStatement 的区别是什么？
两者的区别体现在多个关键维度：

|**对比维度**​|**Statement**​|**PreparedStatement**​|
|---|---|---|
|**SQL注入防护**​|**不安全**，通过字符串拼接易受攻击|**安全**，参数化查询从根本上防止注入|
|**性能表现**​|每次执行都需要**编译解析**，效率低|**预编译一次**，多次执行，性能极高|
|**SQL语法**​|纯字符串拼接，易出错|参数化查询，使用`?`占位符|
|**可读性**​|复杂SQL拼接可读性差|结构清晰，易于维护|
|**适用场景**​|DDL操作、临时简单查询|业务系统核心DML操作，特别是带参查询|
**源码角度分析**：
- **预编译机制**：当调用`connection.prepareStatement(sql)`时，JDBC驱动会将SQL模板发送到数据库服务器进行**预编译**，生成执行计划并缓存。后续调用`setXxx()`设置参数时，参数值作为纯数据处理，不会改变执行计划结构
- **防注入原理**：恶意参数如`' OR '1'='1`通过`setString()`方法设置时，会被转义为普通字符串数据，无法改变原有SQL语义。而Statement直接拼接会改变SQL逻辑
**最佳实践**：**生产环境一律使用PreparedStatement**，只有在执行DDL或确定安全的静态SQL时才考虑Statement。
###### 4. JDBC 如何处理事务？
JDBC事务管理是确保数据一致性的核心机制，主要通过Connection对象控制：
**1. 手动事务管理基本流程**
```java
// 1. 关闭自动提交，开启事务
connection.setAutoCommit(false);  // 关键步骤[1,12](@ref)

try {
    // 2. 执行多个DML操作
    String sql1 = "UPDATE accounts SET balance=balance-100 WHERE id=1";
    statement.executeUpdate(sql1);
    
    String sql2 = "UPDATE accounts SET balance=balance+100 WHERE id=2"; 
    statement.executeUpdate(sql2);
    
    // 3. 提交事务
    connection.commit();  // 所有操作持久化[1,12](@ref)
} catch (SQLException e) {
    // 4. 异常时回滚
    connection.rollback();  // 撤销所有操作[1,12](@ref)
} finally {
    connection.setAutoCommit(true);  // 恢复自动提交模式[12](@ref)
}
```
**2. 保存点（Savepoint）机制**
对于复杂事务，可以在事务内设置保存点进行部分回滚：
```java
Savepoint savepoint = null;
try {
    // 第一部分操作
    statement.executeUpdate("INSERT INTO log VALUES(...)");
    
    savepoint = connection.setSavepoint("SAVEPOINT_1");  // 设置保存点[12](@ref)
    
    // 第二部分操作
    statement.executeUpdate("UPDATE accounts SET ...");
    
    connection.commit();
} catch (SQLException e) {
    if(savepoint != null) {
        connection.rollback(savepoint);  // 仅回滚到保存点[12](@ref)
        connection.commit();  // 提交保存点之前的操作
    } else {
        connection.rollback();  // 全事务回滚
    }
}
```
**3. 事务隔离级别**
通过`connection.setTransactionIsolation()`设置，控制事务间的可见性：
- `READ_UNCOMMITTED`：可能脏读
- `READ_COMMITTED`：避免脏读（最常用）
- `REPEATABLE_READ`：避免不可重复读
- `SERIALIZABLE`：完全串行化（性能最低）
**源码角度**：在MySQL Connector/J驱动中，`setAutoCommit(false)`会向数据库发送`BEGIN`命令启动事务，`commit()`/`rollback()`对应`COMMIT`/`ROLLBACK`命令。
###### 5. JDBC 连接池的作用是什么？常见的连接池有哪些？
**连接池的核心价值**：
数据库连接创建是**昂贵操作**（TCP三次握手、认证、上下文分配）。连接池通过**复用连接**极大提升性能：
- **降低延迟**：避免频繁创建/关闭连接
- **控制资源**：防止连接数耗尽导致系统崩溃
- **统一管理**：提供监控、故障转移等高级功能
**主流连接池对比**：

|**连接池**​|**特点**​|**适用场景**​|
|---|---|---|
|**HikariCP**​|**高性能**，代码精简（130KB），监控完备|**Spring Boot默认**，追求极致性能的现代应用|
|**Druid**​|功能全面（SQL监控、防御注入、加密）|需要监控和防护的企业级应用|
|**C3P0**​|稳定可靠，但性能较差|传统老项目，稳定性要求高|
|**DBCP**​|Apache项目，功能基础|Tomcat内置，简单Web应用|
**HikariCP配置示例**：
```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
config.setUsername("user");
config.setPassword("password");
config.setMaximumPoolSize(20);  // 最大连接数[1](@ref)
config.setMinimumIdle(5);      // 最小空闲连接
config.setConnectionTimeout(30000);  // 连接超时[1](@ref)

HikariDataSource dataSource = new HikariDataSource(config);
Connection conn = dataSource.getConnection();  // 从池中获取[1,2](@ref)
```
**源码角度**：连接池本质是**包装器模式**，真实的Connection被代理包装，`close()`方法被重写为"归还到池中"而非真正关闭。