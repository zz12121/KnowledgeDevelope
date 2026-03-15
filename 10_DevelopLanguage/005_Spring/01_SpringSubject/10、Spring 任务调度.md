###### 1. Spring 如何实现定时任务？

Spring 定时任务基于**任务调度抽象层**实现，整个流程分三个阶段。

**任务注册阶段**：`@EnableScheduling` 通过 `@Import` 导入 `SchedulingConfiguration`，注册 `ScheduledAnnotationBeanPostProcessor`。这个后处理器在 Bean 初始化后阶段扫描所有带 `@Scheduled` 注解的方法，解析注解属性，封装为 `Task` 对象注册到 `ScheduledTaskRegistrar` 中。

**任务调度阶段**：`ScheduledTaskRegistrar` 管理所有注册的任务。应用启动完成后，调用 `TaskScheduler` 的调度方法。Spring Boot 默认配置 `ThreadPoolTaskScheduler`，内部封装了 JDK 的 `ScheduledThreadPoolExecutor`。

**任务执行阶段**：调度时间到达时，`ThreadPoolTaskScheduler` 从线程池分配工作线程执行任务，实际上是通过反射调用目标业务方法。

底层本质是 JDK 的 `ScheduledExecutorService`，Spring 做的是注解解析、任务注册和容器生命周期整合这些上层工作。

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/02、任务调度与异步执行#定时任务实现原理]]

---

###### 2. @Scheduled 注解如何使用？

`@Scheduled` 标记一个方法为定时任务，需配合 `@EnableScheduling` 使用。支持四种调度策略：

**`fixedRate`**（固定频率）：从上次开始执行到下次开始执行的间隔是固定的，不管任务本身执行了多久。适合严格定期执行的场景，比如每 5 秒采集一次监控数据。

**`fixedDelay`**（固定延迟）：上次执行完成后，等待指定时间再执行下次。适合需要保证两次执行之间有间隔的场景，比如队列消费、消息轮询。

**`cron`**（Cron 表达式）：适合复杂调度需求，比如每天凌晨 2 点执行。

**`initialDelay`**（初始延迟）：第一次执行前的等待时间，常与 `fixedRate` 或 `fixedDelay` 配合用，给应用留出完整启动时间。

```java
@Component
public class ReportSchedulerService {
    
    @Scheduled(fixedRate = 5000)
    public void collectMetrics() { /* 每5秒采集一次 */ }
    
    @Scheduled(fixedDelay = 3000)
    public void processQueue() { /* 上次处理完后等3秒再处理 */ }
    
    @Scheduled(cron = "0 0 2 * * ?")
    public void generateDailyReport() { /* 每天凌晨2点 */ }
    
    @Scheduled(initialDelay = 10000, fixedRate = 60000)
    public void checkHealth() { /* 启动10秒后开始，之后每分钟检查 */ }
}
```

**最重要的注意事项**：默认所有 `@Scheduled` 任务由**单线程**执行。如果一个任务耗时超过调度间隔，后续任务会排队等待，形成任务积压。生产环境务必配置多线程调度器（见第 8 题）。

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/02、任务调度与异步执行#Scheduled注解用法]]

---

###### 3. @EnableScheduling 注解的作用是什么？

`@EnableScheduling` 是一个**模块驱动注解**，用于全局启用定时任务功能。

它通过 `@Import` 导入 `SchedulingConfiguration` 配置类，该配置类注册核心组件 `ScheduledAnnotationBeanPostProcessor`。这个后处理器负责扫描 Bean 中的 `@Scheduled` 注解方法，并将它们注册到 `ScheduledTaskRegistrar` 中等待执行。

在 Spring Boot 项目里，`@EnableScheduling` 还会触发 `TaskSchedulingAutoConfiguration`，自动配置 `ThreadPoolTaskScheduler`。

简单理解：`@EnableScheduling` 是"引擎"，`@Scheduled` 是"油门"——引擎不启动，踩再多油门也没用。

