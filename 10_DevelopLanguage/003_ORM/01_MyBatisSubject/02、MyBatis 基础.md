###### 1. 什么是 MyBatis？它解决了什么问题？
**MyBatis**是一款优秀的**半自动化持久层框架**，它通过XML或注解方式配置SQL语句，将Java对象与数据库记录进行映射。MyBatis的前身是iBATIS，2010年从Apache迁移到Google Code后更名为MyBatis，2013年转移到GitHub。
**核心价值与解决的问题：**
- **简化JDBC冗余代码**：传统JDBC需要大量样板代码（连接管理、参数设置、结果集处理），MyBatis自动处理这些底层细节
- **SQL与代码分离**：将SQL语句外部化到XML文件中，实现数据访问层与业务逻辑层的解耦，便于维护和优化
- **平衡灵活性与复杂度**：与Hibernate等全自动ORM框架不同，MyBatis允许开发者直接控制SQL，兼顾灵活性和性能优化
- **参数映射与结果集转换**：自动将Java对象属性映射到SQL参数，将查询结果转换为Java对象，减少手动转换代码
**设计哲学**：MyBatis采用"半自动化"设计，不像Hibernate那样完全屏蔽SQL，而是让开发者在享受对象化便利的同时，保留对SQL的完全控制权。
###### 2. MyBatis 的核心组件有哪些？
MyBatis的架构基于分层设计，核心组件各司其职，协同完成数据库操作：

|**核心组件**​|**职责描述**​|**源码实现关键**​|
|---|---|---|
|**SqlSessionFactory**​|全局单例，负责创建SqlSession实例，是MyBatis的门面|`org.apache.ibatis.session.SqlSessionFactory`，通常通过`SqlSessionFactoryBuilder.build()`方法构建|
|**SqlSession**​|核心会话接口，提供增删改查API，代表一次数据库会话|`DefaultSqlSession`是默认实现，但非线程安全|
|**Executor**​|**SQL执行器**，是MyBatis调度核心，处理缓存和事务|有`SimpleExecutor`（简单执行）、`ReuseExecutor`（语句重用）、`BatchExecutor`（批量处理）三种策略|
|**StatementHandler**​|封装JDBC Statement操作，负责SQL语句生成和参数设置|包含`PreparedStatementHandler`等实现，应用**模板方法模式**|
|**ParameterHandler**​|将Java类型数据转换为JDBC所需参数类型|使用TypeHandler体系处理类型转换|
|**ResultSetHandler**​|将JDBC ResultSet结果集转换为Java对象|通过反射机制实现对象属性映射|
|**MappedStatement**​|描述SQL配置信息，全局存储在Configuration中|每个SQL语句对应一个MappedStatement，包含SQL源码、参数映射等信息|
这些组件通过**职责链模式**协同工作：`SqlSession`接收请求后委托给`Executor`，`Executor`通过`StatementHandler`、`ParameterHandler`、`ResultSetHandler`分别处理语句生成、参数设置和结果映射。
###### 3. MyBatis 与 Hibernate 的区别是什么？
两者是持久层框架的不同哲学代表，主要区别如下：

|**对比维度**​|**MyBatis**​|**Hibernate**​|
|---|---|---|
|**架构性质**​|半自动化ORM，SQL可控|全自动化ORM，SQL透明|
|**SQL控制**​|手动编写SQL，完全控制|框架生成SQL，优化受限|
|**学习曲线**​|平缓，易于掌握|陡峭，门槛较高|
|**性能优化**​|直接优化SQL语句|优化HQL或缓存策略|
|**数据库移植性**​|较差，SQL需针对数据库调整|优秀，自动适配不同数据库方言|
|**开发效率**​|SQL编写增加工作量|基础CRUD操作效率高|
|**适用场景**​|高性能要求、复杂SQL、遗留系统改造|快速开发、对象模型复杂、数据库无关性要求高|
**选择建议**：需要精细控制SQL、处理复杂查询或对性能有极致要求的项目选择MyBatis；追求开发效率、对象模型复杂或需要良好数据库移植性的项目选择Hibernate。
###### 4. MyBatis 的工作流程是怎样的？
MyBatis的完整工作流程可以分为以下几个阶段：
1. **配置加载阶段**
    - 应用程序启动时，`SqlSessionFactoryBuilder`读取`mybatis-config.xml`全局配置文件
    - 解析配置信息并构建`Configuration`对象，该对象包含所有配置信息和MappedStatement集合
2. **SQL会话创建**
    - 通过`SqlSessionFactory`创建`SqlSession`实例，每个会话对应一次数据库交互
    - 根据配置的ExecutorType创建相应的Executor执行器实例
