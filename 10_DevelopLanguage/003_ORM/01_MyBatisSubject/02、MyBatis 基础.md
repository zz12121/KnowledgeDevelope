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

---

###### 7.1 你项目中用过MyBatis吗？是怎么使用的？——高频面试引导问题

面试官问这个问题是想了解你是否有过**真实的持久层开发经验**，以及你对MyBatis的掌握深度。

**常见使用场景**：

| 场景 | 使用方式 | 注意事项 |
|------|---------|---------|
| 增删改查 | XML映射文件或注解 | 用动态SQL处理复杂条件 |
| 分页查询 | PageHelper插件或RowBounds | 推荐使用PageHelper，更灵活 |
| 批量操作 | foreach标签 | 注意SQL长度限制 |
| 关联查询 | association/collection | 考虑使用嵌套查询还是嵌套结果 |
| 插件开发 | Interceptor接口 | 需理解四大组件的代理机制 |

**回答示例**：

> 我们项目里用的是 MyBatis-Plus，主要考虑是：
> 1. **减少样板代码**：自带 CRUD 接口，不用每次都写 `SELECT * FROM xxx WHERE id = ?` 这种简单SQL
> 2. **强大的条件构造器**：`QueryWrapper` 和 `UpdateWrapper` 用起来比 XML 动态 SQL 更简洁
> 3. **自动填充**：create_time、update_time 这些字段不用手动 set
> 4. **分页插件**：内置分页插件，配置一下就能用，比手写分页简单太多
>
> 当然我们也用了 XML 写复杂查询，特别是多表关联、动态 SQL 条件比较复杂的时候，XML 更加清晰。
>
> 追问：为什么不用 Hibernate？
> Hibernate 虽然更省代码，但 SQL 不可控，我们业务里经常有复杂 SQL 优化需求，MyBatis 更灵活。

**技术选型建议**：
- 快速开发 + 简单CRUD → MyBatis-Plus
- 复杂SQL + 需要SQL可控 → MyBatis
- 对象模型复杂 + 团队Hibernate经验丰富 → Hibernate

---

###### 7.2 MyBatis的执行流程了解吗？——高频面试引导问题

**执行流程**：

```
1. 加载配置（全局配置文件 + Mapper接口 + 映射文件）
2. 创建SqlSessionFactory
3. SqlSession执行SQL
   - Executor执行器（SIMPLE/REUSE/BATCH）
   - StatementHandler（参数处理 + SQL执行 + 结果处理）
   - ParameterHandler（参数映射）
   - ResultSetHandler（结果映射）
4. 返回结果
```

**回答示例**：

> MyBatis四大核心对象：
> 1. **Executor**：调度SQL执行，有Simple、Reuse、Batch三种模式
> 2. **StatementHandler**：负责JDBC Statement操作，包括参数设置和SQL执行
> 3. **ParameterHandler**：把Java对象参数转换为JDBC参数
> 4. **ResultSetHandler**：把JDBC结果集转换为Java对象
>
> 追问：和Spring整合后流程？
> Spring Boot自动配置，创建SqlSessionFactory和Mapper扫描注册Bean，使用时通过@Autowired注入Mapper代理对象。

---

###### 7.3 MyBatis的$和#有什么区别？——高频面试引导问题

**区别对比**：

| 特性 | #{} | ${} |
|------|-----|-----|
| 原理 | 占位符?，预编译 | 字符串拼接 |
| SQL注入 | 安全 | 危险 |
| 类型处理 | 自动 | 不处理 |
| 性能 | 高（预编译） | 低 |

**回答示例**：

> 记住：能用#{}就不用${}
> - #{}：PreparedStatement参数占位，安全；自动类型转换
> - ${}：直接字符串拼接，SQL注入风险
>
> 什么时候用${}？
> - 动态表名：`SELECT * FROM ${tableName}`
> - 动态排序：`ORDER BY ${columnName}`
> - 批量插入列名：`INSERT INTO user(${columns}) VALUES(${values})`
>
> 追问：为什么#{}能防止SQL注入？
> #{}会生成`?`占位符，参数会以JDBC预编译方式设置，不会被当作SQL的一部分执行。

---

###### 7.4 MyBatis如何实现分页？分页插件原理是什么？——高频面试引导问题

**分页方式**：

