###### 1. MyBatis 映射文件中有哪些常用标签？
MyBatis映射文件中的标签可以分为多个类别，每个类别承担不同的职责：

|**标签类别**​|**核心标签**​|**功能描述**​|
|---|---|---|
|**SQL语句标签**​|`<select>`, `<insert>`, `<update>`, `<delete>`|分别对应CRUD操作，是映射文件的基础|
|**结果映射标签**​|`<resultMap>`, `<result>`, `<id>`, `<association>`, `<collection>`|定义查询结果与Java对象的映射关系|
|**动态SQL标签**​|`<if>`, `<choose>`, `<when>`, `<otherwise>`, `<foreach>`, `<where>`, `<set>`, `<trim>`|根据条件动态生成SQL语句|
|**辅助标签**​|`<sql>`, `<include>`|定义可重用的SQL片段|
|**缓存标签**​|`<cache>`, `<cache-ref>`|配置缓存策略|
**核心标签详解：**
- **`<select>`标签**：最重要的查询标签，包含`id`(必选)、`parameterType`、`resultType`/`resultMap`、`flushCache`等属性
- **`<insert>`标签**：支持`useGeneratedKeys`和`keyProperty`用于获取自增主键
- **`<resultMap>`标签**：MyBatis最强大的特性，用于处理复杂的结果映射
###### 2. MyBatis 中 #{} 和 ${} 的区别是什么？
两者在实现机制和安全性上有本质区别：

|**特性**​|**#{}**​|**${}**​|
|---|---|---|
|**处理机制**​|**预编译处理**（PreparedStatement占位符）|**字符串替换**（直接拼接SQL）|
|**安全性**​|**安全**，能有效防止SQL注入|**不安全**，存在SQL注入风险|
|**引号处理**​|自动添加单引号，处理字符串类型|直接替换，不添加单引号|
|**适用场景**​|参数值传递（WHERE条件值）|SQL关键字、表名、字段名等|
**源码角度分析**：
- **#{}实现**：在`org.apache.ibatis.scripting.xmltags.TextSqlNode`中，`#{}`会被解析为`?`占位符，通过`ParameterHandler`进行类型安全的值设置
- **${}实现**：在`org.apache.ibatis.scripting.xmltags.TextSqlNode`中直接进行字符串替换，不进行预编译处理
**示例对比**：
```xml
<!-- 安全写法 -->
<select id="getUserById" resultType="User">
    SELECT * FROM user WHERE id = #{id}  <!-- 预编译：WHERE id = ? -->
</select>

<!-- 危险写法（SQL注入风险） -->
<select id="getUserById" resultType="User">
    SELECT * FROM user WHERE id = ${id}  <!-- 直接替换：WHERE id = 实际值 -->
</select>

<!-- ${}的正确使用场景 -->
<select id="getAll" resultType="Goods">
    SELECT * FROM goods ORDER BY price ${sort}  <!-- 排序字段 -->
</select>
```
###### 3. MyBatis 如何实现动态 SQL？
MyBatis动态SQL允许根据参数条件动态生成SQL语句，主要通过OGNL表达式和一系列标签实现。
**核心动态SQL标签：**
1. **`<if>`标签**：基础条件判断
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
1. **`<choose>`、`<when>`、`<otherwise>`标签**：多路选择（类似Java的switch-case）
```xml
<select id="findUser" parameterType="map" resultType="User">
    SELECT * FROM users
    <where>
        <choose>
            <when test="name != null">
                AND name = #{name}
            </when>
            <when test="age != null">
                AND age = #{age}
            </when>
            <otherwise>
                AND status = 'active'
            </otherwise>
        </choose>
    </where>
</select>
```
1. **`<foreach>`标签**：遍历集合，常用于IN条件
```xml
<select id="findUsersByIds" parameterType="list" resultType="User">
    SELECT * FROM users
    WHERE id IN
    <foreach item="id" collection="list" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
```
1. **`<where>`和`<set>`标签**：智能处理SQL关键字
    - **`<where>`**：自动去除多余的AND/OR，避免WHERE后面直接跟AND的错误
    - **`<set>`**：自动去除末尾多余的逗号，用于UPDATE语句
**源码机制**：动态SQL在`org.apache.ibatis.scripting.xmltags`包中实现，采用**解释器模式**，每个标签对应一个具体的SqlNode实现类。
###### 4. MyBatis 中 if、choose、foreach 标签的使用场景是什么？

