# MyBatis 源码执行流程解析

> 基于 PDF 图片资料整理，详细分析 MyBatis 从 Mapper 代理创建到结果映射的完整执行流程。

---

## 执行流程概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MyBatis 完整执行流程                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  阶段1：获得Mapper动态代理  ──────────────────────────────────────────────►  │
│                                                                             │
│  阶段2：获得MapperMethod对象  ──────────────────────────────────────────►  │
│                                                                             │
│  阶段3：根据SQL指令跳转执行语句  ────────────────────────────────────────►  │
│                                                                             │
│  阶段4：查询前的缓存处理  ───────────────────────────────────────────────►  │
│                                                                             │
│  阶段5：执行DB查询操作  ─────────────────────────────────────────────────►  │
│                                                                             │
│  阶段6：ResultSet结果集转换为POJO  ─────────────────────────────────────►  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 阶段1：获得 Mapper 动态代理阶段

### 核心流程

```
1. 用户调用 mapperInterface.methodName(params)
           ↓
2. SqlSession.getMapper(mapperInterface)
           ↓
3. Configuration.getMapper(type, sqlSession)
           ↓
4. MapperRegistry.getMapper()
           ↓
5. MapperProxyFactory.newInstance()
           ↓
6. 返回 Mapper 代理对象（MapperProxy）
```

### 关键类

| 类名 | 作用 |
|------|------|
| `SqlSession` | MyBatis 对外操作的统一入口 |
| `Configuration` | 存储所有 MyBatis 配置信息 |
| `MapperRegistry` | 管理所有 Mapper 接口的代理工厂 |
| `MapperProxyFactory` | 创建 Mapper 接口的代理工厂 |
| `MapperProxy` | Mapper 接口的InvocationHandler实现 |

### 源码入口

```java
// 用户调用
UserMapper mapper = sqlSession.getMapper(UserMapper.class);

// 内部调用链
SqlSession.getMapper()
  → Configuration.getMapper()
    → MapperRegistry.getMapper()
      → MapperProxyFactory.newInstance()
        → Proxy.newProxyInstance()
          → 返回 MapperProxy 代理对象
```

---

## 阶段2：获得 MapperMethod 对象

### 核心流程

```
1. 调用 mapperProxy.invoke(method, args)
           ↓
2. MapperMethod 缓存查找
           ↓
3. 未命中则创建 MapperMethod
           ↓
4. 解析方法签名（ParamNameResolver）
           ↓
5. 返回 MapperMethod 对象（包含 SQL 命令和MethodSignature）
```

### MapperMethod 组成

```java
public class MapperMethod {
    private final SqlCommand command;      // SQL 命令类型（SELECT/INSERT/UPDATE/DELETE）
    private final MethodSignature method;   // 方法签名（返回类型、参数类型等）
}
```

### ParamNameResolver

处理方法参数，将 `@Param` 注解的参数和普通参数封装为 `Map`：

```java
// 有 @Param
@Param("id") Long id, @Param("name") String name
  → {0: "id", 1: "name", param1: "id", param2: "name"}

// 无 @Param（使用 param1, param2...）
Long id, String name
  → {0: id, 1: name, param1: id, param2: name}
```

---

## 阶段3：根据 SQL 指令跳转执行语句

### 核心流程

```
1. MapperMethod.execute(sqlSession, args)
           ↓
2. 根据 SQL 命令类型（SELECT/INSERT/UPDATE/DELETE）分发
           ↓
3. SqlSession 调用对应方法
           ↓
4. 进入具体的数据操作流程
```

### SQL 命令分发

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
        case INSERT:
            result = sqlSession.insert(command.getName(), param);
            break;
        case UPDATE:
            result = sqlSession.update(command.getName(), param);
            break;
        case DELETE:
            result = sqlSession.delete(command.getName(), param);
            break;
        case SELECT:
            result = sqlSession.selectList(command.getName(), param);
            break;
        // ... 其他类型
    }
    return result;
}
```

---

## 阶段4：查询前的缓存处理

### 一级缓存流程

```
1. CachingExecutor.query()
           ↓
2. 查询一级缓存（PerpetualCache）
           ↓
3. 命中 → 返回缓存结果
           ↓
4. 未命中 → 委托给 BaseExecutor
           ↓
5. BaseExecutor 查询二级缓存
           ↓
