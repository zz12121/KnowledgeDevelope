# Spring 数据访问

## 1 Spring JDBC 集成

Spring 通过 `JdbcTemplate` 和数据源抽象简化原生 JDBC 操作，消除大量样板代码（获取连接、创建 Statement、处理 ResultSet、关闭资源）。

**配置步骤**：

```java
@Configuration
@PropertySource("classpath:jdbc.properties")
public class DataSourceConfig {

    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setDriverClassName("com.mysql.cj.jdbc.Driver");
        ds.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        ds.setUsername("root");
        ds.setPassword("password");
        ds.setMaximumPoolSize(20);
        ds.setMinimumIdle(5);
        return ds;
    }

    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

`JdbcTemplate` 内部使用 `DataSourceUtils.getConnection()` 获取连接，该方法会尝试从当前事务上下文中取连接，确保事务内复用同一连接。

## 2 JdbcTemplate 核心用法

`JdbcTemplate` 采用**模板方法模式**，提供统一的资源管理和异常处理，开发者只需关注 SQL 和结果映射：

```java
@Repository
public class UserDao {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    // 查询单个对象（结果为空时抛 EmptyResultDataAccessException）
    public User getUserById(long id) {
        String sql = "SELECT * FROM user WHERE id = ?";
        return jdbcTemplate.queryForObject(sql, new Object[]{id},
            (rs, rowNum) -> new User(rs.getLong("id"), rs.getString("name")));
    }

    // 查询列表
    public List<User> getAllUsers() {
        return jdbcTemplate.query("SELECT * FROM user",
            (rs, rowNum) -> new User(rs.getLong("id"), rs.getString("name")));
    }

    // 更新/插入/删除（返回影响行数）
    public int updateUser(User user) {
        return jdbcTemplate.update("UPDATE user SET name = ? WHERE id = ?",
            user.getName(), user.getId());
    }

    // 批量操作
    public void batchInsert(List<User> users) {
        jdbcTemplate.batchUpdate("INSERT INTO user (id, name) VALUES (?, ?)",
            new BatchPreparedStatementSetter() {
                public void setValues(PreparedStatement ps, int i) throws SQLException {
                    ps.setLong(1, users.get(i).getId());
                    ps.setString(2, users.get(i).getName());
                }
                public int getBatchSize() { return users.size(); }
            });
    }

    // 查询单个值
    public int getUserCount() {
        return jdbcTemplate.queryForObject("SELECT COUNT(*) FROM user", Integer.class);
    }
}
```

**异常处理**：`JdbcTemplate` 将 `SQLException` 转换为 Spring 的 `DataAccessException` 体系（unchecked exception），业务代码无需 catch `SQLException`。

## 3 Spring 集成 MyBatis

通过 `mybatis-spring` 库集成，核心组件是 `SqlSessionFactoryBean` 和 `MapperScannerConfigurer`：

```java
@Configuration
public class MyBatisConfig {

    @Bean
    public SqlSessionFactoryBean sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(dataSource);
        factory.setTypeAliasesPackage("com.example.entity");
        factory.setMapperLocations(
            new PathMatchingResourcePatternResolver().getResources("classpath:mappers/*.xml"));
        return factory;
    }

    @Bean
    public MapperScannerConfigurer mapperScannerConfigurer() {
        MapperScannerConfigurer scanner = new MapperScannerConfigurer();
        scanner.setBasePackage("com.example.mapper");
        scanner.setAnnotationClass(Mapper.class);
        return scanner;
    }
}
```

**集成原理**：`MapperScannerConfigurer` 扫描 Mapper 接口 → 为每个接口创建 `MapperFactoryBean` → `MapperFactoryBean` 使用 `SqlSessionTemplate` 生成 JDK 动态代理 → 调用 Mapper 方法时通过代理执行 MyBatis SQL。

Spring Boot 场景下 `mybatis-spring-boot-starter` 会自动完成上述配置，只需在 `application.yml` 中配置扫描路径。

## 4 Spring Data JPA

Spring Data JPA 基于 Repository 抽象，极大减少数据访问样板代码：

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // 方法名推导查询
    List<User> findByName(String name);
    List<User> findByNameContainingIgnoreCase(String name);

    // 分页查询
    Page<User> findByAgeGreaterThan(int age, Pageable pageable);

    // 自定义 JPQL
    @Query("SELECT u FROM User u WHERE u.email LIKE %?1%")
    List<User> findByEmailContaining(String email);

    // 原生 SQL
    @Query(value = "SELECT * FROM users WHERE status = 1", nativeQuery = true)
    List<User> findActiveUsers();
}
```

Repository 接口不需要实现类，Spring Data 在运行时通过 JDK 动态代理生成实现，方法名推导遵循严格的命名规则。

## 5 连接池选型

生产环境首选 **HikariCP**（Spring Boot 2.x 默认），性能最佳：

**核心配置参数**：
- `maximumPoolSize`：最大连接数，建议 = CPU 核数 × 2 + 1
- `minimumIdle`：最小空闲连接数
- `connectionTimeout`：获取连接超时时间（毫秒）
- `idleTimeout`：空闲连接存活时间
- `maxLifetime`：连接最大存活时间（需小于数据库 `wait_timeout`）

**多数据源配置**：

```java
@Configuration
public class MultiDataSourceConfig {

    @Bean
    @Primary
    @ConfigurationProperties("app.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @ConfigurationProperties("app.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @Primary
    public JdbcTemplate primaryJdbcTemplate(@Qualifier("primaryDataSource") DataSource ds) {
        return new JdbcTemplate(ds);
    }
}
```

## 6 Spring 数据访问异常体系

Spring 将各种持久化框架的异常统一转换为 `DataAccessException`（unchecked），调用方无需处理框架特定异常：

```
DataAccessException
├── DataIntegrityViolationException    → 唯一约束、外键约束违反
├── DeadlockLoserDataAccessException   → 死锁检测
├── DataAccessResourceFailureException → 连接失败等资源问题
└── InvalidDataAccessResourceUsageException
    ├── BadSqlGrammarException         → SQL 语法错误
    └── InvalidResultSetAccessException → 结果集访问错误
```

实际业务中按异常类型分别处理：

```java
@Service
public class UserService {

    public User createUser(User user) {
        try {
            return userRepository.save(user);
        } catch (DataIntegrityViolationException ex) {
            throw new BusinessException("用户名已存在", ex);
        } catch (DeadlockLoserDataAccessException ex) {
            throw new RetryableException("系统繁忙，请重试", ex);
        }
    }
}
```

`@Repository` 注解的 `PersistenceExceptionTranslationPostProcessor` 机制负责将 Hibernate/MyBatis 等原生异常转换为 `DataAccessException`。

---

相关面试题 → [[../../10_DevelopLanguage/005_Spring/01_SpringSubject/07、Spring 数据访问|07、Spring数据访问]]
