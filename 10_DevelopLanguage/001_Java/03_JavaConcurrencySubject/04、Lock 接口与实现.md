###### 1. Lock 接口有哪些主要实现类？各有什么特点？
[[../../../20_JavaKnowledge/03_并发编程/07、ReentrantLock与读写锁#一、Lock 接口|📖]]
`Lock` 接口的三个主要实现：

**`ReentrantLock`**（最常用）：可重入的互斥锁，是 `synchronized` 的增强版。支持公平/非公平策略，支持可中断获取锁、超时获取锁，通过 `Condition` 可以实现多条件等待。适用于大多数需要互斥的场景。

**`ReentrantReadWriteLock`**：维护一对关联的锁——**读锁（共享）** 和 **写锁（独占）**。多个线程可以同时持有读锁，但写锁同一时间只能一个线程持有且持有时其他线程无法读。大幅提升了读多写少场景的并发性能。

**`StampedLock`**（JDK 8）：提供三种模式：写锁、悲观读锁、**乐观读**。乐观读不加锁，读完校验"邮戳"是否有效，无效再升级为悲观读。性能通常优于 `ReentrantReadWriteLock`，但不可重入，也不支持 `Condition`。

###### 2. ReentrantLock 的实现原理是什么？
[[../../../20_JavaKnowledge/03_并发编程/07、ReentrantLock与读写锁#二、ReentrantLock 深度解析|📖]]
底层基于 **AQS**（AbstractQueuedSynchronizer）。

**加锁**：用 CAS 尝试把 AQS 的 `state` 从 0 改为 1，成功了就是拿到锁了，同时记录持锁线程。如果 `state > 0` 但持锁线程是自己（可重入），就把 `state` 加 1。否则，线程进入 AQS 等待队列，被 `LockSupport.park()` 挂起。

**解锁**：`state` 减 1，减到 0 才真正释放，然后唤醒队列里的下一个等待线程。

###### 3. ReentrantLock 的公平锁和非公平锁有什么区别？
[[../../../20_JavaKnowledge/03_并发编程/07、ReentrantLock与读写锁#二、ReentrantLock 深度解析|📖]]
核心区别就一点：**新来的线程要不要排队**。

**非公平锁（默认）**：线程来了直接 CAS 抢，不管队列里有没有人等着。能抢到就不用排队。吞吐量更高，但队列里的线程可能长时间得不到执行（饥饿）。

**公平锁**：线程来了先看队列里有没有更早的等待者，有的话老实排队，没有才去抢。保证了先到先得，但每次都要检查队列，吞吐量比非公平锁低一些。

**选择建议**：绝大多数情况用非公平锁（默认就是），性能更好。只有明确需要防止线程饥饿时才考虑公平锁。

###### 4. ReentrantLock 如何实现可重入？
[[../../../20_JavaKnowledge/03_并发编程/07、ReentrantLock与读写锁#三、ReadWriteLock 与 StampedLock|📖]]
"可重入"就是同一个线程可以多次获取同一把锁而不会死锁。

实现原理：AQS 的 `state` 不只是 0 或 1，还记录**重入次数**。线程获取锁时，如果发现 `state > 0` 但持锁者就是自己，就把 `state` 加 1（第二次进入变成 2，第三次变成 3...）。解锁时每次 `state - 1`，只有减到 0 才真正释放锁，唤醒其他等待线程。

`synchronized` 也是可重入的，原理类似（Monitor 里有 `_recursions` 记录重入次数）。

###### 5. ReentrantLock 和 synchronized 的区别是什么？
[[../../../20_JavaKnowledge/03_并发编程/07、ReentrantLock与读写锁#三、ReadWriteLock 与 StampedLock|📖]]
两者都能实现互斥同步，区别主要在功能灵活性上。

**实现层次**不同：`synchronized` 是 JVM 内置关键字，自动加解锁；`ReentrantLock` 是 JDK 的 API，需要手动 `lock()` 和 `unlock()`，必须放在 `try-finally` 里确保能解锁。

**功能丰富程度**不同：`ReentrantLock` 支持**可中断获取锁**（`lockInterruptibly()`）、**超时获取锁**（`tryLock(timeout)`）、**尝试非阻塞获取**（`tryLock()`）、**公平锁**；`synchronized` 这些都不支持，一旦等锁就会一直等。

**等待机制**不同：`synchronized` 只有一个等待队列（`wait/notifyAll`）；`ReentrantLock` 可以通过 `Condition` 绑定多个等待队列，生产者和消费者可以分开等，通知时精确唤醒需要的那一方，效率更高。

**性能上**：JDK 6 以后 `synchronized` 做了很多优化（偏向锁、轻量级锁），两者差距不大。低竞争用 `synchronized` 代码更简洁，高竞争或需要高级控制用 `ReentrantLock`。

###### 6. 读写锁的使用场景和规则是什么？
[[../../../20_JavaKnowledge/03_并发编程/03、synchronized深度解析#八、synchronized vs ReentrantLock|📖]]
**读写锁适合读多写少的场景**：多个线程同时读数据没有安全问题，可以并发；但写数据时不能有任何其他读或写操作。

`ReentrantReadWriteLock` 的规则：
- **读读不互斥**：多个线程可以同时拿读锁
- **读写互斥**：有写锁时，其他线程无法获取读锁或写锁
- **写写互斥**：写锁同一时间只有一个线程能拿

典型场景：本地缓存、配置管理——读操作很频繁，偶尔有更新，用读写锁比全部 `synchronized` 并发性更好。

###### 7. ReentrantReadWriteLock 的实现原理是什么？
[[../../../20_JavaKnowledge/03_并发编程/07、ReentrantLock与读写锁#四、Condition 条件变量|📖]]
同样基于 AQS，**把 32 位的 `state` 一分为二来用**：

- **高 16 位**：读锁的持有数量（多个线程共享，所以是计数）
- **低 16 位**：写锁的重入次数（独占，所以是一个线程的重入计数）

通过位运算来判断读写状态，设计非常精巧。

###### 8. 什么是锁降级？为什么不支持锁升级？
[[../../../20_JavaKnowledge/03_并发编程/07、ReentrantLock与读写锁#二、ReentrantLock 深度解析|📖]]
**锁降级**：持有写锁的线程，先获取读锁，然后释放写锁。这样从写状态降级为读状态，在这个过程中没有其他写线程能进来，保证了数据可见性和一致性。

**为什么不支持锁升级**（持有读锁直接升为写锁）？因为极易死锁：如果线程 A 和线程 B 都持有读锁，A 想升级为写锁，需要等 B 释放读锁；B 也想升级，需要等 A 释放读锁。两个都在等对方，死锁了。所以 `ReentrantReadWriteLock` 直接不支持这种操作。

###### 9. StampedLock 和 ReentrantReadWriteLock 的区别是什么？
[[../../../20_JavaKnowledge/03_并发编程/07、ReentrantLock与读写锁#一、Lock 接口|📖]]
**最大的区别是 StampedLock 有乐观读模式**。

`ReentrantReadWriteLock` 的读锁是悲观的，读的时候也要真正加锁，写线程会被所有读线程挡住。

`StampedLock` 的乐观读**不加锁**，只获取一个"邮戳"（stamp），读完后校验邮戳是否有效（即读期间有没有写操作），如果有效就成功，没加任何锁，开销极小；如果无效，再升级为悲观读锁重读一次。

代价是：`StampedLock` **不可重入**，也不支持 `Condition`，使用起来比 `ReentrantReadWriteLock` 复杂，需要自己处理乐观读失败的情况。

###### 10. tryLock() 方法有什么用？
[[../../../20_JavaKnowledge/03_并发编程/07、ReentrantLock与读写锁#一、Lock 接口|📖]]
`tryLock()` 提供了**非阻塞**的锁获取方式：

- **`tryLock()`（无参）**：立刻尝试获取锁，拿到了返回 `true`，拿不到立刻返回 `false`，线程不会阻塞。
- **`tryLock(timeout, unit)`（带超时）**：在指定时间内尝试获取锁，超时返回 `false`。

典型用途：避免死锁（两个线程按不同顺序获取多把锁时，可以用 `tryLock` 拿不到就放弃重试）；非关键任务快速失败（拿不到锁就跳过，不影响主流程）。

###### 11. Condition 和 Object.wait/notify 有什么区别？
[[../../../20_JavaKnowledge/03_并发编程/07、ReentrantLock与读写锁#一、Lock 接口|📖]]
`Condition` 是和 `Lock` 绑定的等待通知机制，比 `synchronized` + `wait/notify` 更强大的地方在于：

**一把锁可以有多个 `Condition`**：比如生产者等 `notFull` 条件，消费者等 `notEmpty` 条件，通知时能精确唤醒对应的那批线程，而不是把所有等待线程都唤醒（`notifyAll` 的开销）。

`Object.wait()/notify()` 一个对象只有一个等待队列，`notifyAll` 可能唤醒不该唤醒的线程，再让它们因条件不满足重新等待，浪费开销。

###### 12. LockSupport 的 park() 和 unpark() 是什么？
[[../../../20_JavaKnowledge/03_并发编程/07、ReentrantLock与读写锁#一、Lock 接口|📖]]
`LockSupport` 是更底层的线程阻塞工具，AQS 底层就是用它来挂起和唤醒线程的。

- **`park()`**：阻塞当前线程
- **`unpark(Thread t)`**：唤醒指定线程

相比 `Object.wait/notify` 的优势：不需要先持有对象的锁就能调用；以线程为操作对象，更精准；**`unpark` 可以在 `park` 之前调用**（提前"充值"一个许可），这样线程调用 `park` 时发现有许可直接通过，不会错过唤醒，避免了 notify 在 wait 之前发生导致的线程永久等待问题。
