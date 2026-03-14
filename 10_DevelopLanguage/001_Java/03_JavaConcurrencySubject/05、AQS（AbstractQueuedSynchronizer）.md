###### 1. 什么是 AQS？它的作用是什么？
[[../../../20_JavaKnowledge/03_并发编程/06、AQS框架深度解析#一、AQS 概述|📖]]
AQS（`AbstractQueuedSynchronizer`）是 JDK 并发包里的一个**底层同步框架**，可以把它理解为构建锁和同步工具的"脚手架"。`ReentrantLock`、`CountDownLatch`、`Semaphore` 等都是基于 AQS 实现的。

它的价值在于把**线程排队、阻塞、唤醒**这套复杂的底层机制都封装好了，开发者只需要继承 AQS，实现几个关于资源状态（`state`）怎么获取和释放的方法，就能快速构建自定义的同步组件，不需要从头处理这些底层细节。

###### 2. AQS 的核心思想是什么？
[[../../../20_JavaKnowledge/03_并发编程/06、AQS框架深度解析#二、核心数据结构|📖]]
核心思想可以用一句话概括：**用一个 `volatile int state` 表示同步状态，用一个 FIFO 双向链表队列管理所有等待的线程。**

线程来了先抢 `state`，抢到了就是拿到锁了，干活去；抢不到就封装成 Node 节点排进队列，然后被挂起（`LockSupport.park`）。持有锁的线程释放时，唤醒队列里的下一个，让它重新去抢。

###### 3. AQS 的数据结构是怎样的？
[[../../../20_JavaKnowledge/03_并发编程/06、AQS框架深度解析#三、独占模式（acquire/release）|📖]]
两个核心结构：

**同步状态 `state`**：用 `volatile` 修饰的 `int` 变量，是整个 AQS 的"核心资源"。不同的同步组件对它的定义不同：
- `ReentrantLock` 中，`state` 表示锁的重入次数（0 表示无锁）
- `Semaphore` 中，`state` 表示剩余许可证数量
- `CountDownLatch` 中，`state` 表示倒计时剩余值

**同步队列**：一个基于 CLH 锁思想改造的**双向链表**，每个等待线程被包装为一个 `Node` 节点，节点里有线程引用、等待状态（`waitStatus`）、前驱和后继指针。用双向链表是因为取消节点时需要快速找到前驱和后继。

###### 4. AQS 的独占模式和共享模式有什么区别？
[[../../../20_JavaKnowledge/03_并发编程/06、AQS框架深度解析#四、共享模式（acquireShared）|📖]]
**独占模式**：同一时刻只允许一个线程持有资源，比如 `ReentrantLock` 的写锁。对应的核心方法是 `acquire()`/`release()`，子类要实现 `tryAcquire()`/`tryRelease()`。

**共享模式**：同一时刻可以有多个线程同时持有资源，比如 `Semaphore` 的许可、`CountDownLatch` 的读等待。对应的核心方法是 `acquireShared()`/`releaseShared()`，子类要实现 `tryAcquireShared()`/`tryReleaseShared()`。

两种模式的等待队列是同一个，区别在于节点被唤醒后的传播行为：共享模式下，一个节点被唤醒后会继续尝试唤醒后面的共享节点（"传播"效果），独占模式下只唤醒下一个节点。

###### 5. AQS 的 state 变量如何操作？
[[../../../20_JavaKnowledge/03_并发编程/06、AQS框架深度解析#一、AQS 概述|📖]]
AQS 提供了三个 `final` 方法操作 `state`，保证可见性和原子性：
- `getState()`：读取当前状态
- `setState(int newState)`：设置状态（适用于单线程环境）
- `compareAndSetState(int expect, int update)`：CAS 原子更新，这是最核心的操作

子类通过 CAS 操作 `state` 来实现同步语义，比如一个简单的互斥锁：`tryAcquire` 用 CAS 把 state 从 0 改为 1，`tryRelease` 把 state 从 1 改回 0。

###### 6. AQS 的等待队列入队和出队流程是怎样的？
[[../../../20_JavaKnowledge/03_并发编程/06、AQS框架深度解析#三、独占模式（acquire/release）|📖]]
**入队**：获取 `state` 失败后，线程被封装成 Node 节点，通过**自旋 + CAS** 安全地加到队列尾部（处理并发入队）。然后判断如果前驱节点是头节点，再试一次获取锁；否则设置前驱节点状态为 `SIGNAL`，然后调用 `LockSupport.park()` 挂起自己。

**出队**：持锁线程调用 release() 后，找到队列头节点的后继，调用 `LockSupport.unpark()` 唤醒它。被唤醒的线程再次尝试获取 `state`，成功则把自己设为新的头节点，原头节点的引用断掉，方便 GC 回收。

###### 7. AQS 如何实现公平锁和非公平锁？
[[../../../20_JavaKnowledge/03_并发编程/05、CAS与原子类#一、CAS 原理|📖]]
区别只在 `tryAcquire()` 方法里：

**非公平锁**（默认）：线程来了直接 CAS 抢 `state`，不管队列里有没有人在等。能抢到就抢，抢不到再排队。吞吐量更高，因为减少了线程挂起和唤醒的开销，但队列里的线程可能被"插队"饿死。

**公平锁**：线程来了先调用 `hasQueuedPredecessors()` 看看队列里有没有更早的线程在等。有的话老老实实排队，没有才去抢。保证了先到先得，但吞吐量相对低一些。

实现上的差异非常小，公平锁的 `tryAcquire` 只是多了 `!hasQueuedPredecessors()` 这一个条件判断。

###### 8. 基于 AQS 实现的同步组件有哪些？
[[../../../20_JavaKnowledge/03_并发编程/06、AQS框架深度解析#五、Condition 实现原理|📖]]
JUC 里很多并发工具都是基于 AQS 的：

**独占模式**：`ReentrantLock`（可重入互斥锁）、`ReentrantReadWriteLock` 的写锁。

**共享模式**：`Semaphore`（信号量，控制并发数）、`CountDownLatch`（倒计时，等待一组操作完成）、`ReentrantReadWriteLock` 的读锁。

另外 `ThreadPoolExecutor` 的内部类 `Worker` 也继承了 AQS，用来维护线程的运行状态（通过 state 标识线程是否在执行任务，防止被中断）。

###### 9. 如何基于 AQS 实现一个简单的独占锁？
[[../../../20_JavaKnowledge/03_并发编程/06、AQS框架深度解析#一、AQS 概述|📖]]
继承 AQS，只需要实现 `tryAcquire` 和 `tryRelease` 两个方法：

```java
public class SimpleLock {
    private final Sync sync = new Sync();

    static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            // state=0 表示无锁，CAS 改为 1 表示上锁
            return compareAndSetState(0, 1);
        }

        @Override
        protected boolean tryRelease(int arg) {
            setState(0); // 释放锁
            return true;
        }
    }

    public void lock()   { sync.acquire(1); }
    public void unlock() { sync.release(1); }
}
```
这个例子里没有处理重入，仅演示 AQS 的核心用法。
