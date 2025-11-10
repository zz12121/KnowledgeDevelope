### 一、JDBC 基础类

###### 1. JDBC 有几个步骤？
###### 2. JDBC 操作数据库的核心对象有哪些？各自的作用是什么？
###### 3. JDBC 中 Statement 和 PreparedStatement 的区别是什么？
###### 4. JDBC 如何处理事务？
###### 5. JDBC 连接池的作用是什么？常见的连接池有哪些？

### 二、MyBatis 基础类

###### 1. 什么是 MyBatis？它解决了什么问题？
###### 2. MyBatis 的核心组件有哪些？
###### 3. MyBatis 与 Hibernate 的区别是什么？
###### 4. MyBatis 的工作流程是怎样的？
###### 5. MyBatis 中的 SqlSession 是什么？它是线程安全的吗？
###### 6. MyBatis 中的 SqlSessionFactory 是什么？如何创建？
###### 7. MyBatis 的配置文件（mybatis-config.xml）中可以配置哪些内容？

### 三、映射文件与 SQL 类

###### 1. MyBatis 映射文件中有哪些常用标签？
###### 2. MyBatis 中 #{} 和 ${} 的区别是什么？
###### 3. MyBatis 如何实现动态 SQL？
###### 4. MyBatis 中 if、choose、foreach 标签的使用场景是什么？
###### 5. MyBatis 如何处理 SQL 注入问题？
###### 6. MyBatis 中 resultType 和 resultMap 的区别是什么？
###### 7. MyBatis 如何实现结果映射？
###### 8. MyBatis 如何处理字段名与属性名不一致的情况？
###### 9. MyBatis 中如何实现一对一、一对多、多对多关联查询？
###### 10. MyBatis 的延迟加载是什么？如何配置？
###### 11. MyBatis 的 association 和 collection 标签有什么区别？
	
### 四、注解与接口类

###### 1. MyBatis 支持哪些常用注解？
###### 2. MyBatis 中 @Param 注解的作用是什么？
###### 3. MyBatis 的 Mapper 接口是如何工作的？
###### 4. MyBatis 如何实现 Mapper 接口与 XML 的绑定？
###### 5. MyBatis 中使用注解和 XML 配置的优缺点是什么？
###### 6. MyBatis 的 @SelectProvider、@InsertProvider 等注解的作用是什么？

### 五、缓存机制类

###### 1. MyBatis 的缓存机制是怎样的？
###### 2. MyBatis 一级缓存和二级缓存的区别是什么？
###### 3. MyBatis 一级缓存的生命周期是什么？
###### 4. MyBatis 二级缓存如何配置？
###### 5. MyBatis 缓存的失效场景有哪些？
###### 6. MyBatis 如何集成第三方缓存（如 Redis、Ehcache）？
###### 7. MyBatis 缓存的命中率如何优化？

### 六、插件与扩展类

###### 1. MyBatis 的插件机制是什么？如何自定义插件？
###### 2. MyBatis 插件的拦截点有哪些？
###### 3. MyBatis 的分页插件（PageHelper）是如何工作的？
###### 4. MyBatis 如何实现物理分页和逻辑分页？
###### 5. MyBatis 的拦截器（Interceptor）有什么作用？
###### 6. MyBatis 如何实现 SQL 性能监控？

### 七、事务管理类

###### 1. MyBatis 如何管理事务？
###### 2. MyBatis 中事务的隔离级别如何设置？
###### 3. MyBatis 与 Spring 集成后，事务是如何管理的？
###### 4. MyBatis 的事务传播行为有哪些？
###### 5. MyBatis 如何处理事务回滚？

### 八、数据源与连接池类

###### 1. MyBatis 自带的连接池有什么特点？
###### 2. MyBatis 如何配置第三方连接池（如 HikariCP、Druid）？
###### 3. MyBatis 如何实现多数据源配置？
###### 4. MyBatis 多数据源下如何切换数据源？
###### 5. MyBatis 如何实现读写分离？

### 九、批处理与性能优化类

###### 1. MyBatis 如何实现批量插入？
###### 2. MyBatis 的批处理（Batch）模式是什么？如何使用？
###### 3. MyBatis 如何优化大数据量查询？
###### 4. MyBatis 的 N+1 查询问题是什么？如何解决？
###### 5. MyBatis 如何避免全表扫描？
###### 6. MyBatis 的执行器（Executor）有哪几种类型？各有什么特点？
###### 7. MyBatis 如何进行 SQL 调优？

### 十、Spring 集成类

###### 1. MyBatis 如何与 Spring 集成？
###### 2. MyBatis-Spring 的核心组件有哪些？
###### 3. Spring Boot 如何整合 MyBatis？
###### 4. @MapperScan 注解的作用是什么？
###### 5. MyBatis 与 Spring 事务管理如何配合？
###### 6. MyBatis 在 Spring 中如何实现依赖注入？

### 十一、MyBatis-Plus 类

###### 1. 什么是 MyBatis-Plus？它有什么作用？
###### 2. MyBatis-Plus 与 MyBatis 有哪些区别？
###### 3. MyBatis-Plus 的通用 Mapper 有哪些方法？
###### 4. MyBatis-Plus 的条件构造器（Wrapper）如何使用？
###### 5. MyBatis-Plus 的分页插件如何配置和使用？
###### 6. MyBatis-Plus 的代码生成器如何使用？
###### 7. MyBatis-Plus 的逻辑删除功能是什么？如何实现？
###### 8. MyBatis-Plus 的自动填充功能如何使用？
###### 9. MyBatis-Plus 的主键生成策略有哪些？
###### 10. MyBatis-Plus 如何实现乐观锁？
###### 11. MyBatis-Plus 的性能分析插件如何使用？
###### 12. MyBatis-Plus 如何实现多租户功能？

### 十二、高级特性类

###### 1. MyBatis 如何处理枚举类型？
###### 2. MyBatis 如何自定义 TypeHandler？
###### 3. MyBatis 的 TypeHandler 有什么作用？
###### 4. MyBatis 如何实现存储过程调用？
###### 5. MyBatis 如何处理 BLOB 和 CLOB 类型？
###### 6. MyBatis 如何实现动态表名？
###### 7. MyBatis 的鉴别器（discriminator）有什么作用？
###### 8. MyBatis 如何实现结果集的嵌套映射？

### 十三、调试与问题排查类

###### 1. MyBatis 如何开启 SQL 日志？
###### 2. MyBatis 如何查看执行的 SQL 语句？
###### 3. MyBatis 常见的异常有哪些？如何解决？
###### 4. MyBatis 的 Mapper 找不到的问题如何排查？
###### 5. MyBatis 如何调试 SQL 参数绑定？

### 十四、最佳实践类

###### 1. MyBatis 的最佳实践有哪些？
###### 2. MyBatis 项目中如何组织 Mapper 文件？
###### 3. MyBatis 的命名规范有哪些建议？
###### 4. MyBatis 在微服务架构中如何使用？
###### 5. MyBatis 如何与分库分表中间件（如 Sharding-JDBC）集成？
###### 6. MyBatis 的安全性注意事项有哪些？

### 十五、对比与选型类

###### 1. MyBatis 与 JPA 的区别是什么？如何选择？
###### 2. MyBatis 与 Spring Data JDBC 的区别是什么？
###### 3. 什么场景下适合使用 MyBatis？
###### 4. MyBatis 的优缺点是什么？