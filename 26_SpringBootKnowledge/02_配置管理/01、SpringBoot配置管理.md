# SpringBoot 配置管理

## 配置文件格式

Spring Boot 支持两种配置格式，功能等价，语法不同：

**Properties 格式**（简单直接）：
```properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
```

**YAML 格式**（层级清晰，复杂配置可读性更好）：
```yaml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test
    username: root
```

团队协作建议统一格式，简单项目用 properties，配置层级多时用 YAML。

---

## 配置加载优先级

Spring Boot 配置优先级遵循"**由外到内、由特到普**"的原则（数字越小优先级越高）：

1. 命令行参数（`--key=value`）— 优先级最高
2. Java 系统属性（`-Dkey=value`）
3. 操作系统环境变量
4. 外部 Profile 特定配置（`application-prod.yml` 放在 jar 同级目录）
5. 外部通用配置（jar 同级目录的 `application.yml`）
6. 内部 Profile 特定配置（classpath 里的 `application-prod.yml`）
7. 内部通用配置（classpath 里的 `application.yml`）
8. `@PropertySource` 注解指定的文件
9. 默认属性（`SpringApplication.setDefaultProperties()`）

**记住核心原则**：命令行参数永远能覆盖一切，方便运维在不改代码的情况下动态调整配置。

---

## 多环境配置（Profile 机制）

**文件组织方式：**
```
src/main/resources/
├── application.yml          # 公共配置（所有环境共用）
├── application-dev.yml     # 开发环境
├── application-test.yml    # 测试环境
└── application-prod.yml    # 生产环境
```

**激活方式：**
```yaml
# application.yml 中设置默认
spring:
  profiles:
    active: dev
```
```bash
# 打包后命令行指定，覆盖默认值
java -jar app.jar --spring.profiles.active=prod

# 或通过环境变量（适合 Docker/K8s 场景）
export SPRING_PROFILES_ACTIVE=prod
```

**配合 @Profile 实现条件化 Bean：**
```java
@Bean
@Profile("dev")
public DataSource devDataSource() {
    return new EmbeddedDatabaseBuilder().setType(H2).build();
}

@Bean
@Profile("prod")
public DataSource prodDataSource() {
    return DataSourceBuilder.create().build();
}
```

---

## @ConfigurationProperties 批量绑定

比 `@Value` 更推荐用于结构化配置，支持松散绑定（kebab-case、camelCase 自动映射）、类型安全、JSR-303 校验：

```yaml
# application.yml
app:
  datasource:
    url: jdbc:mysql://localhost:3306/test
    pool-size: 10    # 自动映射到 poolSize
    timeout: 5000
```

```java
@Component
@ConfigurationProperties(prefix = "app.datasource")
@Validated
public class DataSourceProperties {
    @NotBlank
    private String url;
    
    @Min(1) @Max(100)
    private int poolSize;          // pool-size 自动映射
    
    private Duration timeout;      // 支持 Duration 类型直接映射
    // getters/setters
}
```

**@Value vs @ConfigurationProperties 对比：**
- `@Value`：字段级别，适合注入单个值，不支持松散绑定和校验
- `@ConfigurationProperties`：类级别批量绑定，支持复杂类型（List/Map/Duration），支持 JSR-303 校验，适合结构化配置

---

## 读取自定义配置文件

默认只加载 `application.*`，自定义文件用 `@PropertySource`：

```java
@Configuration
@PropertySource(value = "classpath:custom.properties", encoding = "UTF-8")
public class CustomConfig {
    @Value("${custom.api.url}")
    private String apiUrl;
}
```

注意：`@PropertySource` 不支持 YAML 格式，只支持 `.properties` 文件。

---

## 敏感信息加密

生产环境的密码、密钥等敏感配置，推荐用 Jasypt 加密：

```yaml
spring:
  datasource:
    password: ENC(加密后的密文)
jasypt:
  encryptor:
    password: ${JASYPT_PASSWORD}  # 密钥从环境变量读取，不写死在配置文件里
```

加密/解密用 `StringEncryptor.encrypt()` / `decrypt()` 完成，运行时框架自动识别 `ENC(...)` 并解密。

---

## 配置动态刷新

结合 Spring Cloud Config Server 和 `@RefreshScope`，无需重启即可更新配置：

```java
@RestController
@RefreshScope  // 标记为可刷新的 Bean
public class ConfigController {
    @Value("${dynamic.config.value}")
    private String dynamicValue;
}
```

发送 POST 请求到 `/actuator/refresh` 端点触发刷新，标注了 `@RefreshScope` 的 Bean 会重新创建并读取最新配置。

---

## 相关面试题 →

[[../../10_Developlanguage/005_Spring/03_SpringBootSubject/03、配置管理]]
