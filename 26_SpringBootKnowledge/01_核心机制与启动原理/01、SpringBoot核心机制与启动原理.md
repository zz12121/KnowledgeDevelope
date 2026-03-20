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

## PDF 补充内容

### 1. SpringBoot Starter 分类

SpringBoot 提供的 Starter 分为三类：

| 分类 | 说明 | 示例 |
|------|------|------|
| **application starters** | 应用级 Starter，最常用 | spring-boot-starter-web、spring-boot-starter-data-jpa |
| **production starters** | 生产级 Starter，提供监控、健康检查等 | spring-boot-starter-actuator |
| **technical starters** | 技术级 Starter，集成第三方技术 | spring-boot-starter-undertow、spring-boot-starter-logging |

**核心 Starter：**
- `spring-boot-starter`：核心 Starter，包含自动配置、日志、YAML 配置支持
- `spring-boot-starter-web`：Web 应用 Starter，内嵌 Tomcat
- `spring-boot-starter-actuator`：生产级监控 Starter

### 2. @SpringBootApplication 注解源码解析

`@SpringBootApplication` 是一个复合注解，展开后包含三个核心注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@SpringBootConfiguration    // 等价于 @Configuration
@EnableAutoConfiguration    // 启用自动配置
@ComponentScan              // 组件扫描（默认扫描主类所在包及其子包）
public @interface SpringBootApplication {
    // 排除特定自动配置类
    Class<?>[] exclude() default {};
    // 排除特定自动配置类名称
    String[] excludeName() default {};
    // 指定包扫描路径
    String[] scanBasePackages() default {};
    // 指定要扫描的组件类型
    Class<?>[] scanBasePackageClasses() default {};
}
```

**注意**：`scanBasePackages` 默认值为空，实际效果等价于扫描主类所在的包及其子包。

### 3. 按需开启自动配置

SpringBoot 虽然加载了 `spring.factories`（Spring Boot 2.x）或 `spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（Spring Boot 3.x）中所有配置的类，但并非全部加载到 IOC 容器中，而是采用**按需加载**（即 `@ConditionalOnXXX` 条件注解）的方式进行加载。

**按需加载的应用场景：**

| 场景 | 示例 | 说明 |
|------|------|------|
| 容错兼容 | `DispatcherServletAutoConfiguration.multipartResolver` | 如果用户已配置，则使用用户的配置 |
| 用户配置优先 | `WebMvcAutoConfiguration.defaultViewResolver` | 自动配置只是默认值，用户配置优先 |
| 外部配置项修改组件行为 | `WebMvcAutoConfiguration.messageConverters` | 通过配置属性控制组件行为 |

**查看自动配置情况：**
```yaml
# 在 application.yml 中添加
debug: true
```
启动应用后，控制台会输出详细的自动配置报告。

### 4. 常用注解详解

| 注解 | 说明 |
|------|------|
| `@Configuration` | 定义配置类，之前的 Spring 配置写在 xml 里，现在建议写在配置类中 |
| `@ComponentScan` | 定义扫描路径，默认扫描主类所在包及其子包 |
| `@Bean` | 默认方法名就是 bean 的 id，返回类型就是方法返回的类型。也可 `@Bean("xxx")` 指定 bean 名称 |
| `@Import` | 给容器中自动创建出注解中指定的组件，默认组件名字就是全类名 |
| `@Conditional` | 满足指定条件时，才向 IOC 容器中注入组件 |
| `@ImportResource` | 指定对应的 xml 文件，Spring 就可以把 xml 中配置的 Bean 都加载到 IOC 中 |

### 5. 配置绑定方式

**方式一：@ConfigurationProperties + @Component**
```java
@Component
@ConfigurationProperties(prefix = "my.config")
public class MyProperties {
    private String name;
    private int timeout;
}
```

**方式二：@ConfigurationProperties + @EnableConfigurationProperties**
```java
@Configuration
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {
}
```

**添加配置提醒（IDE 提示）：**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

### 6. YAML 书写规则

- 大小写敏感
- 使用缩进表示层级关系，缩进不允许使用 tab，只允许空格
- 缩进的空格数不重要，只要层级对齐即可
- '#' 表示注释
- 字符串无需加引号，如果加了，双引号表示转义字符，单引号表示普通字符

---

## 参考资料

> 本章节对应的原始参考资料和深入学习资源：

- **[SpringBoot-自动装配.md](./E:/md/1/SpringBoot/SpringBoot-自动装配.md)** — 自动装配原理深度解析
- **[SpringBoot-application-load.md](./E:/md/1/SpringBoot/SpringBoot-application-load.md)** — 应用启动流程源码分析
- **[Spring-Boot-Run.md](./E:/md/1/SpringBoot/Spring-Boot-Run.md)** — Spring Boot 启动原理
- **[SpringBoot-ConditionalOnBean.md](./E:/md/1/SpringBoot/SpringBoot-ConditionalOnBean.md)** — 条件装配机制
- **[SpringBoot-ConfigurationProperties.md](./E:/md/1/SpringBoot/SpringBoot-ConfigurationProperties.md)** — 配置属性绑定原理
- **[参考资料索引](../参考资料索引.md)** — 所有参考资料的总索引

**知识库双链**：
- 面试题库：[10_DevelopLanguage/005_Spring/03_SpringBootSubject/01、Spring Boot 基础概念.md](../../10_DevelopLanguage/005_Spring/03_SpringBootSubject/01、Spring%20Boot%20基础概念.md) — Spring Boot 面试题

---

## 相关面试题 →

[[../../10_Developlanguage/005_Spring/03_SpringBootSubject/01、Spring Boot 基础概念]]
[[../../10_Developlanguage/005_Spring/03_SpringBootSubject/02、自动配置与启动机制]]
