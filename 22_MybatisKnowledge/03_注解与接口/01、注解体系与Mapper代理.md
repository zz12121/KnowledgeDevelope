# 注解体系与 Mapper 代理

## 一、注解体系分类

MyBatis 注解体系完整，可按功能分为六类：

**CRUD 注解**：`@Select`、`@Insert`、`@Update`、`@Delete`，声明 SQL 语句，等效于 XML 中的对应标签。

**结果映射**：`@Results`、`@Result`、`@ResultMap`，配置结果集映射关系，等效于 `<resultMap>`。

**关联映射**：`@One`（一对一）、`@Many`（一对多），等效于 `<association>` 和 `<collection>`。

**参数处理**：`@Param`（参数命名）、`@Options`（配置 useGeneratedKeys 等选项）、`@SelectKey`（非自增主键策略）。

**动态 SQL**：`@SelectProvider`、`@InsertProvider`、`@UpdateProvider`、`@DeleteProvider`，将 SQL 生成逻辑提取到独立 Java 类中。

**缓存配置**：`@CacheNamespace`（开启命名空间二级缓存）、`@CacheNamespaceRef`（引用其他命名空间缓存）。

---

## 二、`@Param` 注解

`@Param` 的作用是**为方法参数命名**，解决多参数时的映射问题。

**不用 @Param 的问题**：只有一个参数时 MyBatis 可以直接用参数名；多个参数时，默认命名为 `param1`、`param2`... 或 `arg0`、`arg1`...，极易混乱：
```java
// 错误：多参数时 #{name} 和 #{age} 无法正确解析
User findUser(String name, Integer age);

// 正确：明确命名
User findUser(@Param("name") String name, @Param("age") Integer age);
```

在源码 `ParamNameResolver` 中，有 `@Param` 时用注解值，没有时用 `param1/param2`，同时也保留 `param1/param2` 作为兼容别名。

**使用建议**：只要 Mapper 方法有超过一个参数，就加 `@Param`，清晰明确。

---

## 三、Mapper 接口的动态代理

**为什么 Mapper 接口不需要实现类？**

MyBatis 在运行时通过 JDK 动态代理生成接口的实现。核心类是 `MapperProxyFactory`：

```java
// 源码简化
public T newInstance(SqlSession sqlSession) {
    MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface);
    return (T) Proxy.newProxyInstance(
        mapperInterface.getClassLoader(),
        new Class[]{mapperInterface},
        mapperProxy);
}
```

当调用 `userMapper.findById(1)` 时，实际触发 `MapperProxy.invoke()`，它通过 `MapperMethod.execute()` 根据方法的返回类型（List、单对象、void 等）选择对应的 SqlSession 方法执行。

---

## 四、注解 vs XML 的选择

**注解方式**的优点是简洁，代码即配置，无需切换文件；缺点是长 SQL 难以阅读，动态 SQL 支持很弱（需借助 `<script>` 标签或 Provider 类），修改 SQL 需要重新编译。

**XML 方式**的优点是 SQL 集中管理、结构清晰，原生支持全套动态 SQL 标签，DBA 可独立优化，修改 SQL 热加载无需重新编译；缺点是需要维护独立的 XML 文件。

**生产实践建议**：
- 简单的单表 CRUD 可用注解，减少配置文件
- 复杂 SQL、动态 SQL、多表关联用 XML
- **不要混用过度**，同一 Mapper 尽量统一风格

---

## 五、`@SelectProvider` 动态 SQL

当需要在 Java 代码中构建复杂 SQL 时，可以用 Provider 注解将 SQL 生成逻辑提取到独立类：

```java
// Provider 类
public class UserSqlProvider {
    public String findByCondition(Map<String, Object> params) {
        return new SQL() {{
            SELECT("*");
            FROM("users");
            if (params.get("name") != null) WHERE("name = #{name}");
            if (params.get("age") != null) WHERE("age >= #{age}");
            ORDER_BY("create_time DESC");
        }}.toString();
    }
}

// Mapper 接口
@SelectProvider(type = UserSqlProvider.class, method = "findByCondition")
List<User> findByCondition(@Param("name") String name, @Param("age") Integer age);
```

MyBatis 提供的 `SQL` 工具类可以安全地构建 SQL 语句，避免字符串拼接的错误风险。Provider 方法必须是 `public`，返回 `String`，参数可以无参或与 Mapper 方法参数一致。

---

**相关面试题** → [[../../../10_DevelopLanguage/003_ORM/01_MyBatisSubject/04、注解与接口|注解与接口面试题]]
