# SqlNode和SqlSource

## 概述

SqlNode和SqlSource是MyBatis动态SQL的核心组件，负责解析和生成动态SQL语句。SqlNode表示SQL语句的组成部分，SqlSource代表完整的SQL资源。

## SqlNode

### 接口定义

```java
public interface SqlNode {
    // 应用SQL节点，生成SQL片段
    boolean apply(DynamicContext context);
}
```

### 核心实现

| 实现类 | 说明 |
|--------|------|
| **MixedSqlNode** | 组合多个SqlNode，遍历调用 |
| **IfSqlNode** | 条件判断，类似if标签 |
| **WhereHandler** | 处理where标签，自动添加WHERE |
| **TrimHandler** | 处理trim标签，自定义前后缀 |
| **ForEachHandler** | 处理foreach标签，循环拼接 |
| **StaticTextSqlNode** | 静态文本 |

### 工作原理

动态SQL标签被解析为SqlNode树：

```xml
<select id="findUsers">
    SELECT * FROM user
    <where>
        <if test="name != null">
            AND name = #{name}
        </if>
        <if test="age != null">
            AND age = #{age}
        </if>
    </where>
</select>
```

解析后的树结构：
```
MixedSqlNode
  └── StaticTextSqlNode ("SELECT * FROM user")
  └── WhereHandler
        ├── IfSqlNode (name != null)
        │     └── StaticTextSqlNode ("AND name = #{name}")
        └── IfSqlNode (age != null)
              └── StaticTextSqlNode ("AND age = #{age}")
```

## SqlSource

### 接口定义

```java
public interface SqlSource {
    // 获取绑定的SQL，包含参数信息
    BoundSql getBoundSql(Object parameterObject);
}
```

### 实现类型

| 类型 | 说明 |
|------|------|
| **StaticSqlSource** | 静态SQL，无动态内容 |
| **RawSqlSource** | 静态SQL，启动时解析 |
| **DynamicSqlSource** | 动态SQL，运行时解析 |

## 生成流程

1. **XML解析**：XMLLanguageDriver解析Mapper.xml
2. **构建SqlNode**：XMLScriptBuilder解析动态标签，构建SqlNode树
3. **创建SqlSource**：
   - 包含动态内容 → DynamicSqlSource
   - 纯静态SQL → RawSqlSource

## 执行过程

1. **解析阶段**：递归解析XML节点，构建SqlNode树
2. **执行阶段**：调用根SqlNode的apply方法
3. **生成BoundSql**：将#{}替换为?，生成可执行SQL

## 性能对比

| 类型 | 解析时机 | 性能 |
|------|----------|------|
| RawSqlSource | 启动时 | 更快 |
| DynamicSqlSource | 运行时 | 灵活 |

## 适用场景

- **IfSqlNode**：条件判断
- **WhereHandler**：自动处理WHERE关键字
- **ForEachHandler**：IN查询、批量插入
- **TrimHandler**：自定义前后缀处理

## 总结

MyBatis通过SqlNode组合模式和SqlSource抽象，采用解析时构建SqlNode树、运行时动态拼接的方式，实现了灵活强大的动态SQL能力。
