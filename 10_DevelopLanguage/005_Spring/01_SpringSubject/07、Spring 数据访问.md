###### 1. Spring 如何集成 JDBC？
Spring通过**JdbcTemplate**和**数据源抽象**简化JDBC操作，主要解决原生JDBC的样板代码问题。
**核心配置步骤：**
1. **添加依赖**（Maven示例）：
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>5.3.8</version>
</dependency>
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>4.0.3</version>
</dependency>
```
1. **配置数据源**（Java配置类方式）：
```java
@Configuration
@PropertySource("classpath:jdbc.properties")
public class DataSourceConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
        dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        dataSource.setUsername("root");
        dataSource.setPassword("password");
        dataSource.setMaximumPoolSize(20);
        return dataSource;
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```
1. **XML配置方式**（传统项目）：
```xml
<bean id="dataSource" class="org.apache.commons.dbcp2.BasicDataSource">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/mydatabase"/>
    <property name="username" value="root"/>
    <property name="password" value="password"/>
</bean>

<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
    <property name="dataSource" ref="dataSource"/>
</bean>
```
**源码机制**：Spring通过`JdbcTemplate`封装JDBC操作，内部使用`DataSourceUtils`管理连接生命周期，确保连接从当前事务获取或创建新连接。
###### 2. 什么是 JdbcTemplate？
`JdbcTemplate`是Spring JDBC模块的核心类，采用**模板方法模式**封装JDBC操作。其设计目标是通过回调机制消除样板代码，提供一致的异常处理。
**核心特性：**
- **自动资源管理**：自动处理Connection、Statement、ResultSet的获取和释放
- **统一的异常处理**：将`SQLException`转换为Spring的`DataAccessException`体系
- **简化的API**：提供查询、更新、批量操作等便捷方法
**源码分析**：
```java
// JdbcTemplate核心执行方法
public <T> T execute(StatementCallback<T> action) throws DataAccessException {
    Connection con = DataSourceUtils.getConnection(obtainDataSource());
    Statement stmt = null;
    try {
        stmt = con.createStatement();
        applyStatementSettings(stmt);
        T result = action.doInStatement(stmt);  // 回调用户代码
        handleWarnings(stmt);
        return result;
    } catch (SQLException ex) {
        // 释放资源并转换异常
        JdbcUtils.closeStatement(stmt);
        DataSourceUtils.releaseConnection(con, getDataSource());
        throw translateException("StatementCallback", getSql(action), ex);
    }
}
```
**常用操作示例**：
```java
@Repository
public class UserDao {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    // 查询单个对象
    public User getUserById(int id) {
        String sql = "SELECT * FROM user WHERE id = ?";
        return jdbcTemplate.queryForObject(sql, new Object[]{id}, 
            (rs, rowNum) -> new User(rs.getInt("id"), rs.getString("name")));
    }
    
    // 更新操作
    public void updateUser(User user) {
        String sql = "UPDATE user SET name = ? WHERE id = ?";
        jdbcTemplate.update(sql, user.getName(), user.getId());
    }
    
