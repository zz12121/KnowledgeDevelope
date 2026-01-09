###### 1. MyBatis 如何与 Spring 集成？
MyBatis与Spring集成主要通过**MyBatis-Spring**桥接模块实现，该模块将MyBatis的核心组件生命周期交由Spring容器管理。集成过程包含以下关键步骤：
**依赖配置**
在pom.xml中添加必要依赖：
```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis-spring</artifactId>
    <version>2.0.6</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>5.3.12</version>
</dependency>
```
**核心配置环节**
- **数据源配置**：通过Spring管理数据库连接池（如Druid、HikariCP）
- **SqlSessionFactoryBean**：替代MyBatis原生的SqlSessionFactoryBuilder，创SqlSessionFactory实例
- **MapperScannerConfigurer**：自动扫描Mapper接口并注册为Spring Bean
**XML配置示例**
```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mapperLocations" value="classpath:mapper/*.xml"/>
</bean>

<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="com.example.mapper"/>
</bean>
```
**源码级集成机制**
在传统MyBatis中，SqlSession需手动管理生命周期。集成后，SqlSessionFactoryBean通过实现InitializingBean接口，在afterPropertiesSet()方法中调用buildSqlSessionFactory()创建SqlSessionFactory实例。MapperScannerConfigurer作为BeanDefinitionRegistryPostProcessor，在容器启动时扫描指定包路径，将Mapper接口的BeanClass设置为MapperFactoryBean，从而通过动态代理生成实际Mapper实例。
###### 2. MyBatis-Spring 的核心组件有哪些？
**SqlSessionFactoryBean**
- **角色**：SqlSessionFactory的工厂Bean，替代原生SqlSessionFactoryBuilder
- **核心职责**：解析MyBatis配置文件和映射文件，构建Configuration对象并创建SqlSessionFactory实例
- **源码关键方法**：afterPropertiesSet()中调用buildSqlSessionFactory()完成初始化
**SqlSessionTemplate**
- **角色**：SqlSession的线程安全替代品，是MyBatis-Spring整合的核心
- **核心特性**：
    - 通过SqlSessionInterceptor拦截器管理SqlSession生命周期
    - 自动与Spring事务管理集成，事务内复用相同SqlSession
    - 非事务环境下自动关闭连接，避免资源泄漏
- **源码机制**：内部通过JDK动态代理创建sqlSessionProxy，其InvocationHandler会从TransactionSynchronizationManager获取当前事务关联的SqlSession
**MapperFactoryBean**
- **角色**：单个Mapper接口的工厂Bean
- **核心方法**：getObject()返回Mapper接口的代理对象（实际调用getSqlSession().getMapper()）
- **父类SqlSessionDaoSupport**：提供sqlSessionTemplate的依赖注入支持
**MapperScannerConfigurer**
- **角色**：自动化批量注册Mapper接口
- **工作机制**：通过ClassPathMapperScanner扫描指定包路径，修改BeanDefinition的beanClass为MapperFactoryBean，并注入对应接口类型作为构造参数
###### 3. Spring Boot 如何整合 MyBatis？
Spring Boot通过**自动配置**极大简化了MyBatis集成流程。
**依赖配置**
```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>
```
**配置文件示例（application.yml）**
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver

mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.example.entity
  configuration:
    map-underscore-to-camel-case: true
