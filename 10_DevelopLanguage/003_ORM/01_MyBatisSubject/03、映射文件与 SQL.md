###### 1. MyBatis 映射文件中有哪些常用标签？
[[22_MybatisKnowledge/02_映射与SQL/01、映射文件与标签体系#1. 标签体系|📖]] MyBatis 映射文件中的标签按职责可以分为五大类。

**SQL 语句标签**：`<select>`、`<insert>`、`<update>`、`<delete>`，分别对应 CRUD 操作，是映射文件最基础的部分。`<insert>` 支持 `useGeneratedKeys="true"` + `keyProperty="id"` 自动获取自增主键。

**结果映射标签**：`<resultMap>`、`<result>`、`<id>`、`<association>`、`<collection>`，用于定义查询结果与 Java 对象的映射关系。`<id>` 专门映射主键字段，有助于缓存优化；`<association>` 处理一对一关系；`<collection>` 处理一对多关系。

**动态 SQL 标签**：`<if>`、`<choose>`（`<when>`/`<otherwise>`）、`<foreach>`、`<where>`、`<set>`、`<trim>`，根据参数条件动态生成 SQL，这是 MyBatis 最强大的特性之一。

**辅助标签**：`<sql>` 定义可复用的 SQL 片段，`<include>` 引用它，避免重复编写相同的字段列表或 WHERE 条件。

**缓存标签**：`<cache>` 开启当前 Mapper 的二级缓存，`<cache-ref>` 引用其他 Mapper 的缓存配置。

###### 2. MyBatis 中 #{} 和 ${} 的区别是什么？
[[22_MybatisKnowledge/02_映射与SQL/01、映射文件与标签体系#2. #{} vs ${}|📖]] 两者的本质区别是**预编译 vs 字符串替换**。

**`#{}`** 会被解析成 `PreparedStatement` 的 `?` 占位符，参数值通过 `ParameterHandler` 以类型安全的方式绑定，自动处理单引号等特殊字符。这意味着参数值只作为"数据"被传入，不可能改变已编译的 SQL 结构，从根本上防止了 SQL 注入。

**`${}`** 是纯字符串替换，在 SQL 生成时直接把变量值拼进去，不做任何转义处理。如果用户传入 `' OR '1'='1`，SQL 语义就被改变了，产生 SQL 注入漏洞。

```xml
<!-- 安全：#{} 预编译，id=1 的情况下生成 WHERE id = ? 然后传入 1 -->
<select id="getUserById" resultType="User">
    SELECT * FROM user WHERE id = #{id}
</select>

<!-- 危险：${} 直接替换，攻击者可以注入任意 SQL -->
<select id="getUserById" resultType="User">
    SELECT * FROM user WHERE id = ${id}
</select>

<!-- ${} 的合法场景：动态排序字段（非用户直接输入，需白名单校验） -->
<select id="getAll" resultType="Goods">
    SELECT * FROM goods ORDER BY ${sortField}
</select>
```

**记忆规则**：参数值用 `#{}`，SQL 片段（表名、列名、ORDER BY 字段）用 `${}`，且 `${}` 的值必须经过白名单校验。

###### 3. MyBatis 如何实现动态 SQL？
[[22_MybatisKnowledge/02_映射与SQL/02、动态SQL|📖]] MyBatis 动态 SQL 通过一套标签体系 + OGNL 表达式实现，在 `org.apache.ibatis.scripting.xmltags` 包中每个标签对应一个 `SqlNode` 实现类，采用**解释器模式**。

**`<if>` + `<where>`** 是最常用的组合，`<where>` 会自动去除多余的 `AND`/`OR` 前缀，避免 `WHERE AND name = ?` 这种语法错误：

```xml
<select id="findUserByCondition" resultType="User">
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

**`<choose>`/`<when>`/`<otherwise>`** 类似于 Java 的 switch-case，多个条件只执行第一个匹配的分支：

```xml
<where>
    <choose>
        <when test="name != null">AND name = #{name}</when>
        <when test="age != null">AND age = #{age}</when>
        <otherwise>AND status = 'active'</otherwise>
    </choose>
</where>
```

**`<foreach>`** 遍历集合，常用于 IN 条件和批量插入，注意 `collection` 属性值要对应参数类型（`list`/`array`/Map 的 key）：

```xml
WHERE id IN
<foreach item="id" collection="list" open="(" separator="," close=")">
    #{id}
</foreach>
```

**`<set>`** 用于 UPDATE 语句，自动去除末尾多余的逗号，与 `<where>` 是一对好搭档：

```xml
<update id="updateUser">
    UPDATE user
    <set>
        <if test="name != null">name = #{name},</if>
        <if test="age != null">age = #{age},</if>
    </set>
    WHERE id = #{id}
</update>
```

###### 4. MyBatis 中 if、choose、foreach 标签的使用场景是什么？
[[22_MybatisKnowledge/02_映射与SQL/02、动态SQL#2. 标签详解|📖]] **`<if>`** 适合多条件同时满足的场景，比如多字段模糊搜索——有什么参数就追加什么条件。需要配合 `<where>` 使用，避免第一个条件前有多余的 AND。

**`<choose>`** 适合多选一的场景，类似 if-else if-else 逻辑。比如：有 ID 就按 ID 查，有手机号就按手机号查，都没有就返回最近注册的用户。只有一个分支会被执行。

**`<foreach>`** 有三个典型场景：IN 条件查询（`WHERE id IN (1,2,3)`）、批量插入（INSERT INTO VALUES (...),(...)`）、批量更新。注意 `collection` 参数：传 `List` 时写 `list`，传数组写 `array`，传 Map 时写对应的 key 名。建议大批量操作分批处理，每批 500~1000 条。

###### 5. MyBatis 如何处理 SQL 注入问题？
[[22_MybatisKnowledge/02_映射与SQL/01、映射文件与标签体系#2. #{} vs ${}|📖]] MyBatis 防 SQL 注入的核心是**彻底使用 `#{}`**，它底层基于 `PreparedStatement` 预编译，参数被视为纯数据，无法改变 SQL 语义。

需要特别注意的是 `${}` 的使用场景。动态表名、动态排序字段确实需要用 `${}`，但这些值绝对不能来自用户未经校验的输入——必须在代码层面做**白名单校验**，确保只有合法的表名或列名才能被拼入 SQL。

此外，应用层的输入验证也是防线之一：在 Service 层对参数进行长度、格式校验，拒绝异常输入，不要完全依赖框架层面的防护。

###### 6. MyBatis 中 resultType 和 resultMap 的区别是什么？
[[22_MybatisKnowledge/02_映射与SQL/01、映射文件与标签体系#3. resultType vs resultMap|📖]] 两者的核心区别是**自动映射 vs 手动映射**。

**`resultType`** 适用于简单场景：查询结果的列名与 Java 对象的属性名完全一致（或开启了驼峰转换后一致）。MyBatis 自动完成映射，写法简单。

**`resultMap`** 适用于复杂场景：列名与属性名不一致、需要关联查询（一对一的 `<association>`、一对多的 `<collection>`）、需要自定义 TypeHandler 等。需要手动配置映射关系，但更灵活强大。

**两者不能同时使用**，同一个 `<select>` 标签只能指定 `resultType` 或 `resultMap` 其中之一。

###### 7. MyBatis 如何实现结果映射？
[[22_MybatisKnowledge/02_映射与SQL/03、结果映射与关联查询|📖]] 结果映射由 `DefaultResultSetHandler` 完成，核心流程是遍历 ResultSet 的每一行，根据 `ResultMap` 的配置将行数据转换为 Java 对象。

简单字段通过 `<result column="xxx" property="yyy"/>` 映射；开启 `mapUnderscoreToCamelCase` 后，`user_name` 这类下划线命名自动映射到 `userName`。关联对象通过 `<association>` 和 `<collection>` 处理嵌套映射或嵌套查询。

###### 8. MyBatis 如何处理字段名与属性名不一致的情况？
[[22_MybatisKnowledge/02_映射与SQL/01、映射文件与标签体系#4. 字段名不一致处理|📖]] 有三种方式解决字段名与属性名不一致的问题。

**方式一（推荐）：全局开启驼峰转换**，适合数据库用下划线命名、Java 用驼峰命名的标准规范：

```xml
<setting name="mapUnderscoreToCamelCase" value="true"/>
```

这样 `user_name` 自动对应 `userName`，`create_time` 自动对应 `createTime`，一行配置解决大部分场景。

**方式二：使用 `resultMap` 手动配置**，适合非标准命名或特殊映射需求：

```xml
<resultMap id="userMap" type="User">
    <id property="id" column="user_id"/>
    <result property="username" column="user_name"/>
</resultMap>
```

**方式三：SQL 别名**，在查询语句中直接用 AS 起别名，简单直接但写起来麻烦，不推荐大量使用：

```xml
SELECT user_id as id, user_name as username FROM users
```

###### 9. MyBatis 中如何实现一对一、一对多、多对多关联查询？
[[22_MybatisKnowledge/02_映射与SQL/03、结果映射与关联查询#2. 关联映射|📖]] 一对一用 `<association>`，一对多用 `<collection>`，多对多本质是两个一对多（通过中间表实现）。

**一对一**（订单关联用户）：

```xml
<resultMap id="orderDetailMap" type="Order">
    <id property="id" column="order_id"/>
    <association property="user" javaType="User">
        <id property="id" column="user_id"/>
        <result property="username" column="username"/>
    </association>
</resultMap>
```

**一对多**（用户关联订单列表）：

```xml
<resultMap id="userOrderMap" type="User">
    <id property="id" column="user_id"/>
    <collection property="orders" ofType="Order">
        <id property="id" column="order_id"/>
        <result property="orderNo" column="order_no"/>
    </collection>
</resultMap>
```

**多对多**通过中间表 JOIN 实现，映射方式与一对多相同，SQL 里多一个 JOIN：

```xml
<select id="getUserWithRoles" resultMap="userRoleMap">
    SELECT u.*, r.role_id, r.role_name
    FROM users u
    LEFT JOIN user_role ur ON u.user_id = ur.user_id
    LEFT JOIN roles r ON ur.role_id = r.role_id
    WHERE u.user_id = #{userId}
</select>
```

###### 10. MyBatis 的延迟加载是什么？如何配置？
[[22_MybatisKnowledge/02_映射与SQL/03、结果映射与关联查询#4. 延迟加载|📖]] **延迟加载**是指关联对象不在主查询时加载，而是等到真正访问该属性时才触发查询。适合关联数据不是每次都会用到的场景，避免无谓的查询开销。

```xml
<settings>
    <setting name="lazyLoadingEnabled" value="true"/>
    <setting name="aggressiveLazyLoading" value="false"/>  <!-- 按需加载，不要激进 -->
</settings>
```

或者在 `resultMap` 中通过 `fetchType="lazy"` 单独控制：

```xml
<collection property="orders" ofType="Order"
            select="selectOrdersByUserId" column="user_id"
            fetchType="lazy"/>
```

**实现原理**：通过 CGLIB 或 Javassist 生成的动态代理实现，当访问 `user.getOrders()` 时，代理对象检测到 orders 还未加载，自动触发 `selectOrdersByUserId` 查询。

**注意**：延迟加载本质上还是 N+1 查询，只是把查询时机推迟了。如果场景是遍历用户列表并访问每个用户的订单，延迟加载并不能减少 SQL 次数，这种情况应该用 JOIN 或批量 IN 查询。

###### 11. MyBatis 的 association 和 collection 标签有什么区别？
[[22_MybatisKnowledge/02_映射与SQL/03、结果映射与关联查询#2. 关联映射|📖]] 两者的区别很简单：**`<association>` 映射单个对象（一对一），`<collection>` 映射对象集合（一对多）**。

`<association>` 使用 `javaType` 指定关联对象的 Java 类型（如 `javaType="User"`）；`<collection>` 使用 `ofType` 指定集合元素的 Java 类型（如 `ofType="Order"`）。

两者都支持通过 `select` 属性触发嵌套查询（懒加载），也都支持通过内嵌字段映射实现 JOIN 查询结果的映射。