|**标签**​|**使用场景**​|**示例**​|**注意事项**​|
|---|---|---|---|
|**`<if>`**​|简单的条件判断，多个条件同时满足|多条件查询过滤|配合`<where>`使用避免语法错误|
|**`<choose>`**​|多选一场景，类似if-else if-else|根据不同类型采用不同查询策略|只有一个分支会被执行|
|**`<foreach>`**​|遍历集合，构建IN条件或批量操作|批量插入、批量删除、IN查询|注意collection属性值（list/array/map）|
**实战示例**：
```xml
<!-- 批量插入 -->
<insert id="batchInsert" parameterType="list">
    INSERT INTO users (name, age) VALUES
    <foreach collection="list" item="user" separator=",">
        (#{user.name}, #{user.age})
    </foreach>
</insert>

<!-- 复杂条件查询 -->
<select id="searchUsers" parameterType="map" resultType="User">
    SELECT * FROM users
    <where>
        <if test="ids != null and ids.size() > 0">
            id IN
            <foreach collection="ids" item="id" open="(" separator="," close=")">
                #{id}
            </foreach>
        </if>
        <choose>
            <when test="status != null">
                AND status = #{status}
            </when>
            <otherwise>
                AND status = 'ACTIVE'
            </otherwise>
        </choose>
    </where>
</select>
```
###### 5. MyBatis 如何处理 SQL 注入问题？
MyBatis通过多层防御机制处理SQL注入问题：
**1. 预编译机制（最主要防御）**
- **#{}占位符**：所有使用`#{}`的参数都会经过预编译处理，参数值不会改变SQL结构
- **底层实现**：基于`PreparedStatement`，数据库先编译SQL结构，再传入参数值
**2. 严格的输入验证**
```java
// 应用层验证示例
public List<User> searchUsers(SearchParams params) {
    if (params.getName() != null && !isValidName(params.getName())) {
        throw new IllegalArgumentException("Invalid name parameter");
    }
    return userMapper.searchUsers(params);
}
```
**3. 安全的${}使用规范**
- 永远不要用`${}`接收用户输入的参数值
- `${}`仅用于SQL关键字、表名、字段名等不可信源可控的场景
```xml
<!-- 安全使用${}的场景 -->
<select id="selectWithOrder" resultType="User">
    SELECT * FROM users 
    ORDER BY ${orderColumn} ${orderDirection}
</select>
```
**4. MyBatis的安全配置**
- 启用MyBatis的日志功能，监控生成的SQL
- 使用MyBatis的SQL过滤器进行参数校验
###### 6. MyBatis 中 resultType 和 resultMap 的区别是什么？

