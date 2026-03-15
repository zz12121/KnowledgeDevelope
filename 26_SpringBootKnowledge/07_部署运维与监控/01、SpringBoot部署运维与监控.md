# SpringBoot 监控与运维

## 一、Spring Boot Actuator

Actuator 是 Spring Boot 提供的生产级监控模块，通过内置 RESTful 端点暴露应用内部状态。

**四大核心价值：**

- **健康监控**：检查数据库、磁盘、Redis 等关键组件状态
- **指标收集**：JVM 内存、GC、HTTP 请求统计等运行时指标
- **操作控制**：动态修改日志级别、优雅关闭应用
- **生态集成**：通过 Micrometer 与 Prometheus/Grafana 无缝对接

**常用端点：**

| 端点 | 路径 | 生产环境建议 |
|------|------|------------|
| 健康检查 | `/actuator/health` | 必须开放 |
| 指标监控 | `/actuator/metrics` | 推荐开放 |
| 环境信息 | `/actuator/env` | 受限访问 |
| 日志控制 | `/actuator/loggers` | 受限访问 |
| 线程信息 | `/actuator/threaddump` | 故障排查时开放 |
| URL 映射 | `/actuator/mappings` | 调试时开放 |
| Bean 信息 | `/actuator/beans` | 调试时开放 |

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,info,prometheus
      base-path: /monitor
  endpoint:
    health:
      show-details: always
```

---

## 二、自定义健康检查

实现 `HealthIndicator` 接口或继承 `AbstractHealthIndicator` 类。

```java
@Component
public class DatabaseHealthIndicator extends AbstractHealthIndicator {

    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        if (isDatabaseConnected()) {
            builder.up()
                   .withDetail("database", "MySQL")
                   .withDetail("responseTime", checkResponseTime() + "ms");
        } else {
            builder.down()
                   .withDetail("error", "数据库连接失败");
        }
    }
}
```

**健康状态聚合：** `SimpleStatusAggregator` 按严重程度排序（DOWN > OUT_OF_SERVICE > UP > UNKNOWN），返回最严重的状态。

**Kubernetes 就绪/存活探针与 Actuator 集成：**

- 存活探针：`/actuator/health/liveness` — 应用是否需要重启
- 就绪探针：`/actuator/health/readiness` — 应用是否可以接收流量

---

## 三、Prometheus + Grafana 监控

**集成步骤：**

1. 引入 `spring-boot-starter-actuator` 和 `micrometer-registry-prometheus`
2. 暴露 Prometheus 端点：`management.endpoints.web.exposure.include: prometheus`
3. Prometheus 配置抓取：

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8080']
    scrape_interval: 15s
```

4. Grafana 导入 Spring Boot 监控仪表板（ID: 6756）

**核心监控指标：**

- `jvm_memory_used_bytes`：JVM 内存使用
- `jvm_gc_pause_seconds`：GC 停顿时间
- `http_server_requests_seconds`：HTTP 请求延迟
- `tomcat_threads_busy_threads`：Tomcat 繁忙线程
- `hikaricp.connections.active`：活跃数据库连接

**自定义业务指标：**

```java
@Service
public class OrderMetricsService {

    private final Counter orderCounter;
    private final Timer orderProcessingTimer;

    public OrderMetricsService(MeterRegistry registry) {
        orderCounter = Counter.builder("orders.total")
            .tag("type", "online")
            .register(registry);
        orderProcessingTimer = Timer.builder("orders.processing.time")
            .register(registry);
    }

    public void recordOrder(Order order) {
        orderCounter.increment();
        Timer.Sample sample = Timer.start();
        processOrder(order);
        sample.stop(orderProcessingTimer);
    }
}
```

---

## 四、APM 应用性能监控

**Micrometer 是 Java 指标收集的通用 API**，支持多种监控后端（Prometheus、InfluxDB、Datadog 等）。

**方法级性能监控：**

```java
@Service
public class UserService {

    @Timed(value = "user.service.find.by.id", description = "根据ID查询用户时间")
    @Counted(value = "user.service.find.by.id.count")
    public User findUserById(Long id) {
        return userRepository.findById(id);
    }
}
```

需要在配置类中注册 `TimedAspect` Bean：

```java
@Bean
public TimedAspect timedAspect(MeterRegistry registry) {
    return new TimedAspect(registry);
}
```

---

## 五、日志管理

### Logback 配置

```xml
<configuration>
    <property name="LOG_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"/>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxHistory>30</maxHistory>
            <maxFileSize>100MB</maxFileSize>
        </rollingPolicy>
    </appender>

    <root level="INFO">
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

### 动态日志级别

通过 Actuator `loggers` 端点动态调整，无需重启应用：

```bash
# 查看当前级别
curl http://localhost:8080/actuator/loggers/com.example.service

# 动态修改为 DEBUG
curl -X POST -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}' \
  http://localhost:8080/actuator/loggers/com.example.service
```

### 日志分级配置

```yaml
logging:
  level:
    com.example.service: INFO
    org.springframework.web: WARN
    org.hibernate.SQL: DEBUG
  group:
    business: com.example.service,com.example.repository
```

---

## 六、ELK 日志集中收集

**架构：** Spring Boot → Logstash（采集/过滤）→ Elasticsearch（存储）→ Kibana（可视化）。

**Spring Boot 接入 Logstash：**

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>6.6</version>
</dependency>
```

```xml
<!-- logback.xml -->
<appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>logstash:5000</destination>
    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
        <providers>
            <timestamp/><logLevel/><loggerName/>
            <threadName/><mdc/><message/><stackTrace/>
        </providers>
    </encoder>
</appender>
```

**MDC（Mapped Diagnostic Context）** 可以在日志中携带请求 traceId、userId 等上下文信息，方便链路追踪。

---

## 七、应用优雅关闭

**Spring Boot 2.3+ 内置优雅关闭支持：**

```yaml
server:
  shutdown: graceful  # 等待正在处理的请求完成

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # 等待超时时间
```

**自定义关闭钩子：**

```java
@Component
public class GracefulShutdown implements ApplicationListener<ContextClosedEvent> {

    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        // 1. 停止接收新请求
        // 2. 等待进行中的请求完成（最多 30s）
        // 3. 关闭线程池
        // 4. 关闭数据库连接池
        // 5. 其他资源清理
    }
}
```

**关闭触发方式：**

- `kill -15 <pid>`：发送 SIGTERM 信号，触发优雅关闭
- Actuator shutdown 端点：`POST /actuator/shutdown`（需开启）
- Kubernetes `preStop` Hook：在容器停止前执行清理

---

## 关联面试题

- 📝 [[10_Developlanguage/005_Spring/03_SpringBootSubject/08、监控与运维]]