```java
@SpringBootApplication
@EnableScheduling
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/02、任务调度与异步执行#EnableScheduling作用]]

---

###### 4. Cron 表达式如何编写？

Spring 的 Cron 表达式由 **6 个字段**组成（也可以有第 7 个可选的年字段），格式为：`秒 分 时 日 月 周 [年]`。

**各字段取值范围：**
- 秒：0–59
- 分：0–59
- 时：0–23
- 日：1–31（注意：日和周字段互斥，用到一个时另一个要写 `?`）
- 月：1–12 或 JAN–DEC
- 周：1–7 或 SUN–SAT（1 = 周日）
- 年：1970–2099（可选）

**特殊字符含义：**
- `*`：匹配任意值，如 `* * * * * ?` 表示每秒
- `?`：放弃指定，日和周字段专用
- `/`：间隔，如 `0/30` 表示从 0 开始每 30 单位一次
- `-`：范围，如 `1-5` 表示 1 到 5
- `,`：枚举，如 `MON,WED,FRI` 表示周一三五

**常用表达式速查：**
```
0 0 * * * ?          每小时整点
0 */30 * * * ?        每30分钟
0 0 9 * * ?           每天上午9点
0 0 12 ? * MON-FRI    周一至周五的中午12点
0 0 2 * * ?           每天凌晨2点
0 0 0 1 * ?           每月1号的0点
0 0 0 L * ?           每月最后一天的0点
0 0 10 ? * 2#1        每月第一个周一上午10点
```

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/02、任务调度与异步执行#Cron表达式]]

---

###### 5. 如何实现异步任务？

Spring 实现异步任务的核心是 `@Async` 注解配合 `@EnableAsync`，底层基于 Spring AOP 代理机制。

启用 `@EnableAsync` 后，Spring 注册 `AsyncAnnotationBeanPostProcessor`，为包含 `@Async` 方法的 Bean 创建代理。调用代理对象的异步方法时，被 `AsyncExecutionInterceptor` 拦截，将方法调用封装为任务提交给线程池执行，主线程立即返回。

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    // 可选：配置自定义线程池
}

@Service
public class EmailService {
    
    @Async
    public void sendWelcomeEmail(String userEmail) {
        // 在独立线程中执行，不阻塞调用方
    }
    
    @Async
    public Future<String> processDataAsync(String data) {
        // 可以有返回值，调用方通过 Future.get() 获取结果
        return new AsyncResult<>("处理完成");
    }
    
    // 指定使用特定线程池（适合区分 IO 密集型和 CPU 密集型任务）
    @Async("batchJobExecutor")
    public CompletableFuture<String> executeBatchJob(String jobId) {
        return CompletableFuture.completedFuture("Job completed");
    }
}
```

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/02、任务调度与异步执行#Async异步任务]]

---

###### 6. @Async 注解的限制与陷阱有哪些？

`@Async` 有几个容易踩的坑，务必牢记。

**自调用失效**：`@Async` 基于 AOP 代理，同一个类内部方法互相调用，绕过了代理，注解不生效。解决方法是把异步方法放到另一个 Bean 里，或者通过 `ApplicationContext.getBean()` 获取代理对象再调用。

**只能用于 public 方法**：AOP 代理只能拦截 public 方法，在 private 或 protected 方法上加 `@Async` 没有效果。

**异常处理**：异步方法中抛出的异常不会传播给调用方。对于无返回值的方法，必须实现 `AsyncUncaughtExceptionHandler` 来处理；对于有返回值的方法（`Future<T>`），调用 `get()` 时可以捕获 `ExecutionException`。

**默认线程池问题**：如果没有配置自定义线程池，Spring 会使用 `SimpleAsyncTaskExecutor`，这个执行器每次都创建新线程，**没有线程复用**，高并发下会有性能问题，生产环境务必替换成 `ThreadPoolTaskExecutor`。

**事务不跨线程**：事务绑定在 ThreadLocal 上，`@Async` 方法在新线程里执行，无法继承调用方的事务，需要独立管理。

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/02、任务调度与异步执行#Async注解陷阱]]

---

###### 7. @EnableAsync 注解的作用是什么？

`@EnableAsync` 用于在 Spring 应用中激活异步方法执行功能，内部通过 `@Import` 导入 `AsyncConfigurationSelector`，根据代理模式（JDK 代理还是 AspectJ 织入）选择对应的配置类。

核心是注册 `AsyncAnnotationBeanPostProcessor`，该后处理器负责扫描 Bean 中的 `@Async` 注解方法并为其创建代理。

高级配置可以通过实现 `AsyncConfigurer` 接口来自定义线程池和异常处理器：

```java
@Configuration
@EnableAsync
public class AsyncConfiguration implements AsyncConfigurer {
    
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("MyAsync-");
        executor.initialize();
        return executor;
    }
    
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("异步方法 {} 执行异常: {}", method.getName(), ex.getMessage());
        };
    }
}
```

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/02、任务调度与异步执行#EnableAsync作用]]

---

###### 8. 如何配置线程池？

定时任务线程池和异步任务线程池要分开配置，Spring 默认都是单线程，生产环境必须替换。

**定时任务线程池**（通过 `SchedulingConfigurer` 接口）：

```java
@Configuration
@EnableScheduling
public class SchedulerConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("MyScheduledTask-");
        scheduler.setAwaitTerminationSeconds(30);
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.initialize();
        taskRegistrar.setTaskScheduler(scheduler);
    }
}
```

**异步任务线程池**（通过 `AsyncConfigurer` 接口）：

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("MyAsync-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        executor.initialize();
        return executor;
    }
}
```

**多线程池配置**（区分 IO 密集型和 CPU 密集型）：

```java
@Bean("ioBoundExecutor")
public Executor ioBoundExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(20);  // IO 任务可以设置更大的核心数
    executor.setThreadNamePrefix("IOAsync-");
    executor.initialize();
    return executor;
}

@Bean("cpuBoundExecutor")
public Executor cpuBoundExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(Runtime.getRuntime().availableProcessors());
    executor.setThreadNamePrefix("CPUAsync-");
    executor.initialize();
    return executor;
}
```

参数调优的基本原则：IO 密集型任务（网络请求、数据库查询），核心线程数可以设置为 CPU 核数的 2 倍甚至更高，因为线程大部分时间在等待；CPU 密集型任务（计算、压缩），核心线程数接近 CPU 核数就够了，设太多反而增加上下文切换开销。

拒绝策略推荐 `CallerRunsPolicy`，它在队列满时让调用方线程自己来执行任务，既有背压效果，又不会丢任务。

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/02、任务调度与异步执行#线程池配置]]
