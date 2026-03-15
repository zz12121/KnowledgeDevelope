# 动态 SQL

## 一、动态 SQL 的设计思想

动态 SQL 是 MyBatis 最核心的特性之一，基于 **OGNL 表达式**和**解释器模式**实现。每个标签对应 `org.apache.ibatis.scripting.xmltags` 包中的一个 SqlNode 实现类，运行时根据参数动态拼接出最终 SQL。

---

## 二、核心标签详解

### `<if>` 基础条件判断
最常用的标签，配合 `<where>` 使用避免 SQL 语法错误：
```xml
<select id="findUserByCondition" parameterType="map" resultType="User">
    SELECT * FROM users
    <where>
        <if test="name != null and name != ''">
            AND name = #{name}
        </if>
        <if test="age != null">
            AND age = #{age}
        </if>
    </where>
</select>
```
`<where>` 标签会自动去除多余的 `AND`/`OR`，并在有内容时添加 `WHERE` 关键字，避免 `WHERE AND name=?` 这类语法错误。

### `<choose>/<when>/<otherwise>` 多路选择
类似 Java 的 `switch-case`，只有一个分支会被执行：
```xml
<where>
    <choose>
        <when test="name != null">AND name = #{name}</when>
        <when test="age != null">AND age = #{age}</when>
        <otherwise>AND status = 'active'</otherwise>
    </choose>
</where>
```
与 `<if>` 的区别：`<if>` 是多个条件可以同时满足；`<choose>` 是多选一，命中第一个 `<when>` 就停止。

### `<foreach>` 遍历集合
常用于 `IN` 条件和批量插入：
```xml
<!-- IN 查询 -->
WHERE id IN
<foreach item="id" collection="list" open="(" separator="," close=")">
    #{id}
</foreach>

<!-- 批量插入（高效方式） -->
INSERT INTO users (name, age) VALUES
<foreach collection="list" item="user" separator=",">
    (#{user.name}, #{user.age})
</foreach>
```
注意 `collection` 属性值：传入 `List` 时用 `list`，数组用 `array`，Map 里的集合用 Map 的 key 名。

### `<set>` 动态更新
用于 UPDATE 语句，自动去除末尾多余的逗号：
```xml
<update id="updateUser">
    UPDATE users
    <set>
        <if test="name != null">name = #{name},</if>
        <if test="age != null">age = #{age},</if>
    </set>
    WHERE id = #{id}
</update>
```

### `<trim>` 通用裁剪
`<where>` 和 `<set>` 都是 `<trim>` 的语法糖。`<trim prefix="WHERE" prefixOverrides="AND|OR">` 等价于 `<where>`。

### `<sql>` 和 `<include>` 片段复用
```xml
<sql id="userColumns">id, name, age, email, create_time</sql>

<select id="findAll" resultType="User">
    SELECT <include refid="userColumns"/> FROM users
</select>
```

---

## 三、动态 SQL 的源码机制

动态 SQL 在 `org.apache.ibatis.scripting.xmltags` 包中实现，采用**解释器模式**：
- `MixedSqlNode` 持有多个 SqlNode 子节点
- `IfSqlNode` 对应 `<if>` 标签，使用 `OgnlCache.getValue()` 计算 OGNL 表达式
- `ForEachSqlNode` 对应 `<foreach>`，通过 `DynamicContext` 完成字符串拼接
- `WhereSqlNode` / `SetSqlNode` 继承 `TrimSqlNode`，在内容生成后做前后缀处理

运行时，`DynamicSqlSource.getBoundSql()` 遍历 SqlNode 树，将动态部分与参数结合，最终生成完整 SQL 字符串和参数映射列表，封装到 `BoundSql` 对象中。

---

**相关面试题** → [[../../../10_DevelopLanguage/003_ORM/01_MyBatisSubject/03、映射文件与 SQL|映射文件与 SQL 面试题]]
