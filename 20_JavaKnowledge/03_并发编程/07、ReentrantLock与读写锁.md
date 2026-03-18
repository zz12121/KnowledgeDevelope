---
tags:
  - Java/并发编程
  - Java/锁
aliases:
  - ReentrantLock可重入锁
  - ReadWriteLock
  - StampedLock
  - 公平锁非公平锁
date: 2026-03-18
---

# ReentrantLock 与读写锁


> **核心关键词**：ReentrantLock、公平锁、非公平锁、Condition、ReadWriteLock、StampedLock、锁降级、锁超时

---

## 一、ReentrantLock 概述

`ReentrantLock` 是 `java.util.concurrent.locks` 包下基于 **AQS（AbstractQueuedSynchronizer）** 实现的可重入独占锁，提供了 `synchronized` 所不具备的高级特性。

### 1.1 ReentrantLock vs synchronized

| 特性 | synchronized | ReentrantLock |
|------|-------------|--------------|
| 锁的获取 | 隐式，进入/退出同步块 | 显式 `lock()` / `unlock()` |
| 可中断 | ❌ 不可中断 | ✅ `lockInterruptibly()` |
| 超时尝试 | ❌ 不支持 | ✅ `tryLock(timeout)` |
| 公平性 | ❌ 非公平 | ✅ 可选公平/非公平 |
| 条件变量 | 1个（wait/notify） | ✅ 多个 `Condition` |
| 锁状态查询 | ❌ 无 | ✅ `isLocked()`, `getQueueLength()` |
| 性能（JDK 6+） | 与 ReentrantLock 相当 | 相当（JDK 6 优化后） |
| 实现机制 | JVM 内置（字节码 monitorenter/monitorexit） | Java 层面（AQS） |

**选择建议**：
- 简单同步：优先 `synchronized`（代码简洁，JVM 可优化）
- 需要超时/中断/多条件/公平锁：用 `ReentrantLock`

---

## 二、ReentrantLock 核心 API

### 2.1 基础使用模板

```java
private final ReentrantLock lock = new ReentrantLock();

public void doWork() {
    lock.lock();        // 阻塞直到获得锁
    try {
        // 临界区代码
        sharedResource.modify();
    } finally {
        lock.unlock();  // 必须在 finally 中释放！否则异常时锁永久不释放
    }
}
```

> **重要**：`unlock()` 必须放在 `finally` 中。这是 ReentrantLock 与 synchronized 相比的最大使用风险。

### 2.2 可中断锁获取

```java
public void doWorkInterruptible() throws InterruptedException {
    lock.lockInterruptibly();  // 等待锁时可被 interrupt() 打断
    try {
        sharedResource.modify();
    } finally {
        lock.unlock();
    }
}

// 使用场景：任务取消、避免死锁
// 当线程A、B 互相等待对方的锁时，可通过 interrupt 打破死锁
```

### 2.3 超时尝试获取锁

```java
public boolean doWorkWithTimeout() throws InterruptedException {
    // 尝试获取锁，最多等待 500ms
    if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
        try {
            doWork();
            return true;
        } finally {
            lock.unlock();
        }
    } else {
        // 超时未获得锁，执行降级逻辑
        log.warn("获取锁超时，跳过此次操作");
        return false;
    }
}

// tryLock() 无参版本：立即返回（非阻塞）
if (lock.tryLock()) {
    try { ... } finally { lock.unlock(); }
} else {
    // 锁被占用，立即执行其他逻辑
}
```

### 2.4 可重入性

```java
// 同一线程可以多次获得同一把锁，持有计数 +1
public class ReentrantDemo {
    private final ReentrantLock lock = new ReentrantLock();

    public void outer() {
        lock.lock();  // count = 1
        try {
            inner();  // 同一线程再次获锁
        } finally {
            lock.unlock();  // count = 0，真正释放
        }
    }

    public void inner() {
        lock.lock();  // count = 2（不会死锁）
        try {
            doInnerWork();
        } finally {
            lock.unlock();  // count = 1
        }
    }
}
// getHoldCount() 查看当前线程的重入次数
System.out.println(lock.getHoldCount()); // 同步块内调用
```

---

## 三、公平锁 vs 非公平锁

```java
// 非公平锁（默认）：新来的线程可以插队尝试抢锁
ReentrantLock unfairLock = new ReentrantLock();        // 非公平
ReentrantLock fairLock   = new ReentrantLock(true);   // 公平

// 公平锁：严格按照 AQS 等待队列的 FIFO 顺序分配锁
// 非公平锁：线程获取锁时先尝试 CAS 抢锁，失败才进队列

// 性能对比（百万次操作基准测试）：
//   非公平锁：~150ms（减少了线程挂起/唤醒次数）
//   公平锁  ：~300ms（每次都需要检查队列，上下文切换多）

// 何时用公平锁：
//   - 防止线程饥饿（某些线程长期抢不到锁）
//   - 要求严格按序执行的场景
// 何时用非公平锁（大多数情况）：
//   - 追求吞吐量，允许短暂不公平
```

