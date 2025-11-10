### 一、线程基础

###### 1. 说说进程和线程的区别？
###### 2. 如何创建线程？有几种方式？
###### 3. 什么是线程安全？如何保证线程安全？
###### 4. 什么是线程的生命周期？线程有哪些状态？
###### 5. start() 和 run() 方法有什么区别？
###### 6. sleep() 和 wait() 方法有什么区别？
###### 7. yield() 和 join() 方法的作用是什么？
###### 8. 什么是守护线程？如何创建守护线程？

### 二、synchronized 关键字

###### 1. 说说 synchronized 关键字的理解？
###### 2. synchronized 的底层原理是什么？
###### 3. synchronized 可以修饰哪些地方？有什么区别？
###### 4. synchronized 的锁升级过程是怎样的？
###### 5. 什么是偏向锁、轻量级锁、重量级锁？
###### 6. synchronized 和 Lock 的区别是什么？
###### 7. synchronized 和 volatile 的区别是什么？
###### 8. synchronized 的可见性和有序性如何保证？
###### 9. synchronized 的锁优化有哪些？
###### 10. synchronized 能防止指令重排序吗？

### 三、volatile 关键字

###### 1. 说说 volatile 关键字的理解？
###### 2. volatile 如何保证可见性？
###### 3. volatile 如何禁止指令重排序？
###### 4. volatile 能保证原子性吗？为什么？
###### 5. volatile 的应用场景有哪些？
###### 6. volatile 和 synchronized 的区别是什么？

### 四、Lock 接口与实现

###### 1. Lock 接口有哪些实现类？
###### 2. ReentrantLock 的实现原理是什么？
###### 3. ReentrantLock 的公平锁和非公平锁有什么区别？
###### 4. ReentrantLock 如何实现可重入？
###### 5. ReentrantLock 和 synchronized 的区别是什么？
###### 6. 什么是读写锁？ReadWriteLock 的使用场景是什么？
###### 7. ReentrantReadWriteLock 的实现原理是什么？
###### 8. 什么是锁降级？为什么不支持锁升级？
###### 9. StampedLock 是什么？与 ReentrantReadWriteLock 有什么区别？
###### 10. Lock 的 tryLock() 方法有什么用？
###### 11. Condition 接口的作用是什么？如何使用？
###### 12. LockSupport 的 park() 和 unpark() 方法是什么？

### 五、AQS（AbstractQueuedSynchronizer）

###### 1. 什么是 AQS？它的作用是什么？
###### 2. AQS 的核心思想是什么？
###### 3. AQS 的数据结构是怎样的？
###### 4. AQS 的独占模式和共享模式有什么区别？
###### 5. AQS 中的 state 变量是什么？如何使用？
###### 6. AQS 的等待队列是如何实现的？
###### 7. AQS 如何实现公平锁和非公平锁？
###### 8. 基于 AQS 实现的同步组件有哪些？

### 六、线程池

###### 1. 什么是线程池？为什么要使用线程池？
###### 2. 线程池的核心参数有哪些？各有什么作用？
###### 3. 线程池的工作流程是怎样的？
###### 4. 线程池有哪些拒绝策略？
###### 5. 线程池有哪些常见的类型？
###### 6. 如何合理配置线程池的大小？
###### 7. ThreadPoolExecutor 的构造函数参数有哪些？
###### 8. execute() 和 submit() 方法有什么区别？
###### 9. 线程池如何优雅关闭？shutdown() 和 shutdownNow() 的区别？
###### 10. 什么是 ForkJoinPool？它的工作窃取算法是什么？
###### 11. ScheduledThreadPoolExecutor 的实现原理是什么？
###### 12. 为什么不建议使用 Executors 创建线程池？

### 七、并发工具类

###### 1. CountDownLatch 的作用和使用场景是什么？
###### 2. CyclicBarrier 的作用和使用场景是什么？
###### 3. CountDownLatch 和 CyclicBarrier 的区别是什么？
###### 4. Semaphore 的作用和使用场景是什么？
###### 5. Exchanger 的作用和使用场景是什么？
###### 6. Phaser 是什么？与 CyclicBarrier 有什么区别？
###### 7. CompletableFuture 的作用是什么？如何使用？
###### 8. Future 和 CompletableFuture 的区别是什么？
###### 9. 如何实现异步任务的链式调用？
###### 10. CompletableFuture 的异常处理如何实现？

### 八、并发集合

###### 1. ConcurrentHashMap 的实现原理是什么？
###### 2. ConcurrentHashMap 在 JDK 1.7 和 1.8 中的区别是什么？
###### 3. ConcurrentHashMap 如何保证线程安全？
###### 4. ConcurrentHashMap 的 put 操作流程是怎样的？
###### 5. ConcurrentHashMap 能保证复合操作的原子性吗？
###### 6. CopyOnWriteArrayList 的实现原理是什么？
###### 7. CopyOnWriteArrayList 的应用场景是什么？
###### 8. BlockingQueue 有哪些实现类？它们的区别是什么？
###### 9. ArrayBlockingQueue 和 LinkedBlockingQueue 的区别是什么？
###### 10. ConcurrentLinkedQueue 的实现原理是什么？

### 九、原子类

###### 1. 什么是原子类？Java 中有哪些原子类？
###### 2. AtomicInteger 的实现原理是什么？
###### 3. CAS 算法是什么？它的优缺点是什么？
###### 4. CAS 的 ABA 问题是什么？如何解决？
###### 5. AtomicStampedReference 和 AtomicMarkableReference 的区别是什么？
###### 6. LongAdder 和 AtomicLong 的区别是什么？
###### 7. LongAdder 的实现原理是什么？
###### 8. 原子类的应用场景有哪些？

### 十、线程通信与协作

###### 1. 线程间通信的方式有哪些？
###### 2. wait()、notify() 和 notifyAll() 的使用方法是什么？
###### 3. 为什么 wait() 和 notify() 必须在同步代码块中调用？
###### 4. notify() 和 notifyAll() 的区别是什么？
###### 5. 如何实现生产者-消费者模式？
###### 6. ThreadLocal 的作用和实现原理是什么？

### 十一、内存模型与可见性

###### 1. 什么是 Java 内存模型（JMM）？
###### 2. JMM 中的主内存和工作内存是什么？
###### 3. 什么是内存可见性问题？如何解决？
###### 4. 什么是指令重排序？JMM 如何禁止指令重排序？
###### 5. 什么是 happens-before 原则？
###### 6. final 关键字的内存语义是什么？
###### 7. 什么是伪共享？如何避免？
###### 8. CPU 缓存一致性协议是什么？

### 十二、死锁与性能优化

###### 1. 什么是死锁？死锁产生的条件是什么？
###### 2. 如何避免死锁？
###### 3. 如何检测和定位死锁？
###### 4. 什么是活锁？与死锁有什么区别？
###### 5. 什么是饥饿？如何避免线程饥饿？
###### 6. 如何提高并发程序的性能？
###### 7. 什么是无锁编程？有哪些应用场景？
###### 8. 如何使用 JMH 进行并发性能测试？