# 并发工具类（CountDownLatch / CyclicBarrier / Semaphore）

> **核心关键词**：倒计时门闩、循环屏障、信号量、线程协调、AQS共享模式

---

## 一、CountDownLatch（倒计时门闩）

### 1.1 核心语义

让**一组线程等待，直到计数器减到 0 才继续执行**。一次性使用，不可重置。

```
初始化 count=3：
   等待线程 ─────────────────────────────→ await()（阻塞）
   工作线程1 完成 → countDown() count=2
   工作线程2 完成 → countDown() count=1
   工作线程3 完成 → countDown() count=0 → 所有等待线程释放！
```

### 1.2 典型使用场景

```java
// 场景1：主线程等待多个子任务全部完成
CountDownLatch latch = new CountDownLatch(5);  // 5 个任务

for (int i = 0; i < 5; i++) {
    final int taskId = i;
    executor.submit(() -> {
        try {
            processTask(taskId);
        } finally {
            latch.countDown();  // 任务完成，计数 -1（放 finally 确保一定执行）
        }
    });
}

latch.await();                           // 主线程等待所有任务完成
latch.await(30, TimeUnit.SECONDS);       // 带超时的等待（推荐）
System.out.println("所有任务完成！");

// 场景2：并发启动（起跑枪）
CountDownLatch startGun = new CountDownLatch(1);  // 发令枪

for (int i = 0; i < 10; i++) {
    executor.submit(() -> {
        startGun.await();   // 所有线程就位，等待发令
        doWork();
    });
}
// 准备就绪...
startGun.countDown();  // 发令！所有线程同时开始

// 场景3：多阶段任务（先查询，再汇总）
CountDownLatch phase1 = new CountDownLatch(3);
// 3 个线程并行查询...
phase1.await();
// 汇总结果...
```

### 1.3 底层原理（基于 AQS 共享模式）

```java
// CountDownLatch 使用 AQS 的 state 作为计数器
// await() → acquireSharedInterruptibly → 如果 state != 0，加入等待队列阻塞
// countDown() → releaseShared → state - 1，state==0 时唤醒所有等待线程

protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;  // state==0 才能通过
}

protected boolean tryReleaseShared(int releases) {
    for (;;) {
        int c = getState();
        if (c == 0) return false;
        int nextc = c - 1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;  // 减到0时返回 true，触发唤醒等待线程
    }
}
```

---

## 二、CyclicBarrier（循环屏障）

### 2.1 核心语义

让**一组线程互相等待**，直到所有线程都到达屏障点，然后全部继续执行。**可以重用**（cyclic = 循环）。

```
parties=3（需要3个线程同时到达）：

线程A → 工作 → await()（等待其他线程）
线程B → 工作 → await()（等待其他线程）
线程C → 工作 → await()（第3个到达，屏障打开！）
                              ↓
         可选执行 barrierAction（汇总逻辑）
                              ↓
三个线程全部继续执行下一阶段

--- 屏障重置，可以进行第二轮 ---
```

### 2.2 典型使用场景

```java
// 多轮迭代计算（如矩阵分块并行计算）
int workers = 4;
CyclicBarrier barrier = new CyclicBarrier(workers, () -> {
    // barrierAction：所有线程到达后执行一次（由最后一个到达的线程执行）
    System.out.println("本轮计算完成，汇总结果...");
    mergeResults();
});

for (int i = 0; i < workers; i++) {
    final int workerId = i;
    executor.submit(() -> {
        for (int round = 0; round < 3; round++) {
            computePartition(workerId, round);
            
            barrier.await();  // 等待所有 worker 完成本轮计算
            // 所有 worker 到达后，barrierAction 执行，然后所有线程继续第 round+1 轮
        }
    });
}
```

### 2.3 CountDownLatch vs CyclicBarrier

| 对比项 | CountDownLatch | CyclicBarrier |
|--------|----------------|---------------|
| 等待对象 | 等待事件发生（倒计时）| 等待线程互相到达 |
| 可重用 | ❌ 一次性 | ✅ 可重置，循环使用 |
| 计数方向 | 从 N 倒计到 0 | 从 0 累计到 N |
| 屏障动作 | 无 | 可以设置 `barrierAction` |
| 实现 | AQS 共享模式 | ReentrantLock + Condition |
| 适用场景 | 主线程等子任务 / 并发起跑 | 多线程分阶段协调 |

