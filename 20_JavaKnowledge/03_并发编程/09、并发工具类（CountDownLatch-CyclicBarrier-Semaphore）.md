---
tags:
  - Java/并发编程
  - Java/并发工具
aliases:
  - CountDownLatch倒计时
  - CyclicBarrier循环屏障
  - Semaphore信号量
  - Exchanger
date: 2026-03-18
---

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

## 五、三大工具类对比与选型

### 5.1 核心区别一览

| 特性 | CountDownLatch | CyclicBarrier | Semaphore |
|------|----------------|---------------|-----------|
| **核心语义** | 等待事件完成 | 等待线程集合 | 控制访问数量 |
| **等待方** | 一个或多个线程等待 | 所有参与线程互相等待 | 获取许可证的线程 |
| **触发条件** | 计数器减到 0 | 到达线程数达到阈值 | 有可用许可证 |
| **可重用性** | ❌ 一次性 | ✅ 可循环使用 | ✅ 许可证可释放复用 |
| **典型场景** | 主线程等待子任务完成 | 多线程分阶段协作 | 限流、资源池 |
| **AQS 模式** | 共享模式 | 独占/共享模式 | 共享模式 |

### 5.2 选型决策树

```
需要协调多个线程的执行顺序？
    ├── 是 → 需要等待其他线程完成某个事件？
    │           ├── 是 → CountDownLatch（一次性等待）
    │           └── 否 → CyclicBarrier（多线程互相等待，可循环）
    └── 否 → 需要限制并发访问数量？
                ├── 是 → Semaphore（限流、资源池）
                └── 否 → 考虑其他工具（Lock、Condition、BlockingQueue）
```

---

## 六、实战场景详解

### 6.1 多阶段任务协调（CyclicBarrier）

```java
// 模拟分布式计算：分片处理 → 汇总结果 → 下一阶段
public class MultiPhaseComputation {
    private static final int SHARD_COUNT = 4;
    
    public static void main(String[] args) {
        // 第一阶段屏障：所有分片完成本地计算
        CyclicBarrier phase1Barrier = new CyclicBarrier(SHARD_COUNT, () -> {
            System.out.println("所有分片完成本地计算，开始汇总...");
        });
        
        // 第二阶段屏障：汇总完成，开始全局优化
        CyclicBarrier phase2Barrier = new CyclicBarrier(SHARD_COUNT, () -> {
            System.out.println("全局优化完成，输出最终结果...");
        });
        
        ExecutorService executor = Executors.newFixedThreadPool(SHARD_COUNT);
        
        for (int i = 0; i < SHARD_COUNT; i++) {
            final int shardId = i;
            executor.submit(() -> {
                try {
                    // 阶段1：本地计算
                    System.out.println("分片 " + shardId + " 开始本地计算");
                    Thread.sleep(RandomUtil.randomInt(100, 500));
                    phase1Barrier.await();  // 等待其他分片
                    
                    // 阶段2：全局优化
                    System.out.println("分片 " + shardId + " 参与全局优化");
                    Thread.sleep(RandomUtil.randomInt(100, 300));
                    phase2Barrier.await();
                    
                    System.out.println("分片 " + shardId + " 完成全部任务");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
        }
        
        executor.shutdown();
    }
}
```

### 6.2 服务启动依赖管理（CountDownLatch）

```java
// 微服务启动：等待所有依赖服务就绪后才对外提供服务
public class ServiceStartupManager {
    private static final List<String> DEPENDENCIES = Arrays.asList(
        "database", "redis", "kafka", "config-server"
    );
    
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch readyLatch = new CountDownLatch(DEPENDENCIES.size());
        ExecutorService executor = Executors.newFixedThreadPool(DEPENDENCIES.size());
        
        for (String dep : DEPENDENCIES) {
            executor.submit(() -> {
                try {
                    System.out.println("正在初始化: " + dep);
                    checkHealth(dep);  // 检查服务健康状态
                    System.out.println(dep + " 就绪 ✓");
                } finally {
                    readyLatch.countDown();
                }
            });
        }
        
        // 主线程等待所有依赖就绪
        readyLatch.await();
        System.out.println("所有依赖就绪，服务开始对外提供服务...");
        
        // 启动 HTTP 服务...
        startHttpServer();
        executor.shutdown();
    }
    
    private static void checkHealth(String service) throws InterruptedException {
        // 模拟健康检查
        Thread.sleep(RandomUtil.randomInt(500, 2000));
    }
}
```

### 6.3 数据库连接池实现（Semaphore）

