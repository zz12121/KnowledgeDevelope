###### 1. Spring 如何集成 JDBC？

Spring 通过 **JdbcTemplate** 和**数据源抽象**来简化 JDBC 操作，核心目标是消除原生 JDBC 中那些繁琐的样板代码——连接获取、Statement 创建、异常处理、资源释放，统统帮你搞定。

配置上，需要引入 `spring-jdbc` 依赖和一个连接池（推荐 HikariCP），然后用 Java 配置类或 XML 把 `DataSource` 和 `JdbcTemplate` 注册为 Bean。Java 配置方式更现代一些，用 `@PropertySource` 加载配置文件，用 `@Bean` 方法创建 `HikariDataSource` 并设置连接参数，再把它注入 `JdbcTemplate` 即可。

```java
@Configuration
@PropertySource("classpath:jdbc.properties")
public class DataSourceConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
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

底层机制上，`JdbcTemplate` 内部使用 `DataSourceUtils` 来管理连接的生命周期，确保连接能从当前事务中获取，事务结束后正确释放，不会出现连接泄漏。

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/01、Spring数据访问#Spring集成JDBC]]

---

###### 2. 什么是 JdbcTemplate？

`JdbcTemplate` 是 Spring JDBC 模块的核心类，采用**模板方法模式**封装 JDBC 操作。你只需提供 SQL 和回调逻辑，资源管理和异常处理都由框架负责。

它有三个核心特性：**自动资源管理**（自动处理 Connection、Statement、ResultSet 的获取和释放）、**统一异常处理**（将 `SQLException` 转换为 Spring 的 `DataAccessException` 体系，让异常更有意义）、**简洁 API**（查询、更新、批量操作一行搞定）。

使用起来非常直观：

```java
@Repository
public class UserDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    // 查询单个对象
    public User getUserById(int id) {
        return jdbcTemplate.queryForObject(
            "SELECT * FROM user WHERE id = ?",
            new Object[]{id},
            (rs, rowNum) -> new User(rs.getInt("id"), rs.getString("name"))
        );
    }
    
    // 更新操作
    public void updateUser(User user) {
        jdbcTemplate.update("UPDATE user SET name = ? WHERE id = ?",
            user.getName(), user.getId());
    }
    
    // 批量插入
    public void batchInsert(List<User> users) {
        jdbcTemplate.batchUpdate("INSERT INTO user (id, name) VALUES (?, ?)",
            new BatchPreparedStatementSetter() {
                public void setValues(PreparedStatement ps, int i) throws SQLException {
                    ps.setInt(1, users.get(i).getId());
                    ps.setString(2, users.get(i).getName());
                }
                public int getBatchSize() { return users.size(); }
            });
    }
}
```

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/01、Spring数据访问#JdbcTemplate模板方法模式]]

---

###### 3. Spring 如何集成 MyBatis？

Spring 与 MyBatis 的集成通过 `mybatis-spring` 库实现，两个关键组件是 **`SqlSessionFactoryBean`**（创建 MyBatis 的 SqlSession 工厂）和 **`MapperScannerConfigurer`**（扫描 Mapper 接口并自动注册到 Spring 容器）。

Spring Boot 项目最简单，`application.yml` 里配几行就完事：

```yaml
mybatis:
  mapper-locations: classpath:mappers/*.xml
  type-aliases-package: com.example.entity
```

传统项目则需要 Java 配置类，手动创建 `SqlSessionFactoryBean` 和 `MapperScannerConfigurer`。

集成原理是：`MapperScannerConfigurer` 扫描指定包下的 Mapper 接口，为每个接口创建 `MapperFactoryBean`，而 `MapperFactoryBean` 使用 `SqlSessionTemplate` 生成 JDK 动态代理。你调用 Mapper 接口方法，实际上走的是代理对象，代理对象再去找对应的 SQL 执行。

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/01、Spring数据访问#MyBatis集成原理]]

---

###### 4. Spring 如何集成 Hibernate？

Spring 通过 **`LocalSessionFactoryBean`** 集成 Hibernate，提供声明式事务管理和统一的异常转换。配置时指定数据源、实体扫描包和 Hibernate 属性（方言、是否打印 SQL、DDL 策略），再配上 `HibernateTransactionManager` 接管事务管理即可。

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
    
    @Bean
    public PlatformTransactionManager transactionManager(SessionFactory sessionFactory) {
        return new HibernateTransactionManager(sessionFactory);
    }
}
```

不过在现代项目中，Hibernate 通常通过 Spring Data JPA 来使用，而不是直接操作 Session 对象。

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/01、Spring数据访问#Spring数据访问整体架构]]

---

###### 5. 什么是 Spring Data JPA？

Spring Data JPA 是基于 Repository 抽象的数据访问框架，核心理念是**少写代码，多干活**。你只需要定义一个接口继承 `JpaRepository`，Spring 就会自动帮你实现 CRUD、分页、排序等常用操作。

最省力的特性是**查询方法推导**：按照约定规则命名方法，比如 `findByNameAndAge`，Spring 会自动根据方法名生成对应的 JPQL。对于复杂查询，可以用 `@Query` 注解写自定义 JPQL 或原生 SQL：

```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    List<User> findByName(String name);
    Page<User> findByAgeGreaterThan(int age, Pageable pageable);
    
    @Query("SELECT u FROM User u WHERE u.email LIKE %?1%")
    List<User> findByEmailContaining(String email);
    
    @Query(value = "SELECT * FROM users WHERE status = 1", nativeQuery = true)
    List<User> findActiveUsers();
}
```

这套抽象极大简化了数据访问层的代码，让你把精力集中在业务逻辑上。

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/01、Spring数据访问#Spring-Data-JPA]]

---

###### 6. @Repository 注解的特殊作用是什么？

`@Repository` 除了标识这是一个数据访问组件（让 `@ComponentScan` 能扫描到它），还有一个更重要的核心作用：**异常转换**。

具体来说，Spring 通过 `PersistenceExceptionTranslationPostProcessor` 创建代理，拦截 `@Repository` 标注的类中抛出的持久化框架异常（无论是 Hibernate 的、MyBatis 的还是 JPA 的），统一转换为 Spring 的 `DataAccessException` 体系。

这样做的好处是，你的业务层不需要依赖具体的持久化框架，捕获 `DataAccessException` 就够了。换框架时，业务层代码不需要改动。

举个例子，捕到 `EmptyResultDataAccessException` 就知道是查无结果，捕到 `DataIntegrityViolationException` 就知道是违反约束，语义清晰得多。

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/01、Spring数据访问#异常转换机制]]

---

###### 7. DataSource 如何配置？多数据源怎么处理？

`DataSource` 是 Spring 数据访问的基础。连接池选择上，**HikariCP** 是 Spring Boot 的默认选择，性能最好、最轻量；Tomcat JDBC 适合在 Tomcat 容器里用；DBCP2 功能完整，适合老项目。

多数据源配置是实际项目中的常见需求，核心是用 `@Primary` 标记主数据源，其他数据源用 `@Qualifier` 按名称区分：

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
    
    @Bean
    public JdbcTemplate secondaryJdbcTemplate(@Qualifier("secondaryDataSource") DataSource ds) {
        return new JdbcTemplate(ds);
    }
}
```

多数据源更复杂的场景（比如动态路由读写分离）通常借助 `AbstractRoutingDataSource` 实现。

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/01、Spring数据访问#DataSource配置与多数据源]]

---

###### 8. 如何处理数据库异常？

Spring 提供统一的异常处理机制，通过 `DataAccessException` 层次结构屏蔽底层各种持久化框架的差异。常用的子类包括：`DataIntegrityViolationException`（违反约束，如唯一键冲突）、`EmptyResultDataAccessException`（查无结果）、`DeadlockLoserDataAccessException`（死锁）、`BadSqlGrammarException`（SQL 语法错误）。

最佳实践是在 Service 层捕获特定的 `DataAccessException` 子类，转为业务异常后再上抛；配合全局异常处理器 `@ControllerAdvice` 统一响应格式：

```java
@Service
@Transactional
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

@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(DataAccessException.class)
    public ResponseEntity<ErrorResponse> handleDataAccessException(DataAccessException ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("DATABASE_ERROR", "数据库操作失败"));
    }
}
```

这样既能让业务层保持对持久化框架的无感知，又能给前端返回清晰的错误信息。

📖 [[../../../../24_SpringKnowledge/06_数据访问与缓存/01、Spring数据访问#DataAccessException异常体系]]
