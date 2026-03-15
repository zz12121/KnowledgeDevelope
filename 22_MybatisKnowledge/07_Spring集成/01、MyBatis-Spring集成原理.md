# MyBatis-Spring 集成原理

## 1. 集成概述

MyBatis 独立使用时，`SqlSessionFactory` 需要手动创建，`SqlSession` 需要手动管理生命周期，事务也需要单独处理。引入 **MyBatis-Spring** 桥接模块后，这些工作全部交由 Spring 容器托管：

- `SqlSessionFactory` → 由 `SqlSessionFactoryBean` 在容器启动时自动创建
- `SqlSession` → 由 `SqlSessionTemplate` 代理，与 Spring 事务自动绑定
- Mapper 接口 → 由 `MapperScannerConfigurer` 扫描并注册为 Spring Bean

**依赖配置**：

```xml
<!-- 传统 Spring 项目 -->
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis-spring</artifactId>
    <version>2.0.6</version>
</dependency>

<!-- Spring Boot 项目（推荐，自动配置） -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>
```

---

## 2. 核心组件详解

### SqlSessionFactoryBean

`SqlSessionFactoryBean` 实现了 `InitializingBean` 接口，在 Spring 容器完成属性注入后，`afterPropertiesSet()` 方法被自动调用，内部执行 `buildSqlSessionFactory()` 完成初始化。

职责：解析 `mybatis-config.xml`、扫描 Mapper XML 文件、构建 `Configuration` 对象、最终创建 `SqlSessionFactory` 单例。

```xml
<!-- XML 配置示例 -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mapperLocations" value="classpath:mapper/*.xml"/>
</bean>
```

### SqlSessionTemplate

`SqlSessionTemplate` 是 **MyBatis-Spring 整合的核心**，它实现了 `SqlSession` 接口，作为线程安全的代理对象供业务代码使用。

**实现原理**：内部通过 JDK 动态代理创建 `sqlSessionProxy`，其 `InvocationHandler`（即 `SqlSessionInterceptor`）在每次方法调用时执行以下逻辑：

```java
// 简化的 SqlSessionInterceptor 逻辑
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // 1. 从 TransactionSynchronizationManager 获取当前事务关联的 SqlSession
    //    如果没有事务，则创建新的 SqlSession
    SqlSession sqlSession = getSqlSession(SqlSessionTemplate.this.sqlSessionFactory,
                                          SqlSessionTemplate.this.executorType,
                                          SqlSessionTemplate.this.exceptionTranslator);
    try {
        return method.invoke(sqlSession, args);
    } finally {
        // 2. 非事务环境下自动关闭，事务环境下保持连接直到事务结束
        closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
    }
}
```

这个设计解决了两个核心问题：
- **线程安全**：每个线程通过 `TransactionSynchronizationManager`（本质是 ThreadLocal）绑定自己的 SqlSession，互不干扰
- **事务感知**：同一个事务中的多次数据库操作，复用相同的 SqlSession（即同一个 Connection）

### MapperFactoryBean

单个 Mapper 接口的 Spring 工厂 Bean，`getObject()` 方法通过 `getSqlSession().getMapper(mapperInterface)` 返回 MapperProxy 动态代理对象。

继承了 `SqlSessionDaoSupport`，后者注入了 `SqlSessionTemplate`。

### MapperScannerConfigurer

**批量注册 Mapper 的核心组件**，实现了 `BeanDefinitionRegistryPostProcessor`。

工作机制分三步：
1. **扫描**：通过 `ClassPathMapperScanner` 扫描指定包路径，找到所有接口
2. **修改 BeanDefinition**：将扫描到的接口的 `beanClass` 替换为 `MapperFactoryBean`，并注入原接口类型作为构造参数
3. **代理生成**：Spring 容器初始化时，调用 `MapperFactoryBean.getObject()` 生成 Mapper 动态代理对象

```xml
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="com.example.mapper"/>
</bean>
```

---

## 3. Spring Boot 自动配置

Spring Boot 通过 `MyBatisAutoConfiguration` 实现零配置集成：

- 自动创建 `SqlSessionFactory` 和 `SqlSessionTemplate` Bean
- 通过 `MybatisProperties` 绑定 `mybatis.*` 配置参数
- `@MapperScan` 触发 `MapperScannerRegistrar`（实现了 `ImportBeanDefinitionRegistrar`），批量注册 Mapper

```yaml
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.example.entity
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

```java
@SpringBootApplication
@MapperScan("com.example.mapper")  // 批量扫描，替代每个接口的 @Mapper 注解
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## 4. 依赖注入全流程

```
Spring 容器启动
    ↓
MapperScannerConfigurer 扫描 com.example.mapper 包
    ↓
将每个 Mapper 接口的 BeanDefinition 的 beanClass 改为 MapperFactoryBean
    ↓
容器初始化 MapperFactoryBean，调用 getObject()
    ↓
getSqlSession().getMapper(UserMapper.class)
    ↓
返回 MapperProxy 代理对象，注册为 Spring Bean
    ↓
Service 层 @Autowired 注入 UserMapper 时，得到 MapperProxy
    ↓
调用 userMapper.selectById(1L)
    ↓
MapperProxy → SqlSessionTemplate → SqlSession → Executor → DB
```

---

## 5. 事务整合机制

详见 [[01、事务管理与传播行为#3. Spring 事务整合机制|Spring 事务整合机制]]。

核心要点：`DataSourceTransactionManager` 开启事务时，通过 `TransactionSynchronizationManager` 将 `Connection` 绑定到 ThreadLocal。`SqlSessionTemplate` 通过 `getSqlSession()` 方法从同一个 ThreadLocal 取出 `SqlSession`，保证事务内多次操作使用同一个 Connection。

---

## 6. @MapperScan 详解

```java
@Configuration
@MapperScan(
    basePackages = "com.example.mapper",
    sqlSessionFactoryRef = "sqlSessionFactory",  // 多数据源时指定使用哪个工厂
    lazyInitialization = "true"                  // 延迟初始化，加快启动速度
)
public class MyBatisConfig {}
```

`lazyInitialization = "true"` 在 Spring Boot 项目中可以显著减少启动时间，Mapper 代理对象在第一次被注入时才真正初始化，而不是容器启动时全量初始化。

---

## 相关面试题

- [[003_ORM/01_MyBatisSubject/10、Spring 集成|📖 10、Spring 集成]]
