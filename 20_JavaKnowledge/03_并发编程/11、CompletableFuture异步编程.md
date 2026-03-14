# 11、CompletableFuture 异步编程

> **定位**：JDK8 引入，实现 `Future` + `CompletionStage` 接口，支持链式异步编程、函数式组合、异常处理，是 Java 异步编排的核心工具。

---

## 一、Future 的缺陷 → CompletableFuture 的诞生

```java
// 旧 Future 的问题
Future<String> future = executor.submit(() -> fetchData());
future.get(); // 1. 阻塞等待，无法异步回调
              // 2. 无法组合多个 Future
              // 3. 异常处理繁琐
              // 4. 无法手动完成
```

**CompletableFuture 解决了什么：**

| 痛点 | 解决方案 |
|---|---|
| 阻塞等待 | `.thenApply()` 链式回调，无需阻塞 |
| 无法组合 | `thenCombine` / `allOf` / `anyOf` |
| 异常处理 | `exceptionally` / `handle` |
| 手动完成 | `complete()` / `completeExceptionally()` |

---

## 二、创建 CompletableFuture

```java
// 1. 无返回值异步任务（默认使用 ForkJoinPool.commonPool）
CompletableFuture<Void> f1 = CompletableFuture.runAsync(() -> doWork());

// 2. 有返回值异步任务
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> fetchData());

// 3. 指定自定义线程池（生产推荐！）
Executor executor = Executors.newFixedThreadPool(10);
CompletableFuture<String> f3 = CompletableFuture.supplyAsync(() -> fetchData(), executor);

// 4. 手动创建并完成
CompletableFuture<String> f4 = new CompletableFuture<>();
f4.complete("result");               // 正常完成
f4.completeExceptionally(new RuntimeException()); // 异常完成

// 5. 已完成的 Future（工具方法）
CompletableFuture<String> f5 = CompletableFuture.completedFuture("已完成");
```

---

## 三、链式变换（核心 API）

### 3.1 thenApply / thenApplyAsync — 转换结果

```java
CompletableFuture<Integer> future = CompletableFuture
    .supplyAsync(() -> "hello")      // 异步：返回 String
    .thenApply(s -> s.length());     // 同步回调：String → Integer
    // .thenApplyAsync(s -> s.length(), executor); // 异步回调

// thenApply  ：在完成该任务的线程中执行回调（可能是调用者线程）
// thenApplyAsync：在 ForkJoinPool 或指定 executor 中执行回调
```

### 3.2 thenAccept / thenRun — 消费结果

```java
CompletableFuture.supplyAsync(() -> getData())
    .thenAccept(data -> System.out.println("收到：" + data))  // 消费结果，无返回
    .thenRun(() -> System.out.println("全部完成"));           // 不关心结果，执行动作
```

### 3.3 thenCompose — 扁平化嵌套（类似 flatMap）

```java
// 错误写法：产生 CompletableFuture<CompletableFuture<String>>
CompletableFuture<CompletableFuture<String>> nested =
    CompletableFuture.supplyAsync(() -> userId)
        .thenApply(id -> queryUser(id));  // queryUser 返回 CF<String>

// 正确写法：thenCompose 扁平化
CompletableFuture<String> flat =
    CompletableFuture.supplyAsync(() -> userId)
        .thenCompose(id -> queryUser(id));  // 扁平化为 CF<String>
```

---

## 四、组合多个 CompletableFuture

### 4.1 thenCombine — 两个任务都完成后合并

```java
CompletableFuture<String> userFuture = CompletableFuture.supplyAsync(() -> getUser());
CompletableFuture<String> orderFuture = CompletableFuture.supplyAsync(() -> getOrder());

CompletableFuture<String> combined = userFuture.thenCombine(
    orderFuture,
    (user, order) -> user + " | " + order  // 两者都完成时执行
);
```

### 4.2 allOf — 等待所有任务完成