    // 批量操作
    public void batchInsert(List<User> users) {
        String sql = "INSERT INTO user (id, name) VALUES (?, ?)";
        jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                User user = users.get(i);
                ps.setInt(1, user.getId());
                ps.setString(2, user.getName());
            }
            public int getBatchSize() { return users.size(); }
        });
    }
}
```
###### 3. Spring 如何集成 MyBatis？
Spring与MyBatis集成主要通过`mybatis-spring`库实现，关键组件是`SqlSessionFactoryBean`和`MapperScannerConfigurer`。
**配置方式：**
1. **Spring Boot自动配置**：
```yaml
# application.yml
mybatis:
  mapper-locations: classpath:mappers/*.xml
  type-aliases-package: com.example.entity
```
1. **Java配置类**：
```java
@Configuration
public class MyBatisConfig {
    
    @Bean
    public SqlSessionFactoryBean sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        sessionFactory.setDataSource(dataSource);
        sessionFactory.setTypeAliasesPackage("com.example.entity");
        sessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver()
            .getResources("classpath:mappers/*.xml"));
        return sessionFactory;
    }
    
    @Bean
    public MapperScannerConfigurer mapperScannerConfigurer() {
        MapperScannerConfigurer scanner = new MapperScannerConfigurer();
        scanner.setBasePackage("com.example.mapper");
        scanner.setAnnotationClass(org.apache.ibatis.annotations.Mapper.class);
        return scanner;
    }
}
```
**集成原理**：Spring通过`MapperScannerConfigurer`扫描Mapper接口，为每个接口创建`MapperFactoryBean`，后者使用`SqlSessionTemplate`生成JDK动态代理。
###### 4. Spring 如何集成 Hibernate？
Spring通过`LocalSessionFactoryBean`集成Hibernate，提供声明式事务管理和异常转换。
**配置示例：**
```java
@Configuration
@EnableTransactionManagement
public class HibernateConfig {
    
    @Bean
    public LocalSessionFactoryBean sessionFactory(DataSource dataSource) {
        LocalSessionFactoryBean sessionFactory = new LocalSessionFactoryBean();
        sessionFactory.setDataSource(dataSource);
        sessionFactory.setPackagesToScan("com.example.entity");
        sessionFactory.setHibernateProperties(hibernateProperties());
        return sessionFactory;
    }
    
    private Properties hibernateProperties() {
        Properties props = new Properties();
        props.put("hibernate.dialect", "org.hibernate.dialect.MySQL5Dialect");
        props.put("hibernate.show_sql", true);
        props.put("hibernate.hbm2ddl.auto", "update");
        return props;
    }
    
    @Bean
    public PlatformTransactionManager transactionManager(SessionFactory sessionFactory) {
        return new HibernateTransactionManager(sessionFactory);
    }
}
```
###### 5. 什么是 Spring Data JPA？
Spring Data JPA是基于Repository抽象的数据访问框架，减少JPA样板代码。其核心是`Repository`接口和查询方法推导机制。
**核心特性：**
- **Repository抽象**：提供`CrudRepository`、`PagingAndSortingRepository`等基础接口
- **查询方法推导**：根据方法名自动生成查询，如`findByName(String name)`
- **@Query注解**：支持自定义JPQL或原生SQL查询
**使用示例：**
```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    // 方法名推导查询
    List<User> findByName(String name);
    List<User> findByNameContainingIgnoreCase(String name);
    
    // 分页查询
    Page<User> findByAgeGreaterThan(int age, Pageable pageable);
    
    // 自定义查询
    @Query("SELECT u FROM User u WHERE u.email LIKE %?1%")
    List<User> findByEmailContaining(String email);
    
    // 原生SQL查询
    @Query(value = "SELECT * FROM users WHERE status = 1", nativeQuery = true)
    List<User> findActiveUsers();
}
```
###### 6. @Repository 注解的特殊作用是什么？
`@Repository`除了标识数据访问组件，核心作用是**异常转换**。通过`PersistenceExceptionTranslationPostProcessor`将持久化框架特定异常统一转换为Spring的`DataAccessException`。
**源码机制**：
```java
// PersistenceExceptionTranslationPostProcessor核心逻辑
public class PersistenceExceptionTranslationPostProcessor implements BeanPostProcessor {
    
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // 检查是否为@Repository注解的类
        if (this.exceptionTranslator != null && isCandidateForTranslation(bean)) {
            // 创建代理包装异常转换逻辑
            return createExceptionTranslatingProxy(bean);
        }
        return bean;
    }
}
```
**异常转换示例**：
```java
@Repository
public class UserRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public User findById(Long id) {
        try {
            return jdbcTemplate.queryForObject("SELECT * FROM users WHERE id = ?", 
                new Object[]{id}, User.class);
        } catch (DataAccessException ex) {
            // Spring已将SQLException转换为有意义的异常
            if (ex instanceof EmptyResultDataAccessException) {
                throw new UserNotFoundException("User not found: " + id);
            }
            throw ex;
        }
    }
}
```
###### 7. DataSource 如何配置？
DataSource配置是Spring数据访问的基础，支持多种连接池和配置方式。
**连接池选择比较：**

|连接池|特点|适用场景|
|---|---|---|
|**HikariCP**​|高性能、轻量级|Spring Boot默认，生产环境首选|
|**Tomcat JDBC**​|稳定性好|Tomcat容器内置|
|**DBCP2**​|功能完整|传统项目，需要丰富配置选项|
**多数据源配置：**
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
    public JdbcTemplate primaryJdbcTemplate(@Qualifier("primaryDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
    
    @Bean
    public JdbcTemplate secondaryJdbcTemplate(@Qualifier("secondaryDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```
###### 8. 如何处理数据库异常？
Spring提供统一的异常处理机制，将各种持久化技术的异常转换为一致的`DataAccessException`层次结构。
**异常层次结构：**
```
DataAccessException
├── CleanupFailureDataAccessException
├── DataAccessResourceFailureException
├── DataIntegrityViolationException
├── DataRetrievalFailureException
├── DeadlockLoserDataAccessException
├── IncorrectUpdateSemanticsDataAccessException
└── InvalidDataAccessResourceUsageException
    ├── BadSqlGrammarException
    ├── InvalidResultSetAccessException
    └── ...
```
**最佳实践：**
```java
@Service
@Transactional
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public User createUser(User user) {
        try {
            return userRepository.save(user);
        } catch (DataIntegrityViolationException ex) {
            // 处理唯一约束违反等数据完整性异常
            throw new BusinessException("用户名已存在", ex);
        } catch (DeadlockLoserDataAccessException ex) {
            // 处理死锁异常，通常需要重试
            throw new RetryableException("系统繁忙，请重试", ex);
        }
    }
    
    // 使用声明式事务管理
    @Transactional(rollbackFor = Exception.class)
    public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
        // 业务逻辑，异常时自动回滚
        userRepository.decreaseBalance(fromId, amount);
        userRepository.increaseBalance(toId, amount);
    }
}
```
**全局异常处理：**
```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(DataAccessException.class)
    public ResponseEntity<ErrorResponse> handleDataAccessException(DataAccessException ex) {
        ErrorResponse error = new ErrorResponse("DATABASE_ERROR", "数据库操作失败");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException ex) {
        ErrorResponse error = new ErrorResponse("BUSINESS_ERROR", ex.getMessage());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }
}
```