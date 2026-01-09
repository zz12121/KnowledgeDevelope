###### 1. 什么是 MyBatis-Plus？它有什么作用？
MyBatis-Plus（简称MP）是一个**MyBatis的增强工具**，在MyBatis的基础上只做增强不做改变，旨在简化开发、提高效率。其核心作用是**通过封装通用CRUD操作、提供条件构造器等功能，大幅减少传统MyBatis开发中的重复代码编写**。
**核心特性与价值**：
- **无侵入性**：不修改MyBatis源码，可共存使用，支持随时切换回原生MyBatis操作。
- **通用CRUD自动化**：内置`BaseMapper`和`IService`接口，单表CRUD操作无需编写SQL语句。
- **强大的条件构造器**：通过`Wrapper`体系用Java代码构建复杂查询条件，避免SQL拼接错误。
- **内置插件体系**：提供分页、乐观锁、性能分析等实用插件。
- **代码生成器**：可自动生成Entity、Mapper、Service、Controller等全套代码。
**设计理念**：MP秉承"为简化开发而生"的宗旨，通过封装通用技术细节，让开发者更专注于业务逻辑实现。
###### 2. MyBatis-Plus 与 MyBatis 有哪些区别？

|**对比维度**​|**MyBatis**​|**MyBatis-Plus**​|
|---|---|---|
|**CRUD开发**​|需手动编写SQL和XML|**自动生成**，继承`BaseMapper`即可|
|**条件构造**​|手动拼接字符串|**Wrapper条件构造器**，链式编程|
|**分页功能**​|手动实现（Limit）|**内置分页插件**，自动优化|
|**代码生成**​|无内置支持|**内置代码生成器**，一键生成|
|**SQL注入防护**​|需开发者自行注意|**条件构造器减少拼接风险**|
**关系定位**：MP是MyBatis的**增强套件**而非替代品。复杂SQL（如多表关联、存储过程）仍可使用原生MyBatis的XML方式实现，体现了MP的兼容性。
###### 3. MyBatis-Plus 的通用 Mapper 有哪些方法？
`BaseMapper<T>`接口提供了完整的单表CRUD方法：
**插入操作**
```java
int insert(T entity);
```
**删除操作**
```java
int deleteById(Serializable id); // 根据ID删除
int deleteByMap(Map<String,Object> columnMap); // 根据字段Map删除
int delete(@Param(Constants.WRAPPER) Wrapper<T> wrapper); // 条件删除
int deleteBatchIds(@Param(Constants.COLLECTION) Collection<? extends Serializable> idList); // 批量删除
```
**更新操作**
```java
int updateById(@Param(Constants.ENTITY) T entity); // 根据ID更新
int update(@Param(Constants.ENTITY) T entity, @Param(Constants.WRAPPER) Wrapper<T> updateWrapper); // 条件更新
```
**查询操作**
```java
T selectById(Serializable id); // ID查询
List<T> selectBatchIds(Collection<? extends Serializable> idList); // 批量ID查询
List<T> selectByMap(Map<String,Object> columnMap); // 字段Map查询
T selectOne(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper); // 条件查询单条
List<T> selectList(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper); // 条件列表查询
IPage<T> selectPage(IPage<T> page, @Param(Constants.WRAPPER) Wrapper<T> queryWrapper); // 分页查询
Integer selectCount(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper); // 计数查询
```
###### 4. MyBatis-Plus 的条件构造器（Wrapper）如何使用？
Wrapper是MP的核心功能，用于构建复杂查询条件。
**Wrapper体系结构**：
复制
```
Wrapper (接口)
├── AbstractWrapper (抽象类)
│   ├── QueryWrapper<T> // 查询条件
│   └── UpdateWrapper<T> // 更新条件
└── LambdaWrapper<T>
    ├── LambdaQueryWrapper<T> // Lambda查询
    └── LambdaUpdateWrapper<T> // Lambda更新
```
**QueryWrapper示例**：
```java
// 创建查询条件
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.select("id", "name", "age") // 指定字段
      .eq("age", 18) // age = 18
      .like("name", "张") // name LIKE '%张%'
      .gt("create_time", "2024-01-01") // create_time > '2024-01-01'
      .orderByDesc("create_time"); // 按创建时间降序

List<User> users = userMapper.selectList(wrapper);
```
**LambdaQueryWrapper（推荐）**：
```java
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getAge, 18)
       .like(User::getName, "张")
       .between(User::getCreateTime, startDate, endDate)
       .orderByDesc(User::getCreateTime);

List<User> users = userMapper.selectList(wrapper);
```
**优势**：编译期检查字段名，避免硬编码错误；重命名时自动更新。
**UpdateWrapper更新示例**：
```java
UpdateWrapper<User> updateWrapper = new UpdateWrapper<>();
updateWrapper.eq("status", 0)
             .set("status", 1)
             .set("update_time", new Date());

userMapper.update(null, updateWrapper);
```
###### 5. MyBatis-Plus 的分页插件如何配置和使用？
**配置分页插件**：
```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 添加分页拦截器，指定数据库类型
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```
**分页查询使用**：
```java
// 创建分页对象（当前页，每页大小）
Page<User> page = new Page<>(1, 10);

// 构建查询条件（可选）
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.gt(User::getAge, 18);

// 执行分页查询
IPage<User> result = userMapper.selectPage(page, wrapper);

// 获取分页信息
System.out.println("总记录数：" + result.getTotal());
System.out.println("总页数：" + result.getPages());
System.out.println("当前页数据：" + result.getRecords());
```
**源码机制**：分页插件通过拦截Executor的query方法，自动在执行SQL前添加数据库特定的分页语句（如MySQL的LIMIT）。
###### 6. MyBatis-Plus 的代码生成器如何使用？
代码生成器可自动生成Entity、Mapper、Service、Controller等代码。
**基本配置示例**：
```java
public class CodeGenerator {
    public static void main(String[] args) {
        AutoGenerator generator = new AutoGenerator(dataSourceConfig());
        
        // 全局配置
        GlobalConfig globalConfig = new GlobalConfig.Builder()
            .outputDir(System.getProperty("user.dir") + "/src/main/java")
            .author("YourName")
            .fileOverride(true) // 覆盖已生成文件
            .build();
        
        // 包配置
        PackageConfig packageConfig = new PackageConfig.Builder()
            .parent("com.example.project")
            .entity("entity")
            .mapper("mapper")
            .service("service")
            .controller("controller")
            .build();
            
        // 策略配置
        StrategyConfig strategyConfig = new StrategyConfig.Builder()
            .addInclude("user", "order") // 要生成的表
            .entityBuilder()
            .enableLombok() // 启用Lombok
            .naming(NamingStrategy.underline_to_camel) // 下划线转驼峰
            .build();
            
        generator.global(globalConfig)
                 .packageInfo(packageConfig)
                 .strategy(strategyConfig)
                 .execute();
    }
    
    private static DataSourceConfig dataSourceConfig() {
        return new DataSourceConfig.Builder(
            "jdbc:mysql://localhost:3306/test",
            "root",
            "password"
        ).build();
    }
}
```
###### 7. MyBatis-Plus 的逻辑删除功能是什么？如何实现？
逻辑删除是**用字段标记记录删除状态而非物理删除数据**，避免数据永久丢失。
**实现步骤**：
**实体类配置**：
```java
@Data
public class User {
    @TableLogic
    // 删除值=1，未删除值=0（默认值，可配置）
    private Integer deleted;
}
```
**全局配置**（application.yml）：
```yml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted # 逻辑删除字段名
      logic-delete-value: 1 # 已删除值
      logic-not-delete-value: 0 # 未删除值
```
**使用效果**：
- 执行`deleteById()`时，MP会自动更新`deleted=1`而非物理删除
- 执行`selectList()`等查询时，MP自动添加`WHERE deleted=0`条件
###### 8. MyBatis-Plus 的自动填充功能如何使用？
自动填充用于**自动设置创建时间、更新时间等字段**。
**实体类配置**：
```java
@Data
public class User {
    @TableField(fill = FieldFill.INSERT) // 插入时填充
    private LocalDateTime createTime;
    
    @TableField(fill = FieldFill.INSERT_UPDATE) // 插入和更新时填充
    private LocalDateTime updateTime;
}
```
**实现MetaObjectHandler**：
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
###### 9. MyBatis-Plus 的主键生成策略有哪些？
通过`@TableId`注解配置主键策略：
```java
@Data
public class User {
    @TableId(type = IdType.AUTO) // 数据库自增
    private Long id;
}
```
**常用策略**：
- **AUTO**：数据库自增（MySQL）
- **ASSIGN_ID**：雪花算法生成Long类型ID
- **ASSIGN_UUID**：生成UUID字符串
- **INPUT**：用户自定义输入
**全局配置**：
```yml
mybatis-plus:
  global-config:
    db-config:
      id-type: auto # 全局主键策略
```
###### 10. MyBatis-Plus 如何实现乐观锁？
乐观锁通过**版本号机制**解决并发更新问题。
**实现步骤**：
**实体类添加版本字段**：
```java
@Data
public class User {
    @Version
    private Integer version; // 版本号
}
```
**更新操作流程**：
```java
// 1. 先查询获取当前版本
User user = userMapper.selectById(1L);

// 2. 修改字段
user.setName("New Name");

// 3. 执行更新（MP会自动比较版本号）
// UPDATE user SET name='New Name', version=2 WHERE id=1 AND version=1
userMapper.updateById(user);
```
如果版本号不匹配，更新返回0，表示数据已被其他线程修改。
###### 11. MyBatis-Plus 的性能分析插件如何使用？
性能分析插件用于**监控SQL执行时间**，帮助优化慢SQL。
**配置插件**：
```java
@Bean
@Profile({"dev", "test"}) // 仅在开发测试环境启用
public PerformanceInterceptor performanceInterceptor() {
    PerformanceInterceptor interceptor = new PerformanceInterceptor();
    interceptor.setMaxTime(1000); // 设置SQL最大执行时间（ms）
    interceptor.setFormat(true); // 是否格式化SQL
    return interceptor;
}
```
**输出示例**：
复制
```
Time：48 ms - ID：com.example.mapper.UserMapper.selectById
Execute SQL：SELECT id,name,age FROM user WHERE id=1;
```
###### 12. MyBatis-Plus 如何实现多租户功能？
多租户实现**数据隔离**，常见方案有：
- **独立数据库**：每个租户独立数据库
- **共享数据库独立Schema**：相同DB，不同Schema
- **共享数据表**：相同表，通过租户ID区分
**MP多租户配置**：
```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    
    // 添加多租户拦截器
    TenantLineInnerInterceptor tenantInterceptor = new TenantLineInnerInterceptor();
    tenantInterceptor.setTenantLineHandler(new TenantLineHandler() {
        @Override
        public Expression getTenantId() {
            // 从当前上下文获取租户ID
            return new LongValue(1L);
        }
        
        @Override
        public String getTenantIdColumn() {
            return "tenant_id"; // 租户ID列名
        }
        
        @Override
        public boolean ignoreTable(String tableName) {
            // 忽略不需要多租户的表
            return "common_table".equals(tableName);
        }
    });
    
    interceptor.addInnerInterceptor(tenantInterceptor);
    return interceptor;
}
```
**效果**：MP自动在所有SQL中添加`WHERE tenant_id = ?`条件，实现数据自动隔离。