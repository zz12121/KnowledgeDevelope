# SpringBoot 核心机制与启动原理

## SpringBoot 是什么

Spring Boot 是基于 Spring Framework 的**上层封装**，核心设计理念是"**约定优于配置**"。它不是 Spring 的替代品，而是通过自动配置、Starter 依赖、内嵌容器等机制，大幅降低了 Spring 应用的搭建和配置成本。

简单说：用 Spring 需要写很多配置，Spring Boot 把这些配置都帮你做了，你只需关注业务代码。

---

## 核心特性

**1. 自动配置（Auto-configuration）**

根据类路径中存在的依赖，结合 `@Conditional` 条件注解，自动注册对应的 Bean。比如类路径有 `DataSource`，就自动配置数据源；有 `spring-boot-starter-redis`，就自动配置 RedisTemplate。

**2. Starter 依赖管理**

每个 Starter 聚合了某个功能模块的所有相关依赖，并保证版本兼容。引入 `spring-boot-starter-web` 就自动包含 Spring MVC、Tomcat、Jackson 等，不用一个个手动管理版本。

**3. 内嵌 Servlet 容器**

支持 Tomcat（默认）、Jetty、Undertow，应用打包为可执行 JAR，直接 `java -jar` 运行，无需外部服务器。

**4. 外部化配置**

`application.properties` / `application.yml` 集中管理配置，配合 Profile 机制实现多环境隔离。配置加载遵循优先级规则，外部配置可以覆盖打包内的配置。

**5. 生产就绪监控（Actuator）**

通过 `spring-boot-starter-actuator` 开箱即用，提供健康检查（`/actuator/health`）、指标收集（`/actuator/metrics`）、配置查看（`/actuator/env`）等端点。

---

## 自动配置原理

自动配置的完整链路：

```
@SpringBootApplication
    ↓ 包含
@EnableAutoConfiguration
    ↓ 通过 @Import 导入
AutoConfigurationImportSelector
    ↓ 从 META-INF/spring.factories 或
      META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 读取
所有候选自动配置类（200+ 个）
    ↓ 经过 @Conditional 条件过滤
实际生效的自动配置类
    ↓
注册对应的 Bean
```

**核心类 `AutoConfigurationImportSelector`：**

```java
public class AutoConfigurationImportSelector implements DeferredImportSelector {
    public String[] selectImports(AnnotationMetadata metadata) {
        List<String> configurations = getCandidateConfigurations(metadata, attributes);
        configurations = removeDuplicates(configurations); // 去重
        configurations = filter(configurations, autoConfigurationMetadata); // 条件过滤
        return StringUtils.toStringArray(configurations);
    }
}
```

**@Conditional 系列注解（条件过滤的核心）：**

| 注解 | 生效条件 |
|------|----------|
| `@ConditionalOnClass` | 类路径存在指定类 |
| `@ConditionalOnMissingBean` | 容器中没有指定 Bean |
| `@ConditionalOnProperty` | 配置属性满足条件 |
| `@ConditionalOnWebApplication` | 是 Web 应用 |
| `@ConditionalOnBean` | 容器中存在指定 Bean |

---

## @SpringBootApplication 注解解析

这是个复合注解，展开来是三个注解：

```java
@SpringBootConfiguration  // 本质是 @Configuration，标记为配置类
@EnableAutoConfiguration  // 启用自动配置
@ComponentScan            // 开启组件扫描（扫描同级包及子包）
public @interface SpringBootApplication {}
```

实际开发中可以用 `@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})` 排除不需要的自动配置。

---

## SpringBoot 启动流程

`SpringApplication.run()` 方法是整个启动过程的入口，关键步骤：

1. **创建 SpringApplication 实例**：推断应用类型（SERVLET/REACTIVE/NONE），加载初始化器（`ApplicationContextInitializer`）和监听器（`ApplicationListener`）
2. **准备运行环境**：加载配置文件，处理 Profile，发布 `ApplicationEnvironmentPreparedEvent`
3. **打印 Banner**
4. **创建 ApplicationContext**：根据应用类型创建对应的上下文（Servlet → `AnnotationConfigServletWebServerApplicationContext`）
5. **刷新上下文（核心）**：调用 `AbstractApplicationContext.refresh()`，完成 Bean 定义加载、自动配置、所有单例 Bean 的实例化
6. **执行 Runner**：依次调用所有 `CommandLineRunner` 和 `ApplicationRunner`
7. **发布 `ApplicationReadyEvent`**，启动完成

`refresh()` 是最重的一步，里面包含了 BeanFactory 初始化、BeanPostProcessor 注册、非懒加载 Bean 的实例化等所有核心操作。

---

## 启动时执行代码的方式

**方式一：`CommandLineRunner`**（参数是原始字符串）

```java
@Component
@Order(1)
public class DataInitRunner implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        // 应用启动完成后执行
        initBaseData();
    }
}
```

**方式二：`ApplicationRunner`**（参数封装为 `ApplicationArguments`，支持 `--key=value` 格式解析）

```java
@Component
@Order(2)
public class CacheWarmupRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        if (args.containsOption("skip-warmup")) return;
        warmUpCache();
    }
}
```

**方式三：`@EventListener(ApplicationReadyEvent.class)`**（应用完全就绪后触发）

**方式四：`@PostConstruct`**（Bean 初始化后立即执行，早于 Runner）

执行顺序：`@PostConstruct` → `CommandLineRunner`/`ApplicationRunner`（按 `@Order` 排序）。

---

## 自定义 Starter

开发可重用的业务模块时，可以封装成自定义 Starter：

```java
// 1. 配置属性类
@ConfigurationProperties(prefix = "myapp.service")
public class MyServiceProperties {
    private String name = "default";
    private int timeout = 5000;
}

// 2. 自动配置类
@Configuration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyServiceProperties.class)
public class MyServiceAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyServiceProperties props) {
        return new DefaultMyService(props.getName(), props.getTimeout());
    }
}

// 3. 注册（Spring Boot 3.x）
// META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports：
// com.example.MyServiceAutoConfiguration
```

---

## 相关面试题 →

[[../../10_Developlanguage/005_Spring/03_SpringBootSubject/01、Spring Boot 基础概念]]
[[../../10_Developlanguage/005_Spring/03_SpringBootSubject/02、自动配置与启动机制]]