---

## 三、Semaphore（信号量）

### 3.1 核心语义

控制**同时访问某资源的线程数量**。类似停车场有 N 个车位，进去一辆少一个，出来一辆多一个。

```java
Semaphore semaphore = new Semaphore(3);  // 3 个许可证（车位）

// 获取许可证（车辆进入）
semaphore.acquire();          // 阻塞直到获取到许可证
semaphore.acquire(2);         // 一次获取 2 个许可证
semaphore.tryAcquire();       // 非阻塞尝试，立即返回 true/false
semaphore.tryAcquire(1, TimeUnit.SECONDS);  // 超时尝试

// 释放许可证（车辆离开）
semaphore.release();          // 释放 1 个
semaphore.release(2);         // 释放 2 个
```

### 3.2 典型使用场景

```java
// 限流：控制并发访问数据库的连接数
class ConnectionPool {
    private final Semaphore semaphore;
    private final Queue<Connection> pool;
    
    ConnectionPool(int maxConnections) {
        this.semaphore = new Semaphore(maxConnections);
        this.pool = initPool(maxConnections);
    }
    
    public Connection acquire() throws InterruptedException {
        semaphore.acquire();  // 获取许可（超过限制则阻塞）
        return pool.poll();
    }
    
    public void release(Connection conn) {
        pool.offer(conn);
        semaphore.release();  // 归还许可（在 finally 中）
    }
}

// 使用
Connection conn = pool.acquire();
try {
    conn.execute(sql);
} finally {
    pool.release(conn);  // 一定要释放！
}
```

```java
// API 限流（每秒最多 100 个请求）
Semaphore rateLimiter = new Semaphore(100);

// 每隔 1 秒重置许可证数量
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
scheduler.scheduleAtFixedRate(() -> {
    // 释放所有许可证（重置到100）
    int needed = 100 - rateLimiter.availablePermits();
    if (needed > 0) rateLimiter.release(needed);
}, 1, 1, TimeUnit.SECONDS);

// 请求处理
boolean permitted = rateLimiter.tryAcquire();
if (!permitted) {
    return Response.error("请求过于频繁");
}
handleRequest();
```

### 3.3 公平 vs 非公平

```java
// 默认非公平（性能更好，但可能有线程饥饿）
Semaphore unfair = new Semaphore(3);

// 公平模式（FIFO，保证等待时间长的优先）
Semaphore fair = new Semaphore(3, true);
```

---

## 四、Exchanger（数据交换）

两个线程在同步点交换数据（不常用但值得了解）：

```java
Exchanger<String> exchanger = new Exchanger<>();

// 线程A（生产者）
String result = exchanger.exchange("数据from A");
// 返回值是线程B 传来的数据

// 线程B（消费者）
String data = exchanger.exchange("数据from B");
// 返回值是线程A 传来的数据

// 两个线程都调用 exchange() 后，互换数据并继续执行
// 如果只有一个线程调用 exchange()，它会一直阻塞等待另一个线程
```

---

## 五、面试要点速查

| 问题 | 要点 |
|------|------|
| CountDownLatch 的作用 | 一个或多个线程等待其他线程完成某些操作（倒计时到0才放行）|
| CyclicBarrier 的作用 | 一组线程互相等待，全部到达屏障才一起继续，可重用 |
| Semaphore 的作用 | 控制并发访问数量（许可证机制），用于限流和资源池 |
| CountDownLatch 和 CyclicBarrier 的区别 | CDL 等事件（一次性）；CB 等线程互相到达（可循环），CB 可有屏障动作 |
| Semaphore 是公平的吗 | 默认非公平，可以通过构造器参数设为公平 |
| Semaphore 释放许可证的注意事项 | 必须在 finally 中 release，防止异常导致许可证泄漏永远无法获取 |


---

**相关面试题** → [[../../10_Developlanguage/001_java/03_JavaConcurrencySubject/07、并发工具类|07、并发工具类]]