```
**启动类配置**
```java
@SpringBootApplication
@MapperScan("com.example.mapper")
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
**自动配置原理**
- MyBatisAutoConfiguration自动配置SqlSessionFactory和SqlSessionTemplate
- @MapperScan注解触发MapperScannerRegistrar，批量注册Mapper接口
- 通过MybatisProperties绑定配置参数，如mapper-locations、type-aliases-package等
###### 4. @MapperScan 注解的作用是什么？
@MapperScan是Spring整合MyBatis中的**核心注解**，用于批量扫描和注册Mapper接口。
**核心功能**
- 自动扫描指定包路径下的Mapper接口并注册为Spring Bean
- 替代手动在每个Mapper接口上添加@Mapper注解的繁琐配置
- 支持多数据源场景下的sqlSessionFactoryRef和sqlSessionTemplateRef指定
**配置示例**
```java
@Configuration
@MapperScan(
    basePackages = "com.example.mapper",
    sqlSessionFactoryRef = "sqlSessionFactory",
    lazyInitialization = "true" // 延迟初始化优化启动性能
)
public class MyBatisConfig {}
```
**源码级工作机制**
1. **扫描阶段**：通过MapperScannerRegistrar（ImportBeanDefinitionRegistrar实现）创建ClassPathMapperScanner
2. **过滤注册**：扫描器过滤出接口类型，修改BeanDefinition的beanClass为MapperFactoryBean，并设置Mapper接口作为构造参数
3. **代理生成**：Spring容器初始化时，通过MapperFactoryBean的getObject()方法生成Mapper代理对象
**多数据源支持**
```java
@MapperScan(
    basePackages = "com.example.primary.mapper", 
    sqlSessionFactoryRef = "primarySqlSessionFactory"
)
public class PrimaryConfig {}

@MapperScan(
    basePackages = "com.example.secondary.mapper",
    sqlSessionFactoryRef = "secondarySqlSessionFactory"
)
public class SecondaryConfig {}
```
###### 5. MyBatis 与 Spring 事务管理如何配合？
MyBatis与Spring的事务整合通过**SqlSessionTemplate**和**Spring的事务同步机制**实现。
**事务管理器配置**
```xml
<bean id="transactionManager" 
      class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```
**声明式事务应用**
```java
@Service
public class UserService {
    @Transactional
    public void createUser(User user) {
        userMapper.insert(user); // 同一事务内操作
        logMapper.insertLog(user.getId(), "CREATE");
    }
}
```
**源码级事务同步机制**
1. **事务资源绑定**：当Spring事务开始时，TransactionSynchronizationManager将SqlSessionHolder绑定到当前线程
2. **连接复用**：事务内所有数据库操作通过SqlSessionTemplate获取当前事务关联的SqlSession
3. **事务提交/回滚**：事务结束时，Spring通过DataSourceTransactionManager提交或回滚数据库连接
**SqlSessionTemplate的拦截器机制**
```java
// 简化的SqlSessionInterceptor逻辑
public Object invoke(Method method, Object[] args) throws Throwable {
    // 从事务同步管理器获取当前SqlSession
    SqlSession sqlSession = getSqlSession(transactionManager);
    try {
        return method.invoke(sqlSession, args);
    } finally {
        // 非事务环境下自动关闭连接
        closeSqlSessionIfNecessary(sqlSession);
    }
}
```
###### 6. MyBatis 在 Spring 中如何实现依赖注入？
MyBatis在Spring中的依赖注入主要通过**Mapper接口的自动代理和Bean注册**实现。
**注入流程**
1. **Bean定义注册**：MapperScannerConfigurer扫描Mapper接口，生成MapperFactoryBean的BeanDefinition
2. **代理对象创建**：Spring容器初始化时，MapperFactoryBean通过getObject()返回MapperProxy代理对象
3. **依赖注入**：通过@Autowired等机制将代理对象注入到Service层
**Service层注入示例**
```java
@Service
public class UserService {
    // 直接注入Mapper接口
    @Autowired
    private UserMapper userMapper;
    
    public User getUserById(Long id) {
        return userMapper.selectById(id);
    }
}
```
**源码级注入原理**
- **MapperFactoryBean核心方法**：
```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport {
    @Override
    public T getObject() throws Exception {
        // 通过SqlSessionTemplate获取Mapper动态代理
        return getSqlSession().getMapper(this.mapperInterface);
    }
}
```
- **动态代理机制**：MyBatis通过MapperProxy实现InvocationHandler，将接口方法调用转换为SqlSession的数据库操作
**多数据源下的依赖注入**
```java
@Service
public class MultiDataSourceService {
    @Autowired
    @Qualifier("primaryUserMapper")
    private UserMapper primaryUserMapper;
    
    @Autowired
    @Qualifier("secondaryUserMapper") 
    private UserMapper secondaryUserMapper;
}
```
这种设计使得开发者能够以面向接口的方式操作数据库，同时享受Spring依赖注入带来的便利性，实现了数据访问层与业务逻辑层的完全解耦。