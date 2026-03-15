###### 1. 什么是 Spring 事件机制？

Spring 事件机制是基于**观察者模式**（发布-订阅模式）的实现，让 Bean 之间可以通过事件进行通信，而不需要直接相互依赖。你发布一个事件，关心它的监听器自动处理，发布方完全不知道谁在监听，实现了真正的松耦合。

整个机制围绕四个角色展开：

- **事件（Event）**：数据载体，继承 `ApplicationEvent`（Spring 4.2 后可以是任意 POJO）
- **发布者（Publisher）**：`ApplicationEventPublisher`，负责发布事件
- **监听器（Listener）**：`ApplicationListener<T>` 或 `@EventListener` 注解的方法，负责处理
- **广播器（Multicaster）**：`ApplicationEventMulticaster`，负责把事件分发给所有匹配的监听器

Spring 内置了几个常用事件：`ContextRefreshedEvent`（容器刷新完成）、`ContextClosedEvent`（容器关闭）、`RequestHandledEvent`（Web 请求处理完成）等，这些都可以直接监听。

Spring 事件机制的核心价值在于实现模块间的解耦，特别适合处理跨组件的业务通知，比如订单创建后通知库存、发邮件、记日志——这些都不应该写在订单服务里。

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/01、Spring事件机制#事件机制概述]]

---

###### 2. 如何自定义 Spring 事件？

创建自定义事件只需要继承 `ApplicationEvent` 类，然后在构造函数里把需要传递的业务数据封装进去：

```java
public class OrderCreatedEvent extends ApplicationEvent {
    private final String orderId;
    private final BigDecimal amount;
    private final String customerId;
    
    public OrderCreatedEvent(Object source, String orderId, BigDecimal amount, String customerId) {
        super(source);  // source 通常是发布事件的对象
        this.orderId = orderId;
        this.amount = amount;
        this.customerId = customerId;
    }
    
    public String getOrderId() { return orderId; }
    public BigDecimal getAmount() { return amount; }
    public String getCustomerId() { return customerId; }
}
```

从 Spring 4.2 开始，事件对象不再强制要求继承 `ApplicationEvent`，普通 POJO 也行，进一步简化了使用。

设计要点：事件类最好是**不可变的**，字段用 `final` 修饰，只提供 getter。理由是事件通常在多个监听器间传递，可变状态会引发并发安全问题。

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/01、Spring事件机制#自定义事件]]

---

###### 3. 如何发布 Spring 事件？

发布事件的推荐方式是注入 `ApplicationEventPublisher`，直接调用 `publishEvent()` 方法：

```java
@Service
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;
    
    public OrderService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }
    
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(createOrderFromRequest(request));
        eventPublisher.publishEvent(new OrderCreatedEvent(this, 
            order.getId(), order.getAmount(), order.getCustomerId()));
        return order;
    }
}
```

另外两种方式：实现 `ApplicationEventPublisherAware` 接口（老式写法）、直接注入 `ApplicationContext`（因为 `ApplicationContext` 本身也实现了 `ApplicationEventPublisher` 接口）。两者功能相同，但不如直接注入 `ApplicationEventPublisher` 接口清晰。

发布时机很重要：通常选在**业务操作完成后的关键节点**，比如数据持久化之后。如果需要保证监听器和发布方在同一个事务里，直接在事务内发布即可；如果希望事务提交后再触发监听器（比如发邮件），用 `@TransactionalEventListener` 更合适。

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/01、Spring事件机制#事件发布方式]]

---

###### 4. @EventListener 注解如何使用？

`@EventListener` 是 Spring 4.2 引入的注解，比实现 `ApplicationListener` 接口简洁得多，直接在方法上加注解就行：