```java
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);
all.join(); // 等待全部完成

// 获取所有结果（allOf 返回 Void，需手动聚合）
CompletableFuture<List<String>> allResults = CompletableFuture.allOf(f1, f2, f3)
    .thenApply(v -> Stream.of(f1, f2, f3)
        .map(CompletableFuture::join)
        .collect(Collectors.toList()));
```

### 4.3 anyOf — 任意一个完成就返回

```java
CompletableFuture<Object> fastest = CompletableFuture.anyOf(f1, f2, f3);
Object result = fastest.join(); // 返回最快完成的那个结果
```

### 4.4 对比

| 方法 | 说明 |
|---|---|
| `thenCombine` | 两个 CF 都完成，合并两个结果 |
| `thenAcceptBoth` | 两个 CF 都完成，消费两个结果（无返回值） |
| `applyToEither` | 任意一个完成，转换结果 |
| `acceptEither` | 任意一个完成，消费结果 |
| `allOf` | 所有完成，返回 CF\<Void\> |
| `anyOf` | 任意一个完成，返回 CF\<Object\> |

---

## 五、异常处理

### 5.1 exceptionally — 兜底处理

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> {
        if (Math.random() > 0.5) throw new RuntimeException("失败");
        return "成功";
    })
    .exceptionally(ex -> {
        log.error("发生异常：", ex);
        return "默认值";  // 兜底返回
    });
```

### 5.2 handle — 无论成功失败都处理

```java
future.handle((result, ex) -> {
    if (ex != null) {
        log.error("异常", ex);
        return "默认值";
    }
    return result.toUpperCase();
});
```

### 5.3 whenComplete — 处理完成事件（不改变结果）

```java
future.whenComplete((result, ex) -> {
    // 无论成功/失败都会执行，但不能改变最终值
    if (ex != null) log.error("异常", ex);
    else log.info("结果：{}", result);
});
```

### 异常处理 API 对比

| 方法 | 是否改变结果 | 参数 | 返回值 |
|---|---|---|---|
| `exceptionally` | 是（仅失败时） | `Throwable` | `T` |
| `handle` | 是（成功/失败都可改） | `(T, Throwable)` | `U` |
| `whenComplete` | 否 | `(T, Throwable)` | `void` |

---

## 六、超时控制（JDK9+）

```java
// JDK9 新增
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> slowOperation())
    .orTimeout(3, TimeUnit.SECONDS)          // 超时抛出 TimeoutException
    .completeOnTimeout("默认值", 3, TimeUnit.SECONDS); // 超时返回默认值
```

**JDK8 超时处理（手动实现）：**

```java
public static <T> CompletableFuture<T> withTimeout(
        CompletableFuture<T> future, long timeout, TimeUnit unit, T defaultValue) {
    CompletableFuture<T> timeoutFuture = new CompletableFuture<>();
    scheduler.schedule(() -> timeoutFuture.complete(defaultValue), timeout, unit);
    return future.applyToEither(timeoutFuture, Function.identity());
}
```

---

## 七、实战场景

### 场景1：电商首页聚合查询（并行加速）

```java
public PageVO buildHomePage(Long userId) {
    CompletableFuture<UserInfo> userFuture =
        CompletableFuture.supplyAsync(() -> userService.getUser(userId), executor);

    CompletableFuture<List<Order>> orderFuture =
        CompletableFuture.supplyAsync(() -> orderService.getOrders(userId), executor);

    CompletableFuture<List<Product>> recommendFuture =
        CompletableFuture.supplyAsync(() -> recommendService.recommend(userId), executor);

    // 等待全部完成
    CompletableFuture.allOf(userFuture, orderFuture, recommendFuture).join();

    return PageVO.builder()
        .user(userFuture.join())
        .orders(orderFuture.join())
        .recommendations(recommendFuture.join())
        .build();
}
```

### 场景2：重试机制

```java
public CompletableFuture<String> withRetry(Supplier<String> supplier, int maxRetry) {
    CompletableFuture<String> future = CompletableFuture.supplyAsync(supplier, executor);
    for (int i = 0; i < maxRetry; i++) {
        future = future.exceptionally(ex -> supplier.get());
    }
    return future;
}
```

### 场景3：流水线处理

```java
CompletableFuture.supplyAsync(() -> fetchRawData())          // 1. 获取原始数据
    .thenApplyAsync(data -> parse(data), parseExecutor)      // 2. 解析（IO线程池）
    .thenApplyAsync(parsed -> enrich(parsed), enrichExecutor) // 3. 数据增强
    .thenAcceptAsync(result -> save(result), dbExecutor)     // 4. 入库
    .exceptionally(ex -> { log.error("流水线异常", ex); return null; });
