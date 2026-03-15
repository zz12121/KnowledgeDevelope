# MyBatis 架构与核心组件

## 一、MyBatis 是什么

MyBatis 是一款**半自动化持久层框架**，前身是 Apache 的 iBATIS，2010 年更名迁移至 GitHub。它的核心设计理念是**让开发者保留对 SQL 的完全控制权**，同时帮你处理掉 JDBC 的所有样板代码——连接管理、参数设置、结果集转换一概不用手写。

与 Hibernate 这类全自动 ORM 不同，MyBatis 不尝试屏蔽 SQL，而是把 SQL 从 Java 代码中分离出去，放到 XML 或注解里集中管理。这样的设计让 DBA 可以独立优化 SQL，Java 开发者专注业务逻辑，两者互不干扰。

---

## 二、核心组件详解

MyBatis 的架构是典型的分层设计，各组件通过**职责链模式**协同工作。

### SqlSessionFactory（会话工厂）
全局**单例**，是 MyBatis 的入口点，负责创建 SqlSession 实例。**线程安全**，应用启动时创建一次，整个应用生命周期共用。通过 `SqlSessionFactoryBuilder.build(inputStream)` 读取配置文件构建，构建完成后 builder 对象可以丢弃。

### SqlSession（会话）
核心会话接口，提供增删改查 API，代表一次数据库交互。**默认实现 `DefaultSqlSession` 不是线程安全的**，每个线程（每个请求）必须使用独立的 SqlSession，用完后必须关闭。在 Spring 集成中由 `SqlSessionTemplate` 代理，自动绑定到当前线程的事务上下文。

### Executor（执行器）
MyBatis 的调度核心，处理缓存逻辑和实际 SQL 执行。有三种策略：
- **SimpleExecutor**：默认，每次执行创建新 PreparedStatement
- **ReuseExecutor**：复用预编译的 Statement，相同 SQL 频繁执行时减少编译开销
- **BatchExecutor**：缓存多个 SQL 操作，调用 `flushStatements()` 后批量提交，适合大批量写入

### StatementHandler（语句处理器）
封装 JDBC Statement 操作，负责创建 Statement 并设置参数。应用**模板方法模式**，核心实现是 `PreparedStatementHandler`。

### ParameterHandler（参数处理器）
将 Java 对象转换为 JDBC 参数类型，依托 TypeHandler 体系处理各种类型转换（String、Integer、Date、枚举等）。

### ResultSetHandler（结果集处理器）
将 JDBC ResultSet 转换为 Java 对象，通过反射机制填充属性。`DefaultResultSetHandler` 处理嵌套映射（association/collection）时会递归处理关联数据。

### MappedStatement（映射语句）
描述一条 SQL 的全部配置信息，包括 SQL 源码、参数映射、结果映射等，启动时解析 XML 构建，全局存储在 Configuration 中。每个 SQL 语句对应一个 MappedStatement，key 为"接口全限定名.方法名"。

---

## 三、工作流程

1. **启动阶段**：`SqlSessionFactoryBuilder` 读取 `mybatis-config.xml`，构建 `Configuration` 对象，解析所有 Mapper XML 生成 `MappedStatement`，注册到 Configuration
2. **获取 Mapper**：`sqlSession.getMapper(UserMapper.class)` 通过 `MapperProxyFactory` 用 JDK 动态代理生成 Mapper 接口的代理对象
3. **调用方法**：调用 `userMapper.findById(1)` 实际触发 `MapperProxy.invoke()`，通过 `MapperMethod` 找到对应 MappedStatement
4. **执行 SQL**：Executor 创建 StatementHandler → ParameterHandler 设置参数 → JDBC 执行 → ResultSetHandler 映射结果
5. **返回结果**：结果集根据 resultMap 或自动映射规则转换为 Java 对象返回

---

## 四、MyBatis vs Hibernate

两者是持久层框架的不同哲学代表：

**MyBatis** 是半自动化 ORM，开发者手写 SQL，完全掌控查询逻辑，性能优化直接精准，学习曲线平缓。缺点是简单 CRUD 也要写 SQL，代码量较大，数据库移植性较差（SQL 可能含方言）。

**Hibernate** 是全自动化 ORM，通过 HQL/JPQL 操作对象，框架自动生成 SQL，开发效率高，数据库移植性优秀（自动适配方言）。缺点是学习曲线陡峭（实体状态管理、Session 缓存、懒加载等概念较多），复杂查询的 SQL 优化受限，调试时不够直观。

**选型建议**：需要复杂查询、精细性能调优、遗留系统改造的项目选 MyBatis；快速原型开发、对象模型复杂、追求数据库无关性的项目选 Hibernate/JPA。国内互联网项目大多选 MyBatis，因为对 SQL 的掌控力更强。

---

**相关面试题** → [[../../../10_DevelopLanguage/003_ORM/01_MyBatisSubject/02、MyBatis 基础|MyBatis 基础面试题]]