```java
@Component
public class OrderEventListener {
    
    // 基本用法
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        System.out.println("处理订单创建事件，订单ID: " + event.getOrderId());
    }
    
    // 条件监听，只处理金额大于1000的大额订单
    @EventListener(condition = "#event.amount > 1000")
    public void handleLargeOrder(OrderCreatedEvent event) {
        System.out.println("大额订单处理: " + event.getOrderId());
    }
    
    // 同时监听多种事件类型
    @EventListener(classes = {OrderCreatedEvent.class, OrderUpdatedEvent.class})
    public void handleMultipleEvents(ApplicationEvent event) {
        // 分类处理
    }
    
    // 返回值会被当作新事件发布，形成事件处理链
    @EventListener
    public OrderProcessedEvent handleOrderCreated(OrderCreatedEvent event) {
        return new OrderProcessedEvent(this, event.getOrderId(), "PROCESSED");
    }
}
```

跟 `ApplicationListener` 接口比，`@EventListener` 胜在**简洁**——不需要实现整个接口；**灵活**——支持 SpEL 条件表达式、多事件类型、返回值触发链式事件，这些 `ApplicationListener` 都得在方法内手动实现。

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/01、Spring事件机制#EventListener注解用法]]

---

###### 5. Spring 事件机制是同步还是异步的？

**默认是同步的**。当事件发布后，所有匹配的监听器会在**发布者的线程中顺序执行**，直到所有监听器处理完成，`publishEvent()` 才返回。

这意味着：如果有 3 个监听器，其中一个监听器执行了一个耗时 5 秒的操作（比如发送 HTTP 请求），整个调用线程就会被阻塞 5 秒。

从源码看，`SimpleApplicationEventMulticaster` 的 `multicastEvent()` 方法在没有设置 `TaskExecutor` 时，会直接调用 `invokeListener()` 来同步执行监听器。

同步模式的优点是**事务一致性容易保证**（监听器和发布者在同一事务中）、**错误处理简单**（异常会传播给发布者）；缺点是性能瓶颈，一个慢监听器拖累整条链路。

特别要注意的是：如果你在监听器里做了数据库操作，它默认和发布方共用同一个事务。如果想要独立事务，需要在监听方法上加 `@Transactional(propagation = Propagation.REQUIRES_NEW)`。

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/01、Spring事件机制#同步与异步事件]]

---

###### 6. 如何实现异步事件监听？

实现异步事件监听主要有两种方式，推荐优先用第一种。

**方式一：`@Async` + `@EventListener` 组合（推荐）**

先开启异步支持，配好线程池，然后在监听方法上同时加这两个注解：

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        executor.setThreadNamePrefix("Async-Event-");
        executor.initialize();
        return executor;
    }
}

@Slf4j
@Component
public class AsyncOrderEventListener {
    
    @Async("taskExecutor")
    @EventListener
    public void sendOrderConfirmationEmail(OrderCreatedEvent event) {
        // 在独立线程中执行，不会阻塞事件发布者
        log.info("异步发送订单确认邮件，订单ID: {}", event.getOrderId());
    }
}
```

**方式二：自定义 `ApplicationEventMulticaster`**

给广播器设置 `TaskExecutor`，所有事件监听都会变成异步执行。这种方式影响范围是全局的，谨慎使用：

```java
@Bean(name = "applicationEventMulticaster")
public ApplicationEventMulticaster applicationEventMulticaster() {
    SimpleApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster();
    multicaster.setTaskExecutor(Executors.newFixedThreadPool(5));
    return multicaster;
}
```

异步监听有几个注意事项：首先，**异步监听器与发布者不在同一事务中**，需要独立管理事务；其次，**异步执行中的异常不会传播给发布者**，必须在监听方法内自行处理（或配置 `AsyncUncaughtExceptionHandler`）；最后，异步监听器的执行顺序是不确定的，如果需要顺序执行，请用同步模式并配合 `@Order` 注解。

📖 [[../../../../24_SpringKnowledge/07_事件调度与扩展/01、Spring事件机制#异步事件实现]]
