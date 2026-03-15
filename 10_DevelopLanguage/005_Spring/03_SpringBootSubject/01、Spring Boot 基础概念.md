###### 1. 什么是 SpringBoot？

Spring Boot 是基于 Spring Framework 的**上层封装和增强**，核心设计理念是"**约定优于配置**"。它本身不是 Spring 的替代品，底层仍然依赖 Spring 的 IoC、AOP 等核心功能，只是通过自动配置机制、Starter 依赖、内嵌容器等手段，大幅简化了 Spring 应用的搭建和开发过程。

简单说，以前用 Spring 要写一堆 XML 或 Java 配置，Spring Boot 把这些配置都帮你约定好了，开箱即用，你只需关注业务逻辑。

📖 [[26_SpringBootKnowledge/01_核心机制与启动原理/01、SpringBoot核心机制与启动原理]]

---

###### 2. Spring Boot 的核心特性有哪些？

主要有五大核心特性：

**1. 自动配置**：根据类路径中的依赖，结合 `@Conditional` 条件注解自动配置 Bean。比如引入了 Redis 依赖，就自动配置 `RedisTemplate`，不用手动写配置类。

**2. Starter 依赖管理**：每个 Starter 聚合了某个功能所需的所有依赖，并保证版本兼容，解决了传统 Maven 项目的依赖冲突问题：

