###### 1. MyBatis 如何与 Spring 集成？
[[22_MybatisKnowledge/07_Spring集成/01、MyBatis-Spring集成原理|📖]] MyBatis 与 Spring 的集成通过 **MyBatis-Spring** 桥接模块实现，核心是将 MyBatis 的组件生命周期交由 Spring 容器管理。集成后的结构是：Spring 管理 DataSource → `SqlSessionFactoryBean` 根据 DataSource 创建 `SqlSessionFactory` → `SqlSessionTemplate` 代理 SqlSession → `MapperScannerConfigurer` 扫描 Mapper 接口并注册为 Bean。

XML 配置示例：

```xml
<!-- 创建 SqlSessionFactory -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mapperLocations" value="classpath:mapper/*.xml"/>
</bean>

<!-- 批量扫描 Mapper 接口 -->
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="com.example.mapper"/>
</bean>
```

`SqlSessionFactoryBean` 实现了 `InitializingBean`，在 Spring 注入属性完成后自动调用 `afterPropertiesSet()` 初始化 `SqlSessionFactory`。`MapperScannerConfigurer` 实现了 `BeanDefinitionRegistryPostProcessor`，在容器启动时扫描 Mapper 接口，将其 BeanDefinition 的 beanClass 替换为 `MapperFactoryBean`，Spring 通过 `MapperFactoryBean.getObject()` 生成 Mapper 动态代理对象。

###### 2. MyBatis-Spring 的核心组件有哪些？
[[22_MybatisKnowledge/07_Spring集成/01、MyBatis-Spring集成原理#2. 核心组件详解|📖]] **SqlSessionFactoryBean**：SqlSessionFactory 的工厂 Bean，在 `afterPropertiesSet()` 中调用 `buildSqlSessionFactory()` 完成初始化，解析 MyBatis 配置文件和映射文件，构建 `Configuration` 对象。

**SqlSessionTemplate**：这是整合的核心，实现了 SqlSession 接口，是线程安全的代理。内部通过 JDK 动态代理创建 `sqlSessionProxy`，其 InvocationHandler（`SqlSessionInterceptor`）在每次方法调用时从 `TransactionSynchronizationManager` 获取当前事务关联的 SqlSession，非事务环境下自动关闭连接，避免资源泄漏。

**MapperFactoryBean**：单个 Mapper 接口的工厂 Bean，`getObject()` 方法返回通过 `getSqlSession().getMapper(mapperInterface)` 生成的动态代理对象。

**MapperScannerConfigurer**：批量扫描和注册 Mapper 接口，通过 `ClassPathMapperScanner` 扫描指定包，修改 BeanDefinition 的 beanClass 为 `MapperFactoryBean` 并设置接口类型作为构造参数。

###### 3. Spring Boot 如何整合 MyBatis？
[[22_MybatisKnowledge/07_Spring集成/01、MyBatis-Spring集成原理#3. Spring Boot 自动配置|📖]] Spring Boot 通过 `mybatis-spring-boot-starter` 实现零配置集成，`MyBatisAutoConfiguration` 自动创建 `SqlSessionFactory` 和 `SqlSessionTemplate` Bean，`MybatisProperties` 绑定 `mybatis.*` 配置参数。

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>
```

```yaml
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.example.entity
  configuration:
    map-underscore-to-camel-case: true
```

```java
@SpringBootApplication
@MapperScan("com.example.mapper")  // 批量扫描，无需每个接口加 @Mapper
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

`@MapperScan` 触发 `MapperScannerRegistrar`（`ImportBeanDefinitionRegistrar` 实现），在容器启动时批量注册 Mapper。`lazyInitialization = "true"` 可以延迟 Mapper 代理对象的初始化，加快应用启动速度。

###### 4. @MapperScan 注解的作用是什么？
[[22_MybatisKnowledge/07_Spring集成/01、MyBatis-Spring集成原理#6. @MapperScan 详解|📖]] `@MapperScan` 是 Spring 整合 MyBatis 的核心注解，用于**批量扫描并注册 Mapper 接口为 Spring Bean**，替代手动在每个接口上添加 `@Mapper` 注解。

工作机制分三步：`MapperScannerRegistrar` 创建 `ClassPathMapperScanner` → 扫描指定包路径，过滤出接口 → 修改 BeanDefinition 的 beanClass 为 `MapperFactoryBean`，设置接口类型为构造参数 → Spring 初始化时调用 `MapperFactoryBean.getObject()` 生成代理对象。

多数据源时，配合 `sqlSessionFactoryRef` 将不同包的 Mapper 绑定到不同的 SqlSessionFactory：

```java
@MapperScan(basePackages = "com.example.primary.mapper",
            sqlSessionFactoryRef = "primarySqlSessionFactory")
public class PrimaryConfig {}

@MapperScan(basePackages = "com.example.secondary.mapper",
            sqlSessionFactoryRef = "secondarySqlSessionFactory")
public class SecondaryConfig {}
```

###### 5. MyBatis 与 Spring 事务管理如何配合？
[[22_MybatisKnowledge/07_Spring集成/01、MyBatis-Spring集成原理#5. 事务整合机制|📖]] MyBatis 与 Spring 事务的整合通过 `SqlSessionTemplate` 和 `TransactionSynchronizationManager` 实现无缝配合。

当 Spring 事务开始时，`DataSourceTransactionManager` 从 DataSource 获取 Connection，通过 `TransactionSynchronizationManager` 将 `ConnectionHolder`（包含这个 Connection）绑定到当前线程的 ThreadLocal。

当 `SqlSessionTemplate` 执行 SQL 操作时，内部的 `SqlSessionInterceptor` 调用 `getSqlSession()` 方法，从 `TransactionSynchronizationManager` 取出当前线程的 `ConnectionHolder`，用里面的 Connection 创建 SqlSession，保证事务内所有操作使用同一个 Connection。

事务方法结束时，`TransactionInterceptor` 根据是否有异常决定提交还是回滚，并解绑 ThreadLocal 中的 Connection。

这个机制保证了：同一个 `@Transactional` 方法内，不管调用多少个不同的 Mapper，都在同一个 JDBC 事务中，要么全部成功，要么全部回滚。

###### 6. MyBatis 在 Spring 中如何实现依赖注入？
[[22_MybatisKnowledge/07_Spring集成/01、MyBatis-Spring集成原理#4. 依赖注入全流程|📖]] MyBatis Mapper 的依赖注入依赖于 Spring 的 Bean 机制，整个流程是：

`MapperScannerConfigurer` 在容器启动时把所有 Mapper 接口注册为 `MapperFactoryBean` 类型的 BeanDefinition → Spring 初始化时调用 `MapperFactoryBean.getObject()` 返回 `MapperProxy` 代理对象 → 这个代理对象成为 Spring Bean。

当 Service 层声明 `@Autowired UserMapper userMapper` 时，Spring 注入的就是这个 `MapperProxy` 代理对象。调用 `userMapper.selectById(1L)` 时，代理拦截后转成 SqlSession 的数据库操作执行。

整个过程对开发者完全透明，像使用普通 Bean 一样使用 Mapper，数据访问层与业务逻辑层完全解耦。
