###### 1. 什么是 MyBatis？它解决了什么问题？
[[22_MybatisKnowledge/01_JDBC与MyBatis基础/02、MyBatis架构与核心组件#1. MyBatis 设计理念|📖]] **MyBatis** 是一款优秀的**半自动化持久层框架**，通过 XML 或注解方式配置 SQL，将 Java 对象与数据库记录进行映射。MyBatis 的前身是 iBATIS，2013 年迁移到 GitHub 后正式更名。

它解决的核心问题是 **JDBC 的冗余代码**：传统 JDBC 需要手动注册驱动、管理连接、设置参数、处理结果集、关闭资源，这些样板代码与业务逻辑完全无关，MyBatis 自动处理这些底层细节。

MyBatis 的设计哲学是"**半自动化**"——不像 Hibernate 那样完全屏蔽 SQL，而是让开发者在享受对象化便利的同时，保留对 SQL 的完全控制权。需要优化某条慢 SQL？直接改 XML 里的 SQL 语句就行，不需要去猜框架自动生成了什么。

###### 2. MyBatis 的核心组件有哪些？
[[22_MybatisKnowledge/01_JDBC与MyBatis基础/02、MyBatis架构与核心组件#2. 七大核心组件|📖]] MyBatis 的架构基于分层设计，七大核心组件各司其职。

**SqlSessionFactory** 是全局单例，线程安全，应用启动时创建一次，之后一直复用，负责生产 SqlSession 实例。**SqlSession** 是核心会话接口，代表一次数据库会话，提供增删改查 API 和事务管理方法——但它本身不是线程安全的，每次请求应该独立创建使用。

**Executor** 是真正的调度核心，负责 SQL 执行和缓存处理，有 SimpleExecutor（默认，每次创建新 Statement）、ReuseExecutor（复用 PreparedStatement）、BatchExecutor（批量操作，延迟提交）三种实现。

**StatementHandler** 封装了 JDBC Statement 的创建和参数设置，内部包含 **ParameterHandler**（负责把 Java 参数转换为 JDBC 参数，依赖 TypeHandler 体系处理类型转换）和 **ResultSetHandler**（负责把 JDBC ResultSet 结果集转换为 Java 对象，通过反射映射属性）。

**MappedStatement** 是 SQL 的"描述对象"，每条 XML 里的 `<select>/<insert>/<update>/<delete>` 语句都对应一个 MappedStatement，全局存储在 Configuration 中，包含 SQL 源码、参数映射、结果映射等完整信息。

这些组件通过**职责链模式**协同：SqlSession 接收请求 → 委托 Executor → Executor 调度 StatementHandler → StatementHandler 利用 ParameterHandler 设参数，执行后由 ResultSetHandler 映射结果。

###### 3. MyBatis 与 Hibernate 的区别是什么？
[[22_MybatisKnowledge/01_JDBC与MyBatis基础/02、MyBatis架构与核心组件#3. MyBatis vs Hibernate|📖]] 两者代表了 ORM 框架的两种截然不同的哲学。

**MyBatis 是半自动化 ORM**，SQL 完全由开发者掌控，框架只做参数绑定和结果映射。好处是调试直观、性能可精确控制；代价是简单 CRUD 也需要写 SQL，数据库换了可能需要改 SQL 语句。

**Hibernate 是全自动化 ORM**，通过 `@Entity`、`@Column` 等注解配置对象与表的映射，框架根据操作对象自动生成 SQL。好处是简单 CRUD 零代码、自动适配不同数据库方言；代价是复杂场景生成的 SQL 很难控制，出了性能问题也很难排查（因为不知道它生成了什么 SQL）。

**学习曲线**上 MyBatis 更平缓，会写 SQL 基本就能上手；Hibernate 需要额外掌握实体状态（瞬态/持久态/脱管态）、懒加载、一级/二级缓存等概念，门槛较高。

**选择建议**：有大量复杂查询、对 SQL 性能有极致要求的项目选 MyBatis；快速开发、CRUD 为主、需要良好数据库移植性的项目选 Hibernate/JPA。

###### 4. MyBatis 的工作流程是怎样的？
[[22_MybatisKnowledge/01_JDBC与MyBatis基础/03、MyBatis工作流程#3. 完整工作流程|📖]] MyBatis 的完整工作流程分为三个阶段。

**启动阶段**：`SqlSessionFactoryBuilder` 读取 `mybatis-config.xml`，解析所有配置和 Mapper XML 文件，将每条 SQL 封装为 `MappedStatement` 对象，全部存入 `Configuration` 这个"大管家"对象，最终构建出 `SqlSessionFactory` 单例。这个过程只在应用启动时执行一次。

**请求处理阶段**：业务代码调用 Mapper 接口方法（如 `userMapper.selectById(1L)`），MyBatis 通过 **JDK 动态代理**生成的 `MapperProxy` 拦截这次调用，将其转换为对 `SqlSession` 的具体操作（如 `selectOne("com.example.UserMapper.selectById", 1L)`）。

**SQL 执行阶段**：`Executor` 根据方法名找到对应的 `MappedStatement`，`StatementHandler` 创建 JDBC `PreparedStatement`，`ParameterHandler` 将参数绑定到 `?` 占位符，SQL 执行后 `ResultSetHandler` 将 `ResultSet` 转换为 Java 对象返回。

###### 5. MyBatis 中的 SqlSession 是什么？它是线程安全的吗？
[[22_MybatisKnowledge/01_JDBC与MyBatis基础/03、MyBatis工作流程#4. SqlSession 线程安全问题|📖]] **SqlSession** 是 MyBatis 的核心接口，代表一次数据库会话，提供了执行 SQL、获取 Mapper 代理、管理事务等方法。

**它不是线程安全的**。`DefaultSqlSession` 是默认实现，内部持有一个 `Executor` 对象（进而持有数据库连接），如果多个线程共享同一个 SqlSession，会造成数据混乱或事务状态错误。正确用法是每次请求创建一个独立的 SqlSession，用完及时关闭：

```java
SqlSession sqlSession = sqlSessionFactory.openSession();
try {
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    // 执行数据库操作
    sqlSession.commit();
} finally {
    sqlSession.close();
}
```

在 **Spring 集成**场景中，不需要自己管理 SqlSession。`SqlSessionTemplate` 通过动态代理，将 SqlSession 的生命周期与 Spring 事务绑定：同一个事务中的所有操作复用同一个 SqlSession（同一个 Connection），事务结束后自动关闭，从而解决了线程安全问题。

###### 6. MyBatis 中的 SqlSessionFactory 是什么？如何创建？
[[22_MybatisKnowledge/01_JDBC与MyBatis基础/03、MyBatis工作流程#2. SqlSessionFactory 创建|📖]] **SqlSessionFactory** 是 MyBatis 的入口，负责创建 SqlSession 实例，本身是**线程安全**的全局单例，整个应用生命周期内创建一次即可。

创建方式有两种。**XML 配置方式**（推荐）：

```java
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(inputStream);
```

**Java 代码方式**（Spring Boot 中常见）：

```java
Configuration configuration = new Configuration(environment);
configuration.addMapper(UserMapper.class);
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(configuration);
```

创建过程使用了**建造者模式**（`SqlSessionFactoryBuilder`），将复杂的配置构建过程封装起来，配置完成后构建出不可变的 `SqlSessionFactory`。

###### 7. MyBatis 的配置文件（mybatis-config.xml）中可以配置哪些内容？
[[22_MybatisKnowledge/01_JDBC与MyBatis基础/03、MyBatis工作流程#1. mybatis-config.xml 结构|📖]] `mybatis-config.xml` 是 MyBatis 的主配置文件，主要包含以下几个核心配置块：

**properties（属性）**：加载外部 `.properties` 文件，在配置文件其他地方用 `${}` 占位符引用，便于多环境隔离。

**settings（全局设置）**：控制 MyBatis 的核心行为，常用的有 `cacheEnabled`（二级缓存开关）、`lazyLoadingEnabled`（延迟加载开关）、`mapUnderscoreToCamelCase`（下划线自动转驼峰，强烈建议开启）。

**typeAliases（类型别名）**：给 Java 类设置短名称，在 XML 里写 `User` 代替 `com.example.entity.User`。配置包扫描后，包内所有类自动获得别名（类名首字母小写）。

**typeHandlers（类型处理器）**：Java 类型与 JDBC 类型之间的转换器，MyBatis 内置了大量处理器，也可以自定义（如 JSON 字段处理、枚举转 code 值）。

**environments（环境配置）**：配置数据源和事务管理器，支持多环境切换（开发/测试/生产），通过 `default` 属性指定当前激活的环境。

**mappers（映射器）**：注册 Mapper XML 文件或 Mapper 接口的位置，支持 `resource`（XML 路径）、`class`（接口全限定名）、`package`（包扫描）等方式。
