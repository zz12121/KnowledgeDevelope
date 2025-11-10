### 一、Spring 核心概念

###### 1. 什么是 Spring 框架？它的主要特点是什么？
###### 2. 什么是控制反转（IoC）？
###### 3. 什么是依赖注入（DI）？
###### 4. Spring 中的 IoC 容器有哪些类型？
###### 5. BeanFactory 和 ApplicationContext 的区别是什么？
###### 6. Spring 框架的核心模块有哪些？
###### 7. Spring 的优缺点是什么？

### 二、Spring Bean 管理

###### 1. 什么是 Spring Bean？
###### 2. Spring 中 Bean 的作用域有哪些？
###### 3. Spring Bean 的生命周期是怎样的？
###### 4. 如何在 Spring 中配置 Bean？
###### 5. 什么是 Bean 的装配？有哪些装配方式？
###### 6. 什么是自动装配？自动装配有哪些方式？
###### 7. @Autowired 和 @Resource 的区别是什么？
###### 8. @Component、@Service、@Controller、@Repository 的区别是什么？
###### 9. 如何解决循环依赖问题？
###### 10. Spring 如何处理线程安全问题？
###### 11. 什么是懒加载（Lazy Initialization）？
###### 12. 如何在 Spring 中注入集合类型？

### 三、Spring AOP（面向切面编程）

###### 1. 什么是 AOP？
###### 2. AOP 的核心概念有哪些（切面、连接点、切入点、通知、织入等）？
###### 3. Spring AOP 和 AspectJ 的区别是什么？
###### 4. Spring AOP 支持哪些类型的通知（Advice）？
###### 5. 如何实现一个自定义切面？
###### 6. 什么是代理模式？JDK 动态代理和 CGLIB 代理的区别是什么？
###### 7. Spring AOP 的底层实现原理是什么？
###### 8. 切入点表达式如何编写？
###### 9. AOP 有哪些应用场景？

### 四、Spring 事务管理

###### 1. 什么是事务？事务的 ACID 特性是什么？
###### 2. Spring 如何管理事务？
###### 3. 编程式事务和声明式事务的区别是什么？
###### 4. @Transactional 注解的使用方法和注意事项是什么？
###### 5. Spring 事务的传播行为有哪些？
###### 6. Spring 事务的隔离级别有哪些？
###### 7. 什么情况下 @Transactional 会失效？
###### 8. 如何实现分布式事务？
###### 9. 什么是事务的回滚规则？

### 五、Spring 配置与注解

###### 1. Spring 有哪些配置方式？
###### 2. XML 配置和 Java 配置的区别是什么？
###### 3. @Configuration 注解的作用是什么？
###### 4. @Bean 注解的作用是什么？
###### 5. @ComponentScan 注解的作用是什么？
###### 6. @Import 注解的作用是什么？
###### 7. @Conditional 系列注解的作用是什么？
###### 8. @Profile 注解如何使用？
###### 9. @Value 注解如何使用？
###### 10. @PropertySource 注解的作用是什么？
###### 11. @Qualifier 注解的作用是什么？
###### 12. @Primary 注解的作用是什么？
###### 13. @Lazy 注解的作用是什么？
###### 14. @Scope 注解的作用是什么？
###### 15. @PostConstruct 和 @PreDestroy 注解的作用是什么？

### 六、Spring 容器与上下文

###### 1. 什么是 ApplicationContext？
###### 2. ApplicationContext 有哪些常见的实现类？
###### 3. Spring 容器的启动流程是怎样的？
###### 4. 什么是 BeanDefinition？
###### 5. 什么是 BeanFactoryPostProcessor？
###### 6. 什么是 BeanPostProcessor？
###### 7. FactoryBean 和 BeanFactory 的区别是什么？
###### 8. 如何在 Spring 容器中获取 Bean？
###### 9. ApplicationContextAware 接口的作用是什么？

### 七、Spring 数据访问

###### 1. Spring 如何集成 JDBC？
###### 2. 什么是 JdbcTemplate？
###### 3. Spring 如何集成 MyBatis？
###### 4. Spring 如何集成 Hibernate？
###### 5. 什么是 Spring Data JPA？
###### 6. @Repository 注解的特殊作用是什么？
###### 7. DataSource 如何配置？
###### 8. 如何处理数据库异常？

### 八、Spring 事件机制

###### 1. 什么是 Spring 事件机制？
###### 2. 如何自定义 Spring 事件？
###### 3. 如何发布 Spring 事件？
###### 4. @EventListener 注解如何使用？
###### 5. Spring 事件机制是同步还是异步？
###### 6. 如何实现异步事件监听？

### 九、Spring 缓存

###### 1. Spring 如何实现缓存？
###### 2. @Cacheable 注解如何使用？
###### 3. @CacheEvict 注解如何使用？
###### 4. @CachePut 注解如何使用？
###### 5. @Caching 注解如何使用？
###### 6. 如何配置缓存管理器？
###### 7. Spring 支持哪些缓存实现？
###### 8. 缓存的过期策略如何设置？

### 十、Spring 任务调度

###### 1. Spring 如何实现定时任务？
###### 2. @Scheduled 注解如何使用？
###### 3. @EnableScheduling 注解的作用是什么？
###### 4. Cron 表达式如何编写？
###### 5. 如何实现异步任务？
###### 6. @Async 注解如何使用？
###### 7. @EnableAsync 注解的作用是什么？
###### 8. 如何配置线程池？

### 十一、Spring 设计模式

###### 1. Spring 中使用了哪些设计模式？
###### 2. 工厂模式在 Spring 中的应用？
###### 3. 单例模式在 Spring 中的应用？
###### 4. 代理模式在 Spring 中的应用？
###### 5. 模板方法模式在 Spring 中的应用？
###### 6. 观察者模式在 Spring 中的应用？
###### 7. 适配器模式在 Spring 中的应用？
###### 8. 装饰器模式在 Spring 中的应用？

### 十二、Spring 整合与扩展

###### 1. 如何自定义 BeanPostProcessor？
###### 2. 如何自定义 BeanFactoryPostProcessor？
###### 3. 如何实现自定义注解？
###### 4. 如何扩展 Spring 容器？
###### 5. 什么是 SPI 机制？Spring 如何使用 SPI？
###### 6. 如何在 Spring 中集成第三方框架？

### 十三、Spring 性能优化

###### 1. 如何优化 Spring 应用的启动速度？
###### 2. 如何减少 Bean 的创建开销？
###### 3. 懒加载如何提升性能？
###### 4. 如何优化 AOP 性能？
###### 5. 如何优化数据库访问性能？
###### 6. 连接池如何配置？

### 十四、Spring 其他高级特性

###### 1. 什么是 Spring EL（表达式语言）？
###### 2. 什么是 Spring Validation？
###### 3. 如何实现参数校验？
###### 4. 什么是国际化（i18n）？Spring 如何支持国际化？
###### 5. 什么是资源加载？Spring 如何加载资源？
###### 6. Environment 接口的作用是什么？
###### 7. PropertyResolver 接口的作用是什么？
###### 8. 什么是 Type Conversion？
###### 9. 什么是 Formatter？