```java
// 简化的数据库连接池
public class SimpleConnectionPool {
    private final Semaphore permits;
    private final BlockingQueue<Connection> pool;
    private final int maxConnections;
    
    public SimpleConnectionPool(int maxConnections) throws SQLException {
        this.maxConnections = maxConnections;
        this.permits = new Semaphore(maxConnections);
        this.pool = new LinkedBlockingQueue<>(maxConnections);
        
        // 初始化连接
        for (int i = 0; i < maxConnections; i++) {
            pool.add(createNewConnection());
        }
    }
    
    public Connection acquireConnection(long timeout, TimeUnit unit) 
            throws InterruptedException, SQLException {
        // 获取许可证（限流）
        if (!permits.tryAcquire(timeout, unit)) {
            throw new SQLException("获取连接超时，当前活跃连接数: " + 
                (maxConnections - permits.availablePermits()));
        }
        
        Connection conn = pool.poll();
        if (conn == null || conn.isClosed()) {
            conn = createNewConnection();
        }
        
        return new PooledConnection(conn, this);
    }
    
    void releaseConnection(Connection conn) {
        if (conn != null) {
            pool.offer(conn);
            permits.release();  // 释放许可证
        }
    }
    
    public int getActiveConnections() {
        return maxConnections - permits.availablePermits();
    }
}

// 包装类，确保连接归还
class PooledConnection implements Connection {
    private final Connection realConnection;
    private final SimpleConnectionPool pool;
    
    @Override
    public void close() throws SQLException {
        pool.releaseConnection(realConnection);  // 归还到连接池
    }
    
    // 其他方法委托给 realConnection...
}
```

---

## 七、面试要点速查

| 问题 | 要点 |
|------|------|
| CountDownLatch 的作用 | 一个或多个线程等待其他线程完成某些操作（倒计时到0才放行）|
| CyclicBarrier 的作用 | 一组线程互相等待，全部到达屏障才一起继续，可重用 |
| Semaphore 的作用 | 控制并发访问数量（许可证机制），用于限流和资源池 |
| CountDownLatch 和 CyclicBarrier 的区别 | CDL 等事件（一次性）；CB 等线程互相到达（可循环），CB 可有屏障动作 |
| Semaphore 是公平的吗 | 默认非公平，可以通过构造器参数设为公平 |
| Semaphore 释放许可证的注意事项 | 必须在 finally 中 release，防止异常导致许可证泄漏永远无法获取 |
| CyclicBarrier 的 broken 状态 | 如果线程在 await() 时被中断或超时，屏障会被打破，其他线程抛出 BrokenBarrierException |
| CountDownLatch 的计数器能重置吗 | 不能，一次性使用。需要重置的场景用 CyclicBarrier 或 Phaser |
| Semaphore 的 acquire() 和 tryAcquire() 区别 | acquire() 阻塞等待；tryAcquire() 立即返回布尔值，可设置超时 |
| 三个类底层实现 | 都基于 AQS（AbstractQueuedSynchronizer），Semaphore 和 CountDownLatch 使用共享模式 |

### 进阶面试题

**Q：CountDownLatch 的计数器为什么不能重置？**
> CountDownLatch 设计为一次性使用的同步器。如果需要可重置的计数功能，应该使用 CyclicBarrier（线程间互相等待）或 Java 7 引入的 Phaser（更灵活的分阶段同步器）。CountDownLatch 的简单性保证了更高的性能和更明确的语义。

**Q：CyclicBarrier 和 CountDownLatch 在 AQS 实现上有什么区别？**
> - CountDownLatch：使用 AQS 的共享模式，countDown() 释放共享锁，await() 获取共享锁
> - CyclicBarrier：基于 ReentrantLock 和 Condition 实现，不是直接使用 AQS。每次屏障触发后会重置 generation，实现循环使用

**Q：Semaphore 的公平和非公平模式性能差异有多大？**
> 非公平模式性能通常比公平模式高 10~100 倍。公平模式需要维护 FIFO 队列，每次获取都要检查是否有等待时间更长的线程。只有在确实需要避免线程饥饿时才使用公平模式。

**Q：实际项目中如何选择这三个工具类？**
> - **CountDownLatch**：服务启动等待、并行计算结果汇总、超时控制
> - **CyclicBarrier**：多线程分阶段计算、并行迭代算法（如 PageRank）、测试并发场景
> - **Semaphore**：API 限流、数据库连接池、资源访问控制、并发度控制


---

**相关面试题** → [[../../10_Developlanguage/001_Java/03_JavaConcurrencySubject/07、并发工具类|07、并发工具类]]
