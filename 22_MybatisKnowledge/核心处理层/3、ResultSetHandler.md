# ResultSetHandler

## 概述

ResultSetHandler是MyBatis四大核心组件之一，负责将JDBC返回的ResultSet结果集转换为Java对象，是MyBatis数据映射层的核心组件。

## 核心功能

1. **结果集映射**：将数据库列与Java对象属性一一对应
2. **类型转换**：通过TypeHandler处理数据库类型与Java类型转换
3. **关联查询**：处理一对一、一对多关联
4. **懒加载**：延迟加载关联对象
5. **多结果集**：支持存储过程返回多个结果集

## 核心方法

### handleResultSets

```java
// 入口方法，处理多个结果集
List<E> handleResultSets(Statement stmt) throws SQLException;
```

### handleRowValues

```java
// 处理单行数据映射
void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, 
    ResultHandler<?> resultHandler) throws SQLException;
```

### createResultObject

```java
// 根据映射规则创建目标对象实例
Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, 
    String columnPrefix) throws SQLException;
```

## 处理流程

1. **获取ResultSet**：StatementHandler执行查询后获取
2. **解析ResultMap**：解析映射配置
3. **创建对象**：利用反射创建Java对象
4. **属性赋值**：通过TypeHandler进行类型转换并赋值
5. **关联处理**：处理association和collection

## 支持的映射类型

| 类型 | 说明 |
|------|------|
| 简单类型 | String、Integer等 |
| POJO对象 | 普通Java对象 |
| 嵌套对象 | association |
| 集合 | collection |

## TypeHandler

ResultSetHandler依赖TypeHandler进行类型转换：

```java
// 类型转换示例
TypeHandler<?> typeHandler = rsw.getTypeHandler(resultMapping.getJdbcType(), 
    resultMapping.getJavaType());
Object value = typeHandler.getResult(rs, columnName);
```

## 懒加载机制

配置懒加载后，关联对象不立即查询，而是在访问时才触发SQL：

```xml
<resultMap id="userMap" type="User">
    <association property="role" column="role_id" 
        select="selectRole" fetchType="lazy"/>
</resultMap>
```

## 适用场景

1. **单表查询**：查询结果映射到实体对象
2. **关联查询**：一对一、一对多关系
3. **存储过程**：多结果集处理
4. **动态映射**：灵活的ResultMap配置

## 总结

ResultSetHandler通过解析映射配置、利用TypeHandler进行类型转换、结合反射和动态代理机制，实现了数据库结果集到Java对象的高效映射，是MyBatis ORM能力的核心体现。
