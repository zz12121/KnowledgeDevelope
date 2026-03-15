# Spring 事件机制

## 1 什么是 Spring 事件机制

Spring 事件机制基于**观察者模式（发布-订阅模式）**，为组件间提供**松耦合的通信方式**。Bean 之间通过事件通信，不需要直接依赖对方。

**四个核心角色**：
- **事件（Event）**：封装事件数据的载体，继承 `ApplicationEvent`（4.2+ 可以是普通 POJO）
- **事件发布者（Publisher）**：通过 `ApplicationEventPublisher` 接口发布事件
- **事件监听器（Listener）**：实现 `ApplicationListener<T>` 接口或使用 `@EventListener` 注解
- **事件广播器（Multicaster）**：`ApplicationEventMulticaster` 负责将事件分发给所有匹配的监听器

**Spring 内置事件**：
- `ContextRefreshedEvent`：容器 `refresh()` 完成后发布（可用于触发应用初始化逻辑）
- `ContextClosedEvent`：容器关闭时发布
- `ContextStartedEvent` / `ContextStoppedEvent`：容器启动/停止时发布
- `RequestHandledEvent`：Web 请求处理完成后发布

## 2 自定义事件

继承 `ApplicationEvent`，封装业务数据（推荐不可变设计）：

```java
public class OrderCreatedEvent extends ApplicationEvent {
    private final String orderId;
    private final BigDecimal amount;

    public OrderCreatedEvent(Object source, String orderId, BigDecimal amount) {
        super(source);  // source 通常是发布事件的对象（this）
        this.orderId = orderId;
        this.amount = amount;
    }

    public String getOrderId() { return orderId; }
    public BigDecimal getAmount() { return amount; }
}
```

Spring 4.2+ 起，事件对象可以是任意 POJO，不需要继承 `ApplicationEvent`——Spring 会自动包装：

```java
// 简单 POJO 事件，更简洁
public class OrderCreatedEvent {
    private final String orderId;
    // ...
}
```

## 3 发布事件

推荐通过注入 `ApplicationEventPublisher` 发布：

```java
@Service
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;

    public OrderService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(buildOrder(request));

        // 事务提交前发布事件（同步模式下监听器在同一事务中执行）
        eventPublisher.publishEvent(new OrderCreatedEvent(this, order.getId(), order.getAmount()));

        return order;
    }
}
```

`ApplicationContext` 本身也实现了 `ApplicationEventPublisher`，也可以直接用 `@Autowired ApplicationContext` 发布，但语义上注入 `ApplicationEventPublisher` 接口更清晰。

## 4 监听事件：@EventListener

Spring 4.2 引入的 `@EventListener` 比实现接口更简洁，推荐使用：

```java
@Component
public class OrderEventListener {

    // 基本用法：方法参数类型决定监听哪种事件
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("订单创建：{}", event.getOrderId());
    }

    // SpEL 条件过滤：只处理金额大于 1000 的订单
    @EventListener(condition = "#event.amount > 1000")
    public void handleLargeOrder(OrderCreatedEvent event) {
        log.info("大额订单：{}", event.getOrderId());
    }

    // 监听多个事件类型
    @EventListener(classes = {OrderCreatedEvent.class, OrderUpdatedEvent.class})
    public void handleOrderEvents(ApplicationEvent event) {
        if (event instanceof OrderCreatedEvent) { /* ... */ }
    }

    // 返回值生成新事件（事件链）
    @EventListener
    public OrderProcessedEvent processOrder(OrderCreatedEvent event) {
        // 处理完成后，返回的对象会被自动发布为新事件
        return new OrderProcessedEvent(this, event.getOrderId());
    }
}
```

## 5 事件监听：ApplicationListener 接口

更传统的方式，类型安全但需要实现整个接口：

```java
@Component
public class OrderCreatedListener implements ApplicationListener<OrderCreatedEvent> {
    @Override
    public void onApplicationEvent(OrderCreatedEvent event) {
        log.info("处理订单：{}", event.getOrderId());
    }
}
```

`@EventListener` vs `ApplicationListener` 对比：前者支持条件过滤、监听多种类型、返回值生成新事件，写法更灵活；后者泛型类型安全，但功能相对受限。现代项目推荐 `@EventListener`。

## 6 同步 vs 异步事件

**默认是同步的**：事件监听器在事件发布者的**调用线程中顺序执行**，会阻塞发布者直到所有监听器处理完成。

**同步执行的优势**：监听器与发布者在同一事务中，事务一致性有保证；异常能传播到发布者。

**同步执行的问题**：一个慢监听器会拖慢整个调用链；某些场景（如发邮件、推送通知）没必要同步等待。

**异步方式一：@Async（推荐）**

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean("eventExecutor")
    public Executor eventExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("Event-");
        executor.initialize();
        return executor;
    }
}

@Component
public class AsyncOrderEventListener {

    @Async("eventExecutor")
    @EventListener
    @Transactional(propagation = Propagation.REQUIRES_NEW)  // 异步要用独立事务
    public void sendEmail(OrderCreatedEvent event) {
        // 在独立线程中执行，不阻塞发布者
        emailService.sendOrderConfirmation(event.getOrderId());
    }
}
```

**异步方式二：自定义 ApplicationEventMulticaster**（全局异步，影响所有监听器）：

```java
@Bean("applicationEventMulticaster")
public ApplicationEventMulticaster applicationEventMulticaster() {
    SimpleApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster();
    multicaster.setTaskExecutor(Executors.newFixedThreadPool(5));
    return multicaster;
}
```

**异步注意事项**：
- 异步监听器与发布者不在同一事务中，需要独立事务管理
- 异步执行中的异常不会传播到发布者，需单独的错误处理
- 异步执行顺序无法保证

## 7 事务与事件的配合

一个常见场景：订单创建后发送通知，但希望在**事务提交成功后**才发送，避免事务回滚时通知已发出：

```java
@Component
public class OrderCreatedListener {

    @EventListener
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)  // 事务提交后才执行
    public void sendNotification(OrderCreatedEvent event) {
        // 只有当发布事件的事务成功提交后，这里才会被调用
        notificationService.notify(event.getOrderId());
    }
}
```

`@TransactionalEventListener` 是 Spring 4.2 引入的，支持四个阶段：`AFTER_COMMIT`（提交后）、`AFTER_ROLLBACK`（回滚后）、`AFTER_COMPLETION`（完成后，无论提交还是回滚）、`BEFORE_COMMIT`（提交前）。

---

相关面试题 → [[../../10_DevelopLanguage/005_Spring/01_SpringSubject/08、Spring 事件机制|08、Spring事件机制]]