### 3.1 公平锁原理（AQS 层面）

```
非公平锁的 lock()：
  1. CAS 尝试将 state: 0 → 1（直接抢锁）
  2. 失败 → 进入 AQS 等待队列排队

公平锁的 lock()：
  1. 检查 AQS 队列是否有前驱节点
  2. 有前驱 → 直接进队列（不插队）
  3. 无前驱 → CAS 尝试获锁

// 关键方法：hasQueuedPredecessors()
// 公平锁在 tryAcquire 中会先调用此方法检查队列
```

---

## 四、Condition（条件变量）

`Condition` 是 `ReentrantLock` 提供的精细化等待/通知机制，对应 `synchronized` 中的 `wait/notify`，但**支持多个条件队列**。

### 4.1 基础使用

```java
private final ReentrantLock lock = new ReentrantLock();
private final Condition notFull  = lock.newCondition(); // 条件1：未满
private final Condition notEmpty = lock.newCondition(); // 条件2：非空

// 典型模式：有界缓冲区（生产者-消费者）
public class BoundedBuffer<T> {
    private final Object[] items;
    private int head, tail, count;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public BoundedBuffer(int capacity) {
        items = new Object[capacity];
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();  // 满了，等待"未满"条件
            }
            items[tail] = item;
            tail = (tail + 1) % items.length;
            count++;
            notEmpty.signal();  // 通知消费者"非空了"
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();  // 空了，等待"非空"条件
            }
            T item = (T) items[head];
            head = (head + 1) % items.length;
            count--;
            notFull.signal();   // 通知生产者"未满了"
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

### 4.2 Condition vs Object.wait/notify

| | Object.wait/notify | Condition.await/signal |
|--|---------------------|------------------------|
| 绑定锁 | 绑定 synchronized | 绑定 ReentrantLock |
| 条件数量 | 1个 | **多个** |
| 精准通知 | notifyAll 容易误唤醒 | `signal()` 精确唤醒特定条件的等待者 |
| 中断支持 | ✅ | ✅（`awaitUninterruptibly()` 不响应中断）|
| 超时等待 | `wait(ms)` | `await(time, unit)`、`awaitNanos(ns)` |

> **精准通知的威力**：`ArrayBlockingQueue` 内部就是用两个 `Condition`（`notFull` 和 `notEmpty`）避免了生产者唤醒生产者的情况（`notifyAll` 的问题）。

---

## 五、ReadWriteLock（读写锁）

### 5.1 设计思想

```
读多写少场景：
  传统独占锁：读-读互斥（浪费，并发读不需要互斥）

读写锁规则：
  读-读：✅ 可共享（多个读线程同时持有读锁）
  读-写：❌ 互斥（读时不能写，写时不能读）
  写-写：❌ 互斥（写是独占的）
```

### 5.2 ReentrantReadWriteLock

```java
private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
private final ReadWriteLock.ReadLock  readLock  = rwLock.readLock();
private final ReadWriteLock.WriteLock writeLock = rwLock.writeLock();

private Map<String, String> cache = new HashMap<>();

// 读操作（共享锁，多线程并发）
public String get(String key) {
    readLock.lock();
    try {
        return cache.get(key);
    } finally {
        readLock.unlock();
    }
}

// 写操作（独占锁，排他）
public void put(String key, String value) {
    writeLock.lock();
    try {
        cache.put(key, value);
    } finally {
        writeLock.unlock();
    }
}
```

### 5.3 锁降级（Write → Read）

```java
// 锁降级：持有写锁时，可以获取读锁，然后释放写锁
// 目的：写完数据后，继续以读锁读取刚写入的数据（保证数据一致性）

public void updateAndRead() {
    writeLock.lock();
    try {
        // 修改数据
        cache.put("key", "newValue");
        
        // 锁降级：先获取读锁（此时持有写锁，直接成功）
        readLock.lock();
    } finally {
        writeLock.unlock();  // 释放写锁，降级为读锁
    }
    
    try {
        // 用读锁读取数据（此时其他线程也可以读，但不能写）
        String value = cache.get("key");
        processValue(value);
    } finally {
        readLock.unlock();   // 最终释放读锁
    }
}

// ⚠️ 注意：不支持锁升级（读锁→写锁会死锁！）
// 多个读线程都想升级为写锁，相互等待 → 死锁
```

### 5.4 读写锁的缺陷

```
写线程饥饿问题：
  大量读线程持续持有读锁时，写线程可能长时间等待

ReentrantReadWriteLock 的写线程等待策略：
  - 非公平模式：读锁可能多次插队，写线程饥饿
  - 公平模式：按队列顺序，写线程不会饥饿
  - 读线程进入队列头是写线程时，新读线程也排队（防写线程饥饿）
```

---

## 六、StampedLock（JDK 8）

`StampedLock` 是对 `ReadWriteLock` 的改进，引入了**乐观读**，进一步提升读性能。

### 6.1 三种模式

```java
StampedLock sl = new StampedLock();