|**特性**​|**resultType**​|**resultMap**​|
|---|---|---|
|**映射方式**​|自动映射（属性名=列名）|手动映射（自定义映射关系）|
|**复杂度**​|简单场景适用|复杂场景（关联查询、继承等）|
|**灵活性**​|低|高，支持嵌套映射、集合映射等|
|**性能**​|自动映射有一定开销|精确控制，通常性能更好|
**使用场景对比**：
**resultType示例**（简单映射）：
```xml
<select id="getAllUsers" resultType="com.example.User">
    SELECT user_id, user_name, email FROM users
</select>
```
**resultMap示例**（复杂映射）：
```xml
<resultMap id="userDetailMap" type="User">
    <id property="id" column="user_id"/>
    <result property="username" column="user_name"/>
    <result property="email" column="email"/>
    <association property="department" javaType="Department">
        <id property="id" column="dept_id"/>
        <result property="name" column="dept_name"/>
    </association>
    <collection property="roles" ofType="Role">
        <id property="id" column="role_id"/>
        <result property="name" column="role_name"/>
    </collection>
</resultMap>
```
###### 7. MyBatis 如何实现结果映射？
MyBatis的结果映射是通过`ResultMap`和`ResultSetHandler`组件协作完成的：
**1. 结果映射的层次结构**：
- **基本字段映射**：`<result>`标签处理简单字段映射
- **主键映射**：`<id>`标签优化缓存和性能
- **关联映射**：`<association>`处理一对一、多对一关系
- **集合映射**：`<collection>`处理一对多关系
**2. 源码实现机制**：
在`org.apache.ibatis.executor.resultset.DefaultResultSetHandler`中：
```java
// 简化版源码逻辑
while (resultSet.next()) {
    Object rowValue = getRowValue(resultSet, mappedStatement.getResultMaps());
    // 将行数据映射到对象
}
```
**3. 自动映射原理**：
- 基于列名与属性名的匹配（下划线转驼峰）
- 可通过`mapUnderscoreToCamelCase`配置开启自动转换
###### 8. MyBatis 如何处理字段名与属性名不一致的情况？
有三种主要处理方式：
**1. 设置全局自动映射（推荐）**
```xml
<!-- mybatis-config.xml -->
<settings>
    <setting name="mapUnderscoreToCamelCase" value="true"/>
</settings>
```
自动将`user_name`映射到`userName`属性
**2. 使用resultMap手动映射**
```xml
<resultMap id="userMap" type="User">
    <id property="id" column="user_id"/>
    <result property="username" column="user_name"/>
    <result property="createTime" column="create_time"/>
</resultMap>

<select id="getUser" resultMap="userMap">
    SELECT user_id, user_name, create_time FROM users
</select>
```
**3. 使用SQL别名**
```xml
<select id="getUser" resultType="User">
    SELECT 
        user_id as id, 
        user_name as username, 
        create_time as createTime
    FROM users
</select>
```
###### 9. MyBatis 中如何实现一对一、一对多、多对多关联查询？
**一对一关联**（使用`<association>`）：
```xml
<resultMap id="orderDetailMap" type="Order">
    <id property="id" column="order_id"/>
    <result property="orderNo" column="order_no"/>
    <association property="user" javaType="User">
        <id property="id" column="user_id"/>
        <result property="username" column="username"/>
    </association>
</resultMap>
```
**一对多关联**（使用`<collection>`）：
```xml
<resultMap id="userOrderMap" type="User">
    <id property="id" column="user_id"/>
    <result property="username" column="username"/>
    <collection property="orders" ofType="Order">
        <id property="id" column="order_id"/>
        <result property="orderNo" column="order_no"/>
    </collection>
</resultMap>
```
**多对多关联**（通过中间表）：
```xml
<resultMap id="userRoleMap" type="User">
    <id property="id" column="user_id"/>
    <result property="username" column="username"/>
    <collection property="roles" ofType="Role">
        <id property="id" column="role_id"/>
        <result property="name" column="role_name"/>
    </collection>
</resultMap>

<select id="getUserWithRoles" resultMap="userRoleMap">
    SELECT u.*, r.* 
    FROM users u
    LEFT JOIN user_role ur ON u.user_id = ur.user_id
    LEFT JOIN roles r ON ur.role_id = r.role_id
    WHERE u.user_id = #{userId}
</select>
```
###### 10. MyBatis 的延迟加载是什么？如何配置？
**延迟加载**（Lazy Loading）是MyBatis的高级特性，只有在真正使用关联对象时才执行查询。
**配置方式**：
```xml
<!-- mybatis-config.xml -->
<settings>
    <setting name="lazyLoadingEnabled" value="true"/>
    <setting name="aggressiveLazyLoading" value="false"/>
</settings>

<!-- 或在resultMap中单独配置 -->
<resultMap id="userOrderMap" type="User">
    <collection property="orders" ofType="Order" 
                select="selectOrdersByUserId" column="user_id"
                fetchType="lazy"/>
</resultMap>
```
**源码机制**：延迟加载通过动态代理实现，当访问关联属性时，代理对象会触发实际的SQL查询。
###### 11. MyBatis 的 association 和 collection 标签有什么区别？

|**特性**​|**association**​|**collection**​|
|---|---|---|
|**映射关系**​|一对一、多对一|一对多、多对多|
|**Java类型**​|单个对象（`javaType`）|对象集合（`ofType`）|
|**使用场景**​|订单-用户、员工-部门|用户-订单、部门-员工|
|**嵌套查询**​|支持`select`属性|同样支持`select`属性|
**association示例**（多对一）：
```xml
<association property="department" javaType="Department" 
             select="selectDepartmentById" column="dept_id"/>
```
**collection示例**（一对多）：
```xml
<collection property="employees" ofType="Employee" 
            select="selectEmployeesByDeptId" column="dept_id"/>
```