6. 仍未命中 → 查询数据库
```

### 缓存 Key

MyBatis 使用 `BoundSql` 对象生成缓存 Key：

```java
CacheKey cacheKey = new CacheKey();
// 组成元素：statementId + params + rowBounds + SQL + 环境ID
cacheKey.update(ms.getId());
cacheKey.update(rowBounds);
cacheKey.update(boundSql.getSql());
cacheKey.update(params);
```

### 二级缓存流程

- 二级缓存是 namespace 级别的缓存
- 需要在 Mapper.xml 中配置 `<cache/>` 或使用 `@CacheNamespace` 注解
- 存储的数据需要实现 `Serializable` 接口

---

## 阶段5：执行 DB 查询操作

### 核心流程

```
1. BaseExecutor.query()
           ↓
2. 获取 JDBC Connection
           ↓
3. 创建 Statement/PreparedStatement
           ↓
4. 设置参数
           ↓
5. 执行 SQL
           ↓
6. 返回 ResultSet
```

### PreparedStatement 处理

```java
// 创建 PreparedStatement
PreparedStatement ps = connection.prepareStatement(sql);

// 设置参数
for (int i = 0; i < params.size(); i++) {
    ps.setObject(i + 1, params.get(i));
}

// 执行查询
ResultSet rs = ps.executeQuery();
```

### 参数处理器

`ParameterHandler` 负责将 Java 参数转换为 JDBC 参数：

```java
public interface ParameterHandler {
    void setParameters(PreparedStatement ps) throws SQLException;
    Object getParamObject(); // 获取原始参数对象
}
```

---

## 阶段6：ResultSet 结果集转换为 POJO

### 核心流程

```
1. ResultSetHandler 处理 ResultSet
           ↓
2. 根据 resultMap 进行映射
           ↓
3. 简单类型 → 直接返回
           ↓
4. 嵌套结果 → 递归处理
           ↓
5. 关联查询 → 处理 association/collection
           ↓
6. 返回完整 POJO 对象
```

### ResultSetHandler

```java
public interface ResultSetHandler {
    <E> List<E> handleResultSets(Statement stmt) throws SQLException;
    void handleOutputParameters(CallableStatement cs) throws SQLException;
}
```

### 自动映射 vs 手动映射

**自动映射**（根据 `mapUnderscoreToCamelCase` 配置）：
- `user_name` → `userName`
- 需要开启配置：`<setting name="mapUnderscoreToCamelCase" value="true"/>`

**手动映射**（使用 resultMap）：
```xml
<resultMap id="userResultMap" type="User">
    <id column="id" property="id"/>
    <result column="user_name" property="userName"/>
    <result column="user_age" property="userAge"/>
</resultMap>
```

### 嵌套结果映射

```java
// 处理嵌套的 association
if (mapping instanceof AssociationResultMap) {
    AssociationResultMap arm = (AssociationResultMap) mapping;
    // 递归查询关联对象
    Object nestedQueryResult = sqlSession.selectOne(
        arm.getNestedQueryId(), 
        rowValue
    );
}
```

---

## 核心执行流程图

```
用户调用
    ↓
SqlSession.getMapper() → 获取 Mapper 代理对象
    ↓
MapperProxy.invoke() → 进入代理方法
    ↓
MapperMethod.execute() → 分发到具体 SQL 操作
    ↓
CachingExecutor → 一级/二级缓存查询
    ↓
BaseExecutor → 获取数据库连接
    ↓
PreparedStatement → 设置参数、执行 SQL
    ↓
ResultSetHandler → 结果集映射为 POJO
    ↓
返回结果
```

---

## 面试高频问题

### Q1：MyBatis 为什么需要 Mapper 代理？

**答**：MyBatis 使用Mapper 代理机制，将 XML 中定义的 SQL 与 Java 接口方法一一映射，开发者只需面向接口编程，无需手动编写 JDBC 代码。代理对象在运行时生成，底层通过反射调用 SQL。

### Q2：MyBatis 一级缓存和二级缓存的区别？

| 对比项 | 一级缓存 | 二级缓存 |
|--------|---------|---------|
| 作用域 | SqlSession | Mapper Namespace |
| 存储位置 | 内存 | 磁盘/外部缓存 |
| 生命周期 | SqlSession关闭 | 应用关闭 |
| 默认开启 | 是 | 否（需配置）|

### Q3：MyBatis 如何防止 SQL 注入？

**答**：使用 `#{}` 预编译占位符，参数会被当作纯数据处理，不会被解析为 SQL 片段。`${}` 是字符串直接替换，存在 SQL 注入风险。

---

**相关面试题** → [[../../../10_DevelopLanguage/003_ORM/01_MyBatisSubject/04、源码分析|源码分析面试题]]