// 1. 悲观读锁（共享，与 ReadWriteLock 类似）
long stamp = sl.readLock();
try {
    read();
} finally {
    sl.unlockRead(stamp);
}

// 2. 写锁（独占）
long stamp = sl.writeLock();
try {
    write();
} finally {
    sl.unlockWrite(stamp);
}

// 3. 乐观读（核心创新！）——不阻塞写线程
long stamp = sl.tryOptimisticRead();  // 不加锁，获取一个 stamp
read();                               // 读取数据
if (!sl.validate(stamp)) {           // 验证：读期间是否有写操作
    // 验证失败：有写操作，升级为悲观读锁
    stamp = sl.readLock();
    try {
        read();  // 重新读
    } finally {
        sl.unlockRead(stamp);
    }
}
```

### 6.2 完整乐观读示例

```java
class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    public double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead();
        double curX = x, curY = y;  // 乐观读取快照
        
        if (!sl.validate(stamp)) {  // 验证是否有写操作
            stamp = sl.readLock();  // 升级悲观读
            try {
                curX = x;
                curY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.sqrt(curX * curX + curY * curY);
    }

    public void move(double deltaX, double deltaY) {
        long stamp = sl.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);
        }
    }
}
```

### 6.3 StampedLock 注意事项

```
⚠️ StampedLock 的限制：
1. 不可重入（同一线程不能重复获取同类锁）
2. 不支持 Condition
3. 不支持锁升级（读→写）
4. 线程在等待锁时被中断可能导致 CPU 飙高（busy-wait 问题）
   → 使用 readLockInterruptibly() / writeLockInterruptibly() 代替

适用场景：读多写极少的场景，对性能要求极高
不适用：需要可重入、条件变量的场景
```

---

## 七、三种锁性能对比

```
测试场景：10 个读线程 + 1 个写线程，100W 次读，1W 次写

ReentrantLock（独占）：
  读-读串行，写优先无法发挥 → 适合写多读少

ReentrantReadWriteLock：
  读并发高，写独占 → 适合读多写少（读:写 ≥ 10:1）
  注意：写线程等待期间读多可能饥饿

StampedLock（乐观读）：
  无竞争时乐观读接近零开销 → 读频率极高时最优
  写操作少时优势最大
```

---

## 八、实战最佳实践

### 8.1 缓存实现（读写锁经典场景）

```java
public class Cache<K, V> {
    private final Map<K, V> map = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

    public V get(K key) {
        rwLock.readLock().lock();
        try {
            V value = map.get(key);
            if (value != null) return value;
        } finally {
            rwLock.readLock().unlock();
        }
        
        // 缓存未命中，需要加载（写锁）
        rwLock.writeLock().lock();
        try {
            // 双重检查（防止多线程同时 miss 后重复加载）
            V value = map.get(key);
            if (value == null) {
                value = loadFromDB(key);
                map.put(key, value);
            }
            return value;
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

### 8.2 超时获取锁防止死锁

```java
// 转账场景：避免死锁的超时策略
public boolean transfer(Account from, Account to, double amount) {
    ReentrantLock fromLock = from.getLock();
    ReentrantLock toLock   = to.getLock();
    
    long timeout = 500; // ms
    try {
        if (fromLock.tryLock(timeout, TimeUnit.MILLISECONDS)) {
            try {
                if (toLock.tryLock(timeout, TimeUnit.MILLISECONDS)) {
                    try {
                        from.debit(amount);
                        to.credit(amount);
                        return true;
                    } finally {
                        toLock.unlock();
                    }
                }
            } finally {
                fromLock.unlock();
            }
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    return false;  // 超时，转账失败，可重试
}
```

---

## 九、面试要点速查

| 问题 | 要点 |
|------|------|
| ReentrantLock 比 synchronized 多了哪些功能 | 可中断、超时、公平锁、多 Condition |
| 公平锁和非公平锁的区别 | 公平按 FIFO 排队，避免饥饿但吞吐量低；非公平允许插队，吞吐量高 |
| Condition 相比 wait/notify 的优势 | 支持多个条件队列，精准唤醒特定条件的等待线程 |
| 什么是锁降级 | 持有写锁时可以获取读锁，然后释放写锁。降级目的是保证数据一致性 |
| 为什么不支持锁升级 | 多个读线程同时升级会相互等待造成死锁 |
| StampedLock 的乐观读原理 | 不加锁直接读，读后 validate() 检查是否有写操作；失败则升级悲观读 |
| ReadWriteLock 写线程饥饿如何解决 | 使用公平模式；或使用 StampedLock；或限制读锁持有时间 |
| unlock() 必须放 finally 的原因 | 防止临界区抛异常导致锁永久不释放，造成其他线程永久阻塞 |


---

**相关面试题** → [[../../10_Developlanguage/001_Java/03_JavaConcurrencySubject/04、Lock 接口与实现|04、Lock 接口与实现]]
