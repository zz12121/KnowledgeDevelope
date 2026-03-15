# TypeHandler 与高级特性

## 一、TypeHandler 的作用

TypeHandler 是 MyBatis 的类型转换桥梁，承担 **Java 类型 ↔ JDBC 类型**的双向转换：

- **参数设置时**：将 Java 对象属性转换为 JDBC 类型，调用 `PreparedStatement.setXxx()` 设置占位符值
- **结果获取时**：从 ResultSet 中读取 JDBC 类型值，转换为 Java 对象属性

MyBatis 内置了大量 TypeHandler（String、Integer、Date、Boolean 等），通过 `TypeHandlerRegistry` 管理，按 Java 类型和 JDBC 类型双向索引。

---

## 二、自定义 TypeHandler

**典型场景**：
- 枚举类型存储自定义 code 值（而非默认的 name 或 ordinal）
- Java 对象与 JSON 字符串互转存储
- 敏感字段加解密
- BLOB/CLOB 特殊处理

**实现步骤**：继承 `BaseTypeHandler<T>`，实现四个方法：

```java
@MappedTypes(MyComplexType.class)
@MappedJdbcTypes(JdbcType.VARCHAR)
public class JsonTypeHandler extends BaseTypeHandler<MyComplexType> {
    private static final ObjectMapper mapper = new ObjectMapper();

    // 设置参数：Java -> JDBC
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, 
                                    MyComplexType param, JdbcType jdbcType) throws SQLException {
        ps.setString(i, mapper.writeValueAsString(param));
    }

    // 获取结果：JDBC -> Java（按列名）
    @Override
    public MyComplexType getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String json = rs.getString(columnName);
        return json == null ? null : mapper.readValue(json, MyComplexType.class);
    }

    // 获取结果：JDBC -> Java（按列索引）
    @Override
    public MyComplexType getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        String json = rs.getString(columnIndex);
        return json == null ? null : mapper.readValue(json, MyComplexType.class);
    }

    // 获取结果：来自存储过程 CallableStatement
    @Override
    public MyComplexType getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        String json = cs.getString(columnIndex);
        return json == null ? null : mapper.readValue(json, MyComplexType.class);
    }
}
```

**注册方式**：
```yaml
# Spring Boot 中包扫描自动注册（推荐）
mybatis:
  type-handlers-package: com.example.typehandler
```

---

## 三、枚举类型处理

MyBatis 内置两种枚举处理器：
- **EnumTypeHandler**（默认）：存储枚举的 `name()` 字符串，如 `"ACTIVE"`
- **EnumOrdinalTypeHandler**：存储枚举的 `ordinal()` 整数，如 `0`、`1`、`2`

当需要存储自定义 code 值时，需要自定义处理器：定义 `BaseCodeEnum` 接口（约定 `getCode()` 方法），枚举实现该接口，TypeHandler 在 `setNonNullParameter` 中调用 `getCode()` 存值，在 `getNullableResult` 中通过 code 反向找到对应枚举实例。

---

## 四、动态表名

MyBatis 本身没有专门的动态表名语法，有三种实现方式：

**方式一：`${tableName}`**（有 SQL 注入风险，需做白名单校验）：
```xml
SELECT * FROM ${tableName} WHERE id = #{id}
```

**方式二：Provider 类**（Java 代码动态拼接，可做校验）：
```java
public String selectByTable(Map<String, Object> params) {
    String tableName = (String) params.get("tableName");
    // 白名单校验
    if (!ALLOWED_TABLES.contains(tableName)) throw new IllegalArgumentException("非法表名");
    return "SELECT * FROM " + tableName + " WHERE id = #{id}";
}
```

**方式三：拦截器**（最安全灵活，通过 `MetaObject` 修改 `BoundSql` 中的 SQL 文本，实现运行时表名替换）。

---

## 五、存储过程调用

通过 `statementType="CALLABLE"` 声明调用存储过程：
```xml
<select id="callUserProcedure" statementType="CALLABLE" parameterType="map">
    {call getUserDetail(
        #{userId, mode=IN, jdbcType=INTEGER},
        #{userName, mode=OUT, jdbcType=VARCHAR}
    )}
</select>
```
OUT 参数通过传入的 Map 对象接收，MyBatis 通过 `CallableStatementHandler` 处理并回填到 Map 中。

---

**相关面试题** → [[../../../10_DevelopLanguage/003_ORM/01_MyBatisSubject/06、插件与扩展|插件与扩展面试题]] | [[../../../10_DevelopLanguage/003_ORM/01_MyBatisSubject/12、高级特性|高级特性面试题]]