```java
@Configuration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(DataSource.class)
public class DataSourceAutoConfiguration {
    @Bean
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

**3. 内嵌 Servlet 容器**：支持 Tomcat（默认）、Jetty、Undertow，打包成可执行 JAR，直接 `java -jar` 运行，无需外部服务器。

**4. 外部化配置**：通过 `application.properties` / `application.yml` 集中管理配置，配合 Profile 实现多环境隔离，外部配置可以覆盖打包内的配置。

**5. 生产就绪监控（Actuator）**：引入 `spring-boot-starter-actuator` 就能使用健康检查、指标收集、配置查看等开箱即用的运维端点。

📖 [[26_SpringBootKnowledge/01_核心机制与启动原理/01、SpringBoot核心机制与启动原理]]

---

###### 3. Spring Boot 与 Spring Framework 的区别是什么？

两者是增强关系，不是替代关系。Spring Framework 是底层基础，Spring Boot 是基于它的封装层。

核心区别在于：
- **配置方式**：Spring Framework 需要大量 XML 或 Java 显式配置；Spring Boot 约定优于配置，大量配置自动生效
- **依赖管理**：Spring Framework 需要手动管理依赖版本，容易冲突；Spring Boot 通过 Starter POMs 统一管理，版本兼容性有保障
- **部署方式**：Spring Framework 通常打 WAR 包部署到外部服务器；Spring Boot 内嵌容器，可执行 JAR 独立运行
- **入门门槛**：Spring Framework 需要深入理解 Spring 架构才能用好；Spring Boot 快速上手，专注业务
- **监控功能**：Spring Framework 需要手动集成；Spring Boot 内置 Actuator，开箱即用

Spring Boot 建立在 Spring Framework 之上，所有的 IoC、AOP、事务管理等核心能力都来自 Spring Framework，Spring Boot 只是让这些能力更易用。

📖 [[26_SpringBootKnowledge/01_核心机制与启动原理/01、SpringBoot核心机制与启动原理]]

---

###### 4. Spring Boot 的优缺点分别是什么？

**优点：**
- 开发效率高，减少大量样板配置，项目快速启动
- 微服务友好，内嵌容器 + 独立 JAR 天然契合容器化部署
- 生态整合完善，与 Spring Cloud、Spring Data、Spring Security 无缝集成
- Actuator 提供完整的生产监控能力
- 测试支持完善，`@SpringBootTest` 等注解简化集成测试

**缺点与应对：**
- **版本迭代快**，升级时可能有兼容性问题 → 建立严格的版本管理规范
- **自动配置增加排查难度**，出问题时不知道哪个配置生效了 → 用 `debug=true` 开启自动配置报告，看哪些配置生效/未生效
- **内存占用较高**，自动加载了很多不需要的 Bean → 通过 `spring.autoconfigure.exclude` 排除不需要的自动配置

📖 [[26_SpringBootKnowledge/01_核心机制与启动原理/01、SpringBoot核心机制与启动原理]]

---

###### 5. 为什么要使用 Spring Boot？

核心原因是它解决了传统 Spring 开发的三大痛点：**配置复杂**、**依赖管理困难**、**部署繁琐**。

具体来说：对于团队，降低了学习成本，新成员能快速融入；对于架构，内嵌容器和独立运行特性使其天然适合微服务，和 Spring Cloud 配合可以快速构建分布式系统；对于运维，Actuator 提供健康检查、指标收集，符合云原生的可观测性要求，就绪性探针和存活性探针方便 Kubernetes 部署；对于开发体验，热部署（DevTools）、Profile 多环境切换都是开箱即用的。

📖 [[26_SpringBootKnowledge/01_核心机制与启动原理/01、SpringBoot核心机制与启动原理]]

---

###### 6. Spring Boot 的设计理念是什么？

Spring Boot 有四个核心设计理念：

**1. 约定优于配置**：预设合理的默认值，只有需要偏离默认行为时才需要显式配置。比如静态资源默认从 `/static`、`/public` 目录加载，配置文件默认叫 `application.properties`，这些约定让你不写任何配置就能跑起来一个 Web 应用。

**2. 自动配置**：基于条件注解的智能 Bean 注册，框架会根据你引入的依赖自动决定需要配置什么，不需要的不配置，需要的开箱即用。

**3. Starter 设计模式**：把功能相关的依赖聚合成一个整体，包含所需依赖库、版本和自动配置类，一个 Starter 搞定一类功能的所有依赖问题。

**4. 零代码生成与透明性**：Spring Boot 坚持不生成代码，所有行为都通过标准 Java 代码实现，你可以通过查看自动配置类的源码完全理解框架行为，方便深度定制和问题排查。

📖 [[26_SpringBootKnowledge/01_核心机制与启动原理/01、SpringBoot核心机制与启动原理]]

---

###### 7. Spring Boot Starter 的作用是什么？

Starter 的核心作用是**简化依赖管理**，解决版本冲突问题。引入一个 Starter 就相当于引入了对应功能模块所需的全部依赖，且版本之间互相兼容。

比如 `spring-boot-starter-web` 自动包含了 Spring MVC、Tomcat、Jackson、Bean Validation 这些组件，并保证它们的版本互相兼容，不用自己手动管理。

每个 Starter 还会关联对应的自动配置类，添加依赖的同时触发自动配置，实现真正的开箱即用。

在企业级项目里，可以参照这个机制创建自定义 Starter，把可复用的业务组件（比如公司统一的鉴权客户端、日志组件）封装起来，其他团队引入一个依赖就能用上：

```java
@Configuration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties props) {
        return new DefaultMyService(props);
    }
}
```

📖 [[26_SpringBootKnowledge/01_核心机制与启动原理/01、SpringBoot核心机制与启动原理]]

---

###### 8. Spring Boot 支持哪些嵌入式服务器？

Spring Boot 支持三种嵌入式服务器：

**Tomcat（默认）**：最稳定，社区最成熟，适合大多数 Web 应用。引入 `spring-boot-starter-web` 默认就是 Tomcat。

**Jetty**：高并发和长连接场景（比如 WebSocket）性能更好。切换方式是排除 Tomcat，引入 `spring-boot-starter-jetty`。

**Undertow**：基于 NIO，内存占用低，支持 HTTP/2 和 WebSocket，适合资源紧张或需要高性能的场景。

切换服务器很简单，只需要在 `pom.xml` 里排除默认的 Tomcat 然后引入对应的 Starter 就行了。

选型建议：绝大多数场景用默认的 Tomcat 就够了；高并发长连接场景考虑 Jetty；对内存有要求或需要 HTTP/2 支持考虑 Undertow。

📖 [[26_SpringBootKnowledge/01_核心机制与启动原理/01、SpringBoot核心机制与启动原理]]