3. **SQL请求处理**
    - 应用程序调用Mapper接口方法，MyBatis通过**动态代理**生成代理对象
    - 代理对象将方法调用转换为`SqlSession`的数据库操作方法
4. **SQL解析与执行**
    - Executor根据方法签名查找对应的`MappedStatement`
    - `StatementHandler`创建JDBC Statement并设置参数
    - `ParameterHandler`将Java参数转换为JDBC参数（应用已注册的TypeHandler）
5. **结果映射处理**
    - `ResultSetHandler`将ResultSet结果集转换为Java对象
    - 根据配置的ResultMap或自动映射规则进行属性填充
6. **会话关闭与资源释放**
    - SqlSession关闭时释放数据库连接和其他资源
**源码关键点**：在`org.apache.ibatis.session.defaults.DefaultSqlSession`中可以看到`selectList`、`insert`等方法的具体实现，它们最终都委托给Executor执行。
###### 5. MyBatis 中的 SqlSession 是什么？它是线程安全的吗？
**SqlSession**是MyBatis的核心接口，代表一次数据库会话，提供了执行SQL、获取Mapper、管理事务等方法。
**线程安全性分析**：
- **默认实现非线程安全**：`DefaultSqlSession`是SqlSession的默认实现，**不是线程安全的**
- **设计原理**：每个SqlSession实例应包含独立的数据库连接和事务上下文，多线程共享可能导致数据混乱或事务错误
- **正确用法**：在Web应用中，应该为每个请求创建独立的SqlSession，在请求处理完成后及时关闭
```java
// 正确用法：每个线程使用独立的SqlSession
SqlSession sqlSession = sqlSessionFactory.openSession();
try {
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    // 执行数据库操作
    sqlSession.commit(); // 提交事务
} finally {
    sqlSession.close(); // 确保资源释放
}
```
**Spring集成中的线程安全**：在Spring-MyBatis集成中，通常使用`SqlSessionTemplate`，它通过**动态代理**将SqlSession绑定到当前线程的Spring事务上下文中，实现了线程安全。
###### 6. MyBatis 中的 SqlSessionFactory 是什么？如何创建？
**SqlSessionFactory**是MyBatis的入口点和核心工厂类，负责创建SqlSession实例，是**线程安全**的单例对象。
**创建方式主要有两种**：
1. **XML配置方式（推荐）**
```java
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```
在`mybatis-config.xml`中配置数据源、事务管理器、映射文件等
2. **Java配置方式**
```java
DataSource dataSource = getDataSource(); // 获取数据源
TransactionFactory transactionFactory = new JdbcTransactionFactory();
    
Environment environment = new Environment("development", transactionFactory, dataSource);
Configuration configuration = new Configuration(environment);
configuration.addMapper(UserMapper.class);

SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
```
**设计模式应用**：SqlSessionFactory的创建使用了**建造者模式**（SqlSessionFactoryBuilder），使得配置过程更加灵活。
###### 7. MyBatis 的配置文件（mybatis-config.xml）中可以配置哪些内容？
mybatis-config.xml是MyBatis的主配置文件，采用分层结构设计，主要包含以下配置 sections
1. **properties（属性配置）**
    - 用于加载外部属性文件，可在配置文件中使用`${}`占位符引用
    - 便于环境隔离（开发、测试、生产）
2. **settings（全局设置）**
    - 核心行为配置，如缓存、延迟加载、自动映射等
    - 重要参数包括：
        - `cacheEnabled`：控制二级缓存开关 
        - `lazyLoadingEnabled`：延迟加载开关
        - `mapUnderscoreToCamelCase`：自动下划线转驼峰命名
3. **typeAliases（类型别名）**
    - 为Java类型设置简称，简化映射文件配置
    - MyBatis为常见类型内置了别名（如`string`→`java.lang.String`）
4. **typeHandlers（类型处理器）**
    - 负责JDBC类型与Java类型的相互转换
    - 可自定义处理器处理特殊类型转换需求
5. **environments（环境配置）**
    - 配置数据源、事务管理器，支持多环境切换
    - 通过`default`属性指定默认环境
6. **mappers（映射器配置）**
    - 注册Mapper接口或映射文件的位置
    - 支持多种注册方式：resource、class、package等
**配置示例**：
```xml
<configuration>
    <settings>
        <setting name="cacheEnabled" value="true"/>
        <setting name="lazyLoadingEnabled" value="true"/>
    </settings>
    
    <typeAliases>
        <package name="com.example.model"/>
    </typeAliases>
    
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
            </dataSource>
        </environment>
    </environments>
    
    <mappers>
        <mapper resource="mapper/UserMapper.xml"/>
    </mappers>
</configuration>
```