| 方式 | 原理 | 适用场景 |
|------|------|---------|
| 手动分页 | SQL LIMIT | 通用 |
| RowBounds | 内存分页 | 数据量小 |
| PageHelper插件 | 拦截SQL | 推荐 |

**PageHelper原理**：

1. 拦截Executor的query方法
2. 在SQL后追加LIMIT
3. ThreadLocal存储分页参数
4. 查询结束后清除ThreadLocal

**回答示例**：

> 我们用PageHelper：
> ```java
> PageHelper.startPage(pageNum, pageSize);
> List<User> users = userMapper.selectList(null);
> PageInfo<User> pageInfo = new PageInfo<>(users);
> ```
>
> 原理：插件拦截Executor.query()方法，在SQL后面追加LIMIT，同时把分页参数存到ThreadLocal，结果查询完后在PageInfo里组装total。

---

###### 7.5 MyBatis的一级缓存和二级缓存是什么？——高频面试引导问题

**缓存级别**：

| 级别 | 作用域 | 生命周期 | 默认开启 |
|------|--------|---------|---------|
| 一级缓存 | SqlSession | 同一SqlSession | 是 |
| 二级缓存 | SqlSessionFactory | 跨SqlSession | 否 |

**回答示例**：

> - **一级缓存**：同一个SqlSession内，两次查询相同SQL会命中缓存。但如果有增删改操作，会清空缓存。
> - **二级缓存**：跨SqlSession，需要手动开启。缓存是以namespace为单位的，一个namespace数据变化会清空整个namespace的缓存。
>
> 追问：二级缓存有什么问题？
> - 脏读：如果A查询后修改了数据，B用缓存读到的就是旧数据
> - 分布式下无效：二级缓存是单机缓存，分布式环境需要用Redis
>
> 我们项目里缓存都是用Redis，MyBatis二级缓存基本不用。

---

###### 7.6 MyBatis如何实现动态SQL？——高频面试引导问题

**动态SQL标签**：

| 标签 | 作用 |
|------|------|
| if | 条件判断 |
| where | 自动处理WHERE |
| set | 自动处理SET |
| foreach | 循环遍历 |
| choose/when/otherwise | 分支选择 |
| trim | 自定义前后缀 |

**回答示例**：

> 我们项目里动态SQL用法：
> ```xml
> <select id="selectByCondition" resultType="User">
>     SELECT * FROM user
>     <where>
>         <if test="name != null and name != ''">
>             AND name LIKE CONCAT('%', #{name}, '%')
>         </if>
>         <if test="status != null">
>             AND status = #{status}
>         </if>
>     </where>
> </select>
>
> <insert id="batchInsert">
>     INSERT INTO user(name, age) VALUES
>     <foreach collection="list" item="item" separator=",">
>         (#{item.name}, #{item.age})
>     </foreach>
> </insert>
> ```

---

###### 7.7 MyBatis的接口方法和XML映射文件是如何关联的？——高频面试引导问题

**关联原理**：

1. Mapper接口定义方法签名
2. XML中namespace指定接口全限定名
3. statement的id和方法名一致
4. MyBatis通过JDK动态代理创建Mapper代理对象
5. 代理对象根据方法名找XML中的SQL执行

**回答示例**：

> 接口方法如何找到SQL：
> - namespace = 接口全限定名
> - statement id = 方法名
> - parameterType = 参数类型
> - resultType = 返回值类型
>
> 追问：为什么Mapper接口方法能返回实体对象？
> 因为MyBatis的ResultSetHandler会自动映射列名到属性名（驼峰转换或配置映射）。

---

###### 7.8 MyBatis的插件原理是什么？能用来做什么？——高频面试引导问题

**插件原理**：

- 基于JDK动态代理 + 责任链模式
- 四大对象都可被拦截：Executor、StatementHandler、ParameterHandler、ResultSetHandler
- 按顺序执行插件链

**常见插件场景**：

| 插件 | 功能 |
|------|------|
| 分页插件 | 拦截query，追加LIMIT |
| 乐观锁插件 | 拦截update，检查version |
| 性能监控 | 记录SQL执行时间 |
| 脱敏插件 | 拦截结果，敏感数据脱敏 |

**回答示例**：

> 我们项目没用插件，原因：
> 1. 分页用PageHelper更灵活
> 2. 乐观锁在SQL层面加version字段手写
> 3. 性能监控用SkyWalking
>
> 但插件机制是MyBatis最强大的扩展点，可以拦截四大对象做你想做的事。