```

---

## 八、踩坑总结

### 坑1：默认使用 commonPool 导致阻塞

```java
// 危险！IO 操作阻塞了 ForkJoinPool.commonPool
CompletableFuture.supplyAsync(() -> httpClient.get(url)); // 别这么做

// 正确：IO 场景用独立线程池
CompletableFuture.supplyAsync(() -> httpClient.get(url), ioExecutor);
```

### 坑2：thenApply 和 thenApplyAsync 的区别

```java
// thenApply：谁完成了任务，谁执行回调（可能是主线程！）
// thenApplyAsync：总在新线程（或线程池）执行回调
// 回调有阻塞操作时，务必用 Async 变体
```

### 坑3：join() vs get() 的区别

```java
future.get();   // 抛出 InterruptedException + ExecutionException（受检）
future.join();  // 抛出 CompletionException（非受检），更适合链式调用
```

### 坑4：链中异常被吞掉

```java
// 没有任何异常处理，异常会被静默吞掉
CompletableFuture.supplyAsync(() -> riskyOperation())
    .thenApply(r -> process(r));
// 正确：总是加上 exceptionally 或 handle
```

### 坑5：allOf 结果获取

```java
// allOf 返回 CF<Void>，不直接包含各子任务结果
// 需在 thenApply 中手动 join 各子任务
CompletableFuture.allOf(f1, f2).thenApply(v -> f1.join() + f2.join());
```

---

## 九、底层原理简析

```
CompletableFuture 内部维护：
├── result：volatile Object，存储结果或 AltResult（封装异常）
├── stack：Completion 链表头，存储待触发的回调链
└── 完成时：CAS 设置 result → 遍历 stack → 依次触发所有注册的回调
```

**完成传播机制：**
1. 调用 `complete(value)` 或 `completeExceptionally(ex)`
2. CAS 将 result 从 null 设置为结果值
3. `postComplete()` 遍历回调栈，逐个触发后续 Completion
4. 形成链式传播，直到链尾

---

## 十、面试要点

**Q：CompletableFuture 和 Future 的区别？**
> Future 只能阻塞 get()，不支持回调和组合。CompletableFuture 支持链式回调（thenApply）、多任务组合（allOf/thenCombine）、异常处理（exceptionally/handle）、手动完成（complete）。

**Q：thenApply 和 thenApplyAsync 区别？**
> thenApply 在完成前一个任务的线程中执行回调；thenApplyAsync 在 ForkJoinPool.commonPool 或指定 executor 中执行。回调有阻塞操作时必须用 Async。

**Q：如何优雅处理 CompletableFuture 中的异常？**
> 使用 `exceptionally` 提供兜底值；使用 `handle` 统一处理成功和失败；使用 `whenComplete` 记录日志不改变结果。不处理异常会导致异常被静默吞掉。

**Q：allOf 和 anyOf 的区别？**
> allOf 等待所有任务完成，返回 CF\<Void\>；anyOf 等待最快完成的任务，返回 CF\<Object\>。注意 allOf 结果获取需手动 join 各子任务。

**Q：为什么生产环境不推荐用 commonPool？**
> commonPool 并行度 = CPU-1，适合纯 CPU 计算。IO 阻塞任务会占满线程，影响并行流等其他使用 commonPool 的组件，甚至导致整个应用响应下降。


---

**相关面试题** → [[../../10_Developlanguage/001_java/03_JavaConcurrencySubject/07、并发工具类|07、并发工具类]]
