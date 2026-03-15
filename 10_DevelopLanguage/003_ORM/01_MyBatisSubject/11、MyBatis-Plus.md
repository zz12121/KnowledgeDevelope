###### 1. 什么是 MyBatis-Plus？它有什么作用？
[[22_MybatisKnowledge/08_MyBatis-Plus/01、MyBatis-Plus核心功能#1. MyBatis-Plus 定位|📖]] MyBatis-Plus（简称 MP）是 MyBatis 的**增强工具**，核心理念是"只做增强不做改变"——在 MyBatis 基础上提供更多开箱即用的功能，不修改原有行为，可以随时切换回原生 MyBatis 操作。

核心价值体现在四个方面：通过 `BaseMapper` 内置通用 CRUD，单表操作无需写 SQL；通过 `Wrapper` 条件构造器用 Java 代码构建查询条件，避免 SQL 拼接；内置分页、乐观锁、逻辑删除、自动填充等常用功能；代码生成器一键生成 Entity/Mapper/Service/Controller 全套代码。

对于复杂 SQL（多表关联、存储过程、窗口函数），仍然使用 MyBatis 原生 XML 方式，两种方式可以在同一个项目里共存。

###### 2. MyBatis-Plus 与 MyBatis 有哪些区别？
[[22_MybatisKnowledge/08_MyBatis-Plus/02、MyBatis-Plus进阶特性#6. MyBatis vs MyBatis-Plus 对比|📖]] 两者不是竞争关系，MyBatis-Plus 是 MyBatis 的增强套件。核心差异在于单表 CRUD 的开发效率和内置功能的丰富程度。

**单表 CRUD**：MyBatis 需要手动写 SQL + XML；MyBatis-Plus 继承 `BaseMapper` 后自动拥有十几个通用方法，零 SQL。

**条件构造**：MyBatis 手动拼接字符串，容易出错且不安全；MyBatis-Plus 提供 `LambdaQueryWrapper` 链式构建，编译期检查字段名，重命名时自动更新。

**分页**：MyBatis 需要手动写 LIMIT 或集成 PageHelper；MyBatis-Plus 内置分页插件，配置一次即可全局使用。

**逻辑删除、乐观锁、自动填充**：MyBatis 需要手动在 SQL 里处理；MyBatis-Plus 通过注解自动处理（`@TableLogic`、`@Version`、`@TableField(fill=...)`）。

**复杂 SQL**：两者完全相同，都用 XML。MyBatis-Plus 在复杂场景下等价于 MyBatis。

###### 3. MyBatis-Plus 的通用 Mapper 有哪些方法？
[[22_MybatisKnowledge/08_MyBatis-Plus/01、MyBatis-Plus核心功能#2. BaseMapper 通用 CRUD|📖]] `BaseMapper<T>` 提供了完整的单表 CRUD 方法。

插入：`insert(entity)` 返回受影响行数，执行后主键自动回填到实体的 id 字段。

删除：`deleteById(id)` 按主键删除；`deleteBatchIds(idList)` 批量删除；`delete(wrapper)` 按条件删除（条件为空时等效于全表删除，非常危险，MP 3.x 版本可以配置禁止全表删除）。

更新：`updateById(entity)` 按主键更新（null 字段不更新）；`update(entity, wrapper)` 按条件更新。

查询：`selectById(id)` 按主键查；`selectBatchIds(idList)` 批量按主键查；`selectOne(wrapper)` 查一条（结果多于一条会报异常）；`selectList(wrapper)` 查列表；`selectPage(page, wrapper)` 分页查询，返回 `IPage<T>` 对象包含总数和记录；`selectCount(wrapper)` 计数查询。

###### 4. MyBatis-Plus 的条件构造器（Wrapper）如何使用？
[[22_MybatisKnowledge/08_MyBatis-Plus/01、MyBatis-Plus核心功能#3. Wrapper 条件构造器|📖]] Wrapper 是 MP 的核心，用 Java 代码构建查询/更新条件，体系分为四种：`QueryWrapper`、`LambdaQueryWrapper`（推荐）、`UpdateWrapper`、`LambdaUpdateWrapper`。

**LambdaQueryWrapper**（强烈推荐，用方法引用代替字符串字段名，重命名时自动更新）：

```java
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getAge, 18)
       .like(User::getName, "张")
       .between(User::getCreateTime, startDate, endDate)
       .orderByDesc(User::getCreateTime);

List<User> users = userMapper.selectList(wrapper);
```

**UpdateWrapper 更新**（可以直接 set 字段值，不需要先构造实体）：

```java
UpdateWrapper<User> updateWrapper = new UpdateWrapper<>();
updateWrapper.eq("status", 0)
             .set("status", 1)
             .set("update_time", new Date());

userMapper.update(null, updateWrapper);  // 第一个参数传 null，全靠 wrapper 里的 set
```

常用条件方法：`eq`（等于）、`ne`（不等于）、`gt/ge/lt/le`（比较）、`like/likeRight`（模糊，likeRight 走索引）、`between`（范围）、`in`（IN 条件）、`isNull`（为空）。

###### 5. MyBatis-Plus 的分页插件如何配置和使用？
[[22_MybatisKnowledge/08_MyBatis-Plus/01、MyBatis-Plus核心功能#4. 分页插件|📖]] 配置一次，全局生效：

```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```

使用非常简洁：

```java
Page<User> page = new Page<>(1, 10);  // 第 1 页，每页 10 条
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<User>()
        .gt(User::getAge, 18);

IPage<User> result = userMapper.selectPage(page, wrapper);

System.out.println("总记录数: " + result.getTotal());  // 自动执行 COUNT 查询
System.out.println("总页数: " + result.getPages());
System.out.println("当前页数据: " + result.getRecords());
```

底层通过插件机制（拦截 Executor.query 方法）自动追加 LIMIT 语句并执行 COUNT 查询，与 PageHelper 原理相似但集成更紧密。

###### 6. MyBatis-Plus 的代码生成器如何使用？
[[22_MybatisKnowledge/08_MyBatis-Plus/02、MyBatis-Plus进阶特性#4. 代码生成器|📖]] 代码生成器根据数据库表结构，自动生成 Entity、Mapper、Service、Controller 全套代码，适合项目初期快速搭建脚手架。

```java
AutoGenerator generator = new AutoGenerator(dataSourceConfig());

GlobalConfig globalConfig = new GlobalConfig.Builder()
    .outputDir(System.getProperty("user.dir") + "/src/main/java")
    .author("YourName").build();

StrategyConfig strategyConfig = new StrategyConfig.Builder()
    .addInclude("user", "order")  // 指定要生成的表
    .entityBuilder().enableLombok()
    .naming(NamingStrategy.underline_to_camel).build();

generator.global(globalConfig).strategy(strategyConfig).execute();
```

生成的代码只是起点，实际业务逻辑还需要手动补充，复杂 SQL 仍然需要在 XML 里写。

###### 7. MyBatis-Plus 的逻辑删除功能是什么？如何实现？
[[22_MybatisKnowledge/08_MyBatis-Plus/02、MyBatis-Plus进阶特性#1. 逻辑删除|📖]] 逻辑删除用字段标记代替物理 DELETE，避免数据永久丢失。

```java
@TableLogic
private Integer deleted;  // 0=未删除, 1=已删除
```

效果：执行 `deleteById()` 时自动改写为 `UPDATE user SET deleted=1 WHERE id=?`；执行所有查询时自动追加 `WHERE deleted=0`，已删除的数据对业务代码完全透明。

全局配置可以统一指定字段名和标记值，不需要每个实体类单独配置：

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
```

###### 8. MyBatis-Plus 的自动填充功能如何使用？
[[22_MybatisKnowledge/08_MyBatis-Plus/02、MyBatis-Plus进阶特性#2. 自动填充|📖]] 自动填充用于在插入或更新时自动设置公共字段，比如创建时间、更新时间、创建人等。

实体类用 `@TableField(fill = FieldFill.INSERT)` 标记字段，然后实现 `MetaObjectHandler`：

```java
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }
}
```

`strict` 系列方法只在字段为 null 时才填充，不会覆盖代码里显式设置的值。

###### 9. MyBatis-Plus 的主键生成策略有哪些？
[[22_MybatisKnowledge/08_MyBatis-Plus/01、MyBatis-Plus核心功能#5. 主键生成策略|📖]] 通过 `@TableId(type = IdType.XXX)` 配置，常用策略有四种：`AUTO`（数据库自增，依赖数据库）、`ASSIGN_ID`（雪花算法生成 Long，不依赖数据库，分布式友好，**推荐**）、`ASSIGN_UUID`（UUID 字符串，较长不推荐做主键）、`INPUT`（手动指定，需要在代码里赋值）。

分布式系统推荐 `ASSIGN_ID`，它生成 64 位 Long 类型 ID，包含时间戳+机器ID+序列号，全局唯一且趋势递增（有利于 B+Tree 索引的插入性能）。

###### 10. MyBatis-Plus 如何实现乐观锁？
[[22_MybatisKnowledge/08_MyBatis-Plus/02、MyBatis-Plus进阶特性#3. 乐观锁|📖]] 乐观锁通过版本号机制解决并发更新冲突，不加数据库锁，适合读多写少、冲突概率低的场景。

实体类加 `@Version` 注解后，更新时 MP 自动在 WHERE 里加版本号条件，并将版本号+1：

```java
// 实际执行：UPDATE user SET name='New', version=2 WHERE id=1 AND version=1
userMapper.updateById(user);
```

如果其他线程在这期间也修改了这条记录，版本号变成了 2，这次更新的 `WHERE version=1` 就匹配不到记录，返回 rows=0，业务代码根据 rows 判断是否发生冲突，提示用户刷新重试。

###### 11. MyBatis-Plus 的性能分析插件如何使用？
[[22_MybatisKnowledge/08_MyBatis-Plus/01、MyBatis-Plus核心功能|📖]] 性能分析插件用于监控 SQL 执行时间，帮助发现慢 SQL。建议只在开发和测试环境开启，生产环境会有性能开销：

```java
@Bean
@Profile({"dev", "test"})
public PerformanceInterceptor performanceInterceptor() {
    PerformanceInterceptor interceptor = new PerformanceInterceptor();
    interceptor.setMaxTime(1000);  // 超过 1 秒的 SQL 会抛出异常（拦截慢 SQL）
    interceptor.setFormat(true);   // 格式化 SQL 输出，便于阅读
    return interceptor;
}
```

**注意**：MP 3.x 版本中 `PerformanceInterceptor` 已废弃，建议用自定义 MyBatis 拦截器或 p6spy 替代。

###### 12. MyBatis-Plus 如何实现多租户功能？
[[22_MybatisKnowledge/08_MyBatis-Plus/02、MyBatis-Plus进阶特性#5. 多租户|📖]] MP 内置多租户插件，自动在所有 SQL 中追加租户隔离条件，适用于共享数据库共享数据表的 SaaS 场景。

配置 `TenantLineInnerInterceptor` 后，MP 会自动在 SELECT/INSERT/UPDATE/DELETE 的 SQL 里追加 `tenant_id = ?` 条件，完全透明，业务代码无需感知：

```java
interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(new TenantLineHandler() {
    @Override
    public Expression getTenantId() {
        return new LongValue(TenantContext.getCurrentTenantId()); // 从上下文取当前租户 ID
    }

    @Override
    public String getTenantIdColumn() { return "tenant_id"; }

    @Override
    public boolean ignoreTable(String tableName) {
        return "sys_config".equals(tableName); // 公共表不需要租户隔离
    }
}));
```
