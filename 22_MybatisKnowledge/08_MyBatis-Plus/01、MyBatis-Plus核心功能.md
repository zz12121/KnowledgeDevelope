# MyBatis-Plus 核心功能

## 1. MyBatis-Plus 定位

MyBatis-Plus（简称 MP）是 MyBatis 的**增强工具**，核心理念是「只做增强不做改变」——在 MyBatis 的基础上提供更多开箱即用的功能，但不修改 MyBatis 原有行为，两者可以完全共存。

**核心价值**：
- 单表 CRUD 无需编写 SQL，继承 `BaseMapper` 即可
- Lambda 条件构造器，编译期检查，告别 SQL 拼接
- 内置分页、逻辑删除、乐观锁、自动填充等常用功能
- 代码生成器，一键生成 Entity/Mapper/Service/Controller

**与 MyBatis 的关系**：对于复杂 SQL（多表关联、存储过程、窗口函数），仍然使用原生 MyBatis 的 XML 方式。MP 解决的是"无聊的重复代码"，而不是替代 MyBatis 的 SQL 控制能力。

---

## 2. BaseMapper 通用 CRUD

继承 `BaseMapper<T>` 后，无需编写任何 SQL，即可获得以下方法：

**插入**：
```java
int insert(T entity);
```

**删除**：
```java
int deleteById(Serializable id);
int deleteBatchIds(Collection<? extends Serializable> idList);  // 批量删除
int delete(Wrapper<T> wrapper);                                   // 条件删除
```

**更新**：
```java
int updateById(T entity);                               // 根据 ID，null 字段不更新
int update(T entity, Wrapper<T> updateWrapper);         // 条件更新
```

**查询**：
```java
T selectById(Serializable id);
List<T> selectBatchIds(Collection<? extends Serializable> idList);
T selectOne(Wrapper<T> queryWrapper);                   // 查一条，多条则报错
List<T> selectList(Wrapper<T> queryWrapper);
IPage<T> selectPage(IPage<T> page, Wrapper<T> queryWrapper);  // 分页
Integer selectCount(Wrapper<T> queryWrapper);           // 计数
```

---

## 3. Wrapper 条件构造器

Wrapper 是 MP 的核心，用 Java 代码构建查询条件，避免 SQL 拼接。

**Wrapper 体系**：
```
AbstractWrapper
├── QueryWrapper<T>         // 查询/删除条件
│   └── LambdaQueryWrapper<T>  // Lambda 版本（推荐）
└── UpdateWrapper<T>        // 更新条件
    └── LambdaUpdateWrapper<T>
```

**QueryWrapper 使用**：
```java
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.select("id", "name", "age")
       .eq("age", 18)
       .like("name", "张")
       .gt("create_time", "2024-01-01")
       .orderByDesc("create_time");

List<User> users = userMapper.selectList(wrapper);
```

**LambdaQueryWrapper（强烈推荐）**：

用方法引用代替字符串字段名，重命名时自动更新，编译期检查：

```java
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getAge, 18)
       .like(User::getName, "张")
       .between(User::getCreateTime, startDate, endDate)
       .orderByDesc(User::getCreateTime);

List<User> users = userMapper.selectList(wrapper);
```

**UpdateWrapper 更新**：
```java
UpdateWrapper<User> updateWrapper = new UpdateWrapper<>();
updateWrapper.eq("status", 0)
             .set("status", 1)
             .set("update_time", new Date());

userMapper.update(null, updateWrapper);
```

**常用条件方法**：

| 方法 | SQL | 示例 |
|------|-----|------|
| `eq("col", val)` | `col = val` | `eq("age", 18)` |
| `ne("col", val)` | `col != val` | |
| `gt/ge/lt/le` | `> >= < <=` | `gt("age", 18)` |
| `like("col", val)` | `LIKE '%val%'` | `like("name", "张")` |
| `likeRight("col", val)` | `LIKE 'val%'` | 前缀匹配，走索引 |
| `between(col, v1, v2)` | `BETWEEN v1 AND v2` | |
| `in("col", list)` | `IN (...)` | `in("id", idList)` |
| `isNull("col")` | `col IS NULL` | |
| `and/or` | `AND / OR` | 逻辑组合 |

---

## 4. 分页插件

**配置**：

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

**使用**：

```java
// 第 1 页，每页 10 条
Page<User> page = new Page<>(1, 10);
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<User>()
        .gt(User::getAge, 18);

IPage<User> result = userMapper.selectPage(page, wrapper);

System.out.println("总记录数: " + result.getTotal());
System.out.println("总页数: " + result.getPages());
System.out.println("当前页数据: " + result.getRecords());
```

**源码机制**：分页插件通过拦截 `Executor` 的 `query` 方法，在执行前自动追加 `LIMIT` 语句，并执行一次 COUNT 查询获取总数。底层是 MyBatis 插件机制（JDK 动态代理 + 责任链）。

---

## 5. 主键生成策略

通过 `@TableId` 注解配置：

```java
@Data
public class User {
    @TableId(type = IdType.ASSIGN_ID)  // 雪花算法
    private Long id;
}
```

| 策略 | 说明 |
|------|------|
| `AUTO` | 数据库自增（需数据库支持，如 MySQL AUTO_INCREMENT） |
| `ASSIGN_ID` | 雪花算法生成 Long 类型 ID（默认，不依赖数据库） |
| `ASSIGN_UUID` | UUID 字符串（不带横线） |
| `INPUT` | 用户手动指定 |
| `NONE` | 不设置，框架不介入 |

全局配置：
```yaml
mybatis-plus:
  global-config:
    db-config:
      id-type: assign_id
```

---

## 6. 常用实体注解

```java
@Data
@TableName("user_info")  // 指定表名（类名与表名不一致时使用）
public class User {

    @TableId(type = IdType.ASSIGN_ID)
    private Long id;

    @TableField("user_name")    // 指定列名（属性名与列名不一致时使用）
    private String name;

    @TableField(exist = false)  // 不映射到数据库列
    private String extraField;

    @TableLogic                 // 逻辑删除字段
    private Integer deleted;

    @Version                    // 乐观锁版本字段
    private Integer version;

    @TableField(fill = FieldFill.INSERT)         // 插入时自动填充
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)  // 插入和更新时自动填充
    private LocalDateTime updateTime;
}
```

---

## 相关面试题

- [[003_ORM/01_MyBatisSubject/11、MyBatis-Plus|📖 11、MyBatis-Plus]]
