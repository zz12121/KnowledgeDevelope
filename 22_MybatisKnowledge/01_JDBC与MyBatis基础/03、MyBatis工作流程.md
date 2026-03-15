# MyBatis 工作流程

## 一、配置文件结构

`mybatis-config.xml` 是 MyBatis 的主配置文件，按标签分层配置：

**properties**：加载外部属性文件，用 `${}` 占位符引用，实现开发/测试/生产环境隔离。

**settings**：最重要的全局行为开关，常用配置包括：
- `cacheEnabled`：二级缓存总开关（默认 true）
- `lazyLoadingEnabled`：延迟加载开关
- `mapUnderscoreToCamelCase`：数据库下划线字段自动映射为 Java 驼峰属性（**强烈建议开启**）

**typeAliases**：为 Java 类型设置简称，避免在 Mapper XML 里写一长串全限定名。MyBatis 内置了常见类型的别名（`string`、`int`、`list` 等）。

**typeHandlers**：类型处理器，负责 JDBC 类型与 Java 类型的互转，可自定义处理枚举、JSON 字段等特殊类型。

**environments**：数据源和事务管理器配置，支持多环境切换，通过 `default` 指定当前使用哪套环境。

**mappers**：注册 Mapper XML 或接口位置，支持 resource（路径）、class（接口类）、package（包扫描）三种方式。

---

## 二、SqlSessionFactory 创建

两种方式：

**XML 配置方式（推荐）**：
```java
InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(inputStream);
```

**Java 代码方式**（适合 Spring Boot 场景）：
```java
DataSource ds = getDataSource();
Environment env = new Environment("dev", new JdbcTransactionFactory(), ds);
Configuration config = new Configuration(env);
config.addMapper(UserMapper.class);
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(config);
```

SqlSessionFactory 的创建使用了**建造者模式**（Builder），使配置过程灵活可扩展。创建后它是线程安全的，整个应用只需要一个实例。

---

## 三、Mapper 动态代理机制

这是 MyBatis 最核心的"魔法"：**Mapper 接口没有实现类，为什么可以直接调用？**

答案是 JDK 动态代理。`sqlSession.getMapper(UserMapper.class)` 时，`MapperProxyFactory` 用 `Proxy.newProxyInstance()` 生成一个代理对象。调用 `userMapper.findById(1)` 时，实际触发 `MapperProxy.invoke()` 方法，它从 Configuration 中找到 key 为 `com.example.UserMapper.findById` 的 `MappedStatement`，然后转发给 `SqlSession` 执行。

**Mapper 接口与 XML 的绑定规则**：
- XML 的 `namespace` 必须等于 Mapper 接口的**全限定类名**
- XML 语句的 `id` 必须等于接口的**方法名**
- 参数类型和返回类型也需匹配

这套命名约定是 MyBatis 绑定机制的基础，任何不一致都会抛 `Invalid bound statement (not found)` 异常。

---

## 四、SqlSession 的线程安全问题

`DefaultSqlSession` **不是线程安全的**，原因是它持有独立的数据库连接和事务上下文，多线程共享会导致事务混乱或数据错误。

正确用法是**每个请求（每个线程）使用独立的 SqlSession**：
```java
SqlSession session = sqlSessionFactory.openSession();
try {
    UserMapper mapper = session.getMapper(UserMapper.class);
    // 执行操作
    session.commit();
} finally {
    session.close();  // 必须关闭
}
```

**Spring 集成中的解决方案**：`SqlSessionTemplate` 通过内部的 `SqlSessionInterceptor` 动态代理，每次方法调用时从 `TransactionSynchronizationManager` 获取当前事务绑定的 SqlSession，保证同一事务内使用同一个连接。这样既线程安全，又支持 Spring 声明式事务。

---

**相关面试题** → [[../../../10_DevelopLanguage/003_ORM/01_MyBatisSubject/02、MyBatis 基础|MyBatis 基础面试题]]
