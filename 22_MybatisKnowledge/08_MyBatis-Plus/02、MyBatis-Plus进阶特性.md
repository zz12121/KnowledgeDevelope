# MyBatis-Plus 进阶特性

## 1. 逻辑删除

逻辑删除用**字段标记**代替物理 DELETE，保留历史数据，避免误操作导致数据永久丢失。

**配置方式**：

```java
@Data
public class User {
    @TableLogic  // 标记逻辑删除字段
    private Integer deleted;
}
```

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted   # 逻辑删除字段名
      logic-delete-value: 1         # 已删除值
      logic-not-delete-value: 0     # 未删除值
```

**效果**：
- 执行 `deleteById()` → 实际执行 `UPDATE user SET deleted=1 WHERE id=?`
- 执行 `selectList()` → 自动追加 `WHERE deleted=0` 条件
- **注意**：逻辑删除字段不会出现在 `UPDATE` 的条件中（避免"删除后还能更新"的逻辑矛盾）

---

## 2. 自动填充

自动填充用于在插入或更新时，**自动设置公共字段**（如创建时间、更新时间、操作人）。

**实体类配置**：

```java
@Data
public class User {
    @TableField(fill = FieldFill.INSERT)         // 仅插入时填充
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)  // 插入和更新时填充
    private LocalDateTime updateTime;
}
```

**实现 MetaObjectHandler**：

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

`strictInsertFill` 只有在字段值为 null 时才填充（"strict" 的含义），避免覆盖业务显式设置的值。

---

## 3. 乐观锁

乐观锁通过**版本号机制**解决并发更新冲突：更新时检查版本号是否与读取时一致，不一致说明已被其他线程修改，本次更新失败。

**配置**：

```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        return interceptor;
    }
}
```

**实体类**：

```java
@Data
public class User {
    @Version  // 版本号字段
    private Integer version;
}
```

**使用流程**：

```java
// 1. 先查询获取当前版本
User user = userMapper.selectById(1L);
// user.version = 1

// 2. 修改字段
user.setName("New Name");

// 3. 执行更新（MP 自动在 WHERE 中加版本号条件）
// 实际执行：UPDATE user SET name='New Name', version=2 WHERE id=1 AND version=1
int rows = userMapper.updateById(user);

// rows = 0 表示版本号不匹配，说明已被其他线程修改
if (rows == 0) {
    throw new OptimisticLockException("数据已被其他人修改，请刷新后重试");
}
```

**特点**：不需要加数据库锁，适合读多写少、冲突概率低的场景。

---

## 4. 代码生成器

代码生成器可以根据数据库表结构，一键生成 Entity、Mapper、Service、Controller 全套代码。

```java
public class CodeGenerator {
    public static void main(String[] args) {
        AutoGenerator generator = new AutoGenerator(dataSourceConfig());

        GlobalConfig globalConfig = new GlobalConfig.Builder()
            .outputDir(System.getProperty("user.dir") + "/src/main/java")
            .author("YourName")
            .build();

        PackageConfig packageConfig = new PackageConfig.Builder()
            .parent("com.example.project")
            .entity("entity")
            .mapper("mapper")
            .service("service")
            .controller("controller")
            .build();

        StrategyConfig strategyConfig = new StrategyConfig.Builder()
            .addInclude("user", "order")      // 指定要生成的表
            .entityBuilder()
            .enableLombok()                   // 使用 Lombok
            .naming(NamingStrategy.underline_to_camel)  // 下划线转驼峰
            .build();

        generator.global(globalConfig)
                 .packageInfo(packageConfig)
                 .strategy(strategyConfig)
                 .execute();
    }

    private static DataSourceConfig dataSourceConfig() {
        return new DataSourceConfig.Builder(
            "jdbc:mysql://localhost:3306/test", "root", "password"
        ).build();
    }
}
```

---

## 5. 多租户

MP 内置多租户插件，可自动在所有 SQL 中追加租户隔离条件，实现**共享数据库、共享数据表**方案的数据隔离。

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

    TenantLineInnerInterceptor tenantInterceptor = new TenantLineInnerInterceptor();
    tenantInterceptor.setTenantLineHandler(new TenantLineHandler() {
        @Override
        public Expression getTenantId() {
            // 从请求上下文获取当前租户 ID（如 ThreadLocal 或 JWT Token）
            return new LongValue(TenantContext.getCurrentTenantId());
        }

        @Override
        public String getTenantIdColumn() {
            return "tenant_id";
        }

        @Override
        public boolean ignoreTable(String tableName) {
            // 公共表不需要租户隔离
            return "sys_config".equals(tableName);
        }
    });

    interceptor.addInnerInterceptor(tenantInterceptor);
    return interceptor;
}
```

效果：查询时自动追加 `WHERE tenant_id = ?`，插入时自动设置 `tenant_id`。

---

## 6. MyBatis vs MyBatis-Plus 对比

| 对比维度 | MyBatis | MyBatis-Plus |
|----------|---------|--------------|
| 单表 CRUD | 需手动编写 SQL + XML | 继承 `BaseMapper` 即可，零 SQL |
| 条件构造 | 手动拼接字符串，容易出错 | Wrapper 链式构建，编译期检查 |
| 分页 | 手动写 LIMIT，或集成 PageHelper | 内置分页插件，自动优化 |
| 逻辑删除 | 手动在 SQL 加 `WHERE deleted=0` | `@TableLogic` 自动处理 |
| 乐观锁 | 手动在 SQL 加版本号条件 | `@Version` 自动处理 |
| 自动填充 | 手动在 Service 层设置时间 | `@TableField(fill=...)` 自动处理 |
| 代码生成 | 无内置支持 | 内置代码生成器 |
| 复杂 SQL | 完整支持 | 复用 MyBatis 原生 XML |

---

## 相关面试题

- [[003_ORM/01_MyBatisSubject/11、MyBatis-Plus|📖 11、MyBatis-Plus]]
