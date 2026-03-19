---
tags:
  - Java/并发编程
  - Java/锁
aliases:
  - synchronized原理
  - 偏向锁轻量级锁
  - 监视器锁
date: 2026-03-18
---

# synchronized 深度解析


> **核心关键词**：监视器锁、对象头、偏向锁、轻量级锁、重量级锁、锁升级、可重入

---

## 一、synchronized 的三种用法

```java
// 1. 修饰实例方法（锁是当前实例对象 this）
public synchronized void instanceMethod() {
    // 同一个实例的所有 synchronized 方法互斥
}

// 2. 修饰静态方法（锁是当前类的 Class 对象）
public static synchronized void staticMethod() {
    // 所有实例调用此方法都互斥（锁是 ClassName.class）
}

// 3. 修饰代码块（锁是指定对象）
public void blockMethod() {
    synchronized(this) {          // 锁 this
        // ...
    }
    synchronized(SomeClass.class) { // 锁 Class 对象
        // ...
    }
    synchronized(lockObject) {    // 锁任意对象（推荐，细粒度）
        // ...
    }
}
```

---

## 二、底层原理：对象头（Object Header）

Java 对象的内存布局：

```
┌─────────────────────────────────────────┐
│              对象头（Object Header）     │
│  ┌─────────────────────────────────┐   │
│  │   Mark Word（8字节，64位JVM）    │   │ ← 存储锁信息、GC信息等
│  ├─────────────────────────────────┤   │
│  │   Class Pointer（类型指针）      │   │ ← 指向类的元数据
│  └─────────────────────────────────┘   │
├─────────────────────────────────────────┤
│        实例数据（Instance Data）         │
├─────────────────────────────────────────┤
│        对齐填充（Padding）               │ ← 保证对象大小是8字节的倍数
└─────────────────────────────────────────┘
```

**Mark Word 的状态**（64位JVM）：

```
状态          | 锁标志 | Mark Word 内容
────────────────────────────────────────────
无锁          | 01   | 对象 hashCode(25b) + 分代年龄(4b) + 偏向锁位(0)
偏向锁         | 01   | 线程ID(54b) + 分代年龄(4b) + 偏向锁位(1)
轻量级锁（薄锁）| 00   | 指向栈中 Lock Record 的指针
重量级锁（胖锁）| 10   | 指向 Monitor 对象的指针
GC 标记        | 11   | （GC 过程中临时使用）
```

---

## 三、锁升级过程（偏向→轻量→重量）

### 3.1 偏向锁（Biased Locking）

**背景**：大多数情况下，锁只被一个线程持有，根本没有竞争。

```
第一次加锁：
  检查 Mark Word → 无锁状态
  CAS 将 Mark Word 改为：[当前线程ID + 偏向锁标志位=1]
  后续同一线程加锁：只检查线程ID，直接进入，无需 CAS

其他线程尝试加锁：
  竞争发生，偏向锁撤销（在 SafePoint 处）
  升级为轻量级锁
```

**JDK 15 默认禁用偏向锁**（`-XX:-UseBiasedLocking`），JDK 18 正式移除，因为在多线程环境下撤销偏向锁的 STW 开销反而更大。

**PDF补充：偏向锁详解**

Java偏向锁是Java6引入的一项多线程优化。顾名思义，它会偏向于第一个访问锁对象的线程，如果同步锁只有一个线程访问，则线程是不需要触发同步的，这种情况下，就会给该线程加一个偏向锁；如果在运行过程中，遇到了其他线程抢占锁，则持有偏向锁的线程会被挂起，JVM会消除它身上的偏向锁，将锁升级到轻量级锁，然后再唤醒原持有偏向锁的线程。

**偏向锁撤销的场景**：
1. 一个线程创建了大量对象并执行了初始的同步操作，之后在另一个线程中将这些对象作为锁，进行后续的操作。这种情况下，会导致大量的偏向撤销操作
2. 明知有多线程竞争，还要使用偏向锁，也会导致大量偏向撤销

**偏向锁的批量重偏向（第一阶梯底线）**：
以class为单位，为每个类维护一个偏向锁撤销计数器，每一次该类的对象发生偏向撤销操作时，该计数器+1，当这个值达到重偏向阈值（默认20）时：`BiasedLockingBulkRebiasThreshold = 20`，JVM就认为该类的偏向锁有问题，因此会进行批量重偏向。

Epoch（纪元）：每个class会有一个对应的epoch字段，每个处于偏向锁状态对象的MarkWord中也有该字段，其初始值为创建该class中的epoch的值。每次发生批量重偏向时，就将该值加1，同时遍历JVM中所有线程的栈：找到该class所有正处于加锁状态的偏向锁对象，将其epoch字段改为新值。

**偏向锁的批量撤销（第二阶梯底线）**：
当达到重偏向阈值后，假设该class计数器继续增长，当其达到批量撤销的阈值后（默认40）时：`BiasedLockingBulkRevokeThreshold = 40`，JVM就认为该class的使用场景存在较多并发锁竞争，会标记该class为不可偏向。之后对于该class的锁，直接走轻量级锁的逻辑，这就是第二阶梯底线。

**HashCode哪去了？**
当一个对象计算过一致性hash后，就再也无法进入偏向锁状态了。而当一个对象当前正处于偏向锁状态，又收到需要计算其一致性哈希码请求时，它的偏向状态会被立即撤销，并且锁会膨胀为重量级锁。

**结论**：即便初始化为可偏向状态的对象，一旦调用 `Object.hashCode()` 或者 `System.identityHashCode(Object)`，进入同步块就会直接使用轻量级锁。

### 3.2 轻量级锁（Lightweight Lock）

**适用场景**：有竞争，但竞争不激烈（交替执行）

```
加锁过程：
1. 在当前线程的栈帧中创建 Lock Record（包含 Mark Word 副本 + owner 指针）
2. CAS 将对象 Mark Word 改为指向 Lock Record 的指针
3. CAS 成功 → 获取锁
4. CAS 失败（其他线程也在抢）→ 自旋重试
5. 自旋超过阈值（默认10次）或有更多线程参与 → 升级重量级锁

解锁过程：
1. CAS 将 Mark Word 从 Lock Record 指针还原为原始 Mark Word
2. CAS 成功 → 解锁完成
3. CAS 失败（Mark Word 已被修改为重量级锁）→ 唤醒等待线程
```

**PDF补充：轻量级锁加锁过程详解**

轻量级锁是由偏向锁升级来的，偏向锁运行在一个线程进入同步块的情况下，当第二个线程加入锁争用的时候，偏向锁就会升级为轻量级锁。

轻量级锁的加锁过程：
1. 在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为"01"状态，是否为偏向锁为"0"），虚拟机首先在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的markword的拷贝，官方称之为 Displaced Mark Word
2. 拷贝对象头中的markword复制到锁记录中
3. 拷贝成功后，虚拟机将使用CAS操作尝试将对象的markword更新为指向Lock Record的指针，并将Lock Record里的owner指针指向对象的mark word
4. 如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，将对象markword的锁标志位设置为"00"，即表示此对象处于轻量级锁定状态
5. 如果这个更新操作失败了，则说明多个线程竞争锁，轻量级锁就要膨胀为重量级锁，锁标志的状态值变为"10"，markword中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。而当前线程便尝试使用自旋来获取锁

### 3.3 重量级锁（Heavyweight Lock）

**适用场景**：竞争激烈，线程需要阻塞等待

```
基于操作系统的互斥量（Mutex）实现：
  每个对象关联一个 Monitor（监视器）对象

Monitor 结构：
  ├── _owner：当前持有锁的线程
  ├── _count：重入次数
  ├── _EntryList：阻塞等待锁的线程队列
  ├── _WaitSet：调用 wait() 后等待的线程队列
  └── _recursions：锁重入计数
```

**升级不可逆**：锁只能升级，不能降级（JDK 有条件降级的实验性功能）。

**PDF补充：为什么重量级锁耗费性能资源？**

Java的线程的阻塞或唤醒是映射到操作系统原生线程之上的，是需要在用户态与核心态之间切换，这种切换会消耗大量的系统资源，因为用户态与内核态都有各自专用的内存空间，专门的寄存器等，用户态切换至内核态需要传递许多变量、参数给内核，内核也要保存好用户态在切换时的一些寄存器值、变量等，以便内核态调用结束后可以顺利切换回用户态继续工作。

如果线程状态切换是一个高频操作时，这将会消耗很多CPU处理时间；如果对于那些需要同步的简单的代码块，获取锁挂起操作消耗的时间甚至要比用户代码执行的时间还要长，这种同步策略显然是非常糟糕的。

synchronized会导致竞争不到锁的线程进入到阻塞状态，所以说它是java语言中一个重量级的同步操作，被称为重量级锁。为了缓解上述性能问题，JVM从1.5开始，引入了轻量锁与偏向锁，默认启用了自旋锁，他们都属于乐观锁。

有四种锁类型：偏向锁、轻量级锁、自旋锁、重量级锁。

加锁流程顺序：偏向锁加锁 → 轻量级锁 → 自旋锁 → 重量级锁

---

## 四、字节码层面

```java
// 源码
synchronized(obj) {
    doSomething();
}
```

```
// 字节码（javap -c 反编译）
monitorenter    // 获取监视器锁（对应 Mark Word 操作）
  doSomething   // 被保护的代码
monitorexit     // 释放监视器锁（正常退出）
// 编译器自动插入异常处理：
monitorexit     // 释放监视器锁（异常退出，保证锁一定被释放）
```

---

## 五、JDK 6+ 的四项关键优化

### 5.1 锁消除（Lock Elimination）

```java
// JIT 编译器通过逃逸分析，发现锁对象不会逃逸出当前线程
// → 直接消除无竞争的 synchronized

// 看似有锁，实则被 JIT 消除
public String concat(String s1, String s2, String s3) {
    // StringBuffer 的 append 是 synchronized 的
    // 但 sb 是局部变量，不会逃逸 → JIT 消除 synchronized
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    sb.append(s3);
    return sb.toString();
}

// 验证：-XX:+EliminateLocks（默认开启）
// -XX:+PrintEliminateLocks 打印消除日志
```

### 5.2 锁粗化（Lock Coarsening）

```java
// 如果对同一对象反复加锁解锁，JIT 会将多次加锁合并为一次
// 避免频繁加锁解锁的开销

// ❌ 优化前（三次加锁解锁）
synchronized(obj) { op1(); }
synchronized(obj) { op2(); }
synchronized(obj) { op3(); }

// ✅ JIT 粗化后（一次加锁解锁）
synchronized(obj) {
    op1();
    op2();
    op3();
}

// 常见场景：循环内部的 synchronized
for (int i = 0; i < 100; i++) {
    synchronized(obj) { process(i); }  // ← JIT 会将锁提到循环外
}
// 等效：
synchronized(obj) {
    for (int i = 0; i < 100; i++) { process(i); }
}
```

### 5.3 自适应自旋（Adaptive Spinning）

```
轻量级锁 CAS 失败后，不立即挂起线程，而是先自旋等待：

JDK 6+ 自适应自旋策略：
  - 如果上次自旋成功：增加自旋次数（乐观：可能很快释放锁）
  - 如果上次自旋失败：减少甚至跳过自旋（悲观：锁可能长期被持有）
  - 自旋次数由 JVM 根据历史记录动态调整（非固定的 10 次）

参数（了解即可）：
  -XX:PreBlockSpin=10       # JDK 5 的固定自旋次数（JDK 6 已被自适应替代）
  -XX:+UseSpinning          # JDK 6 默认开启
```

### 5.4 偏向锁的 epoch 机制

```
批量重偏向（Bulk Rebiasing）：
  当某个类的对象被多个线程竞争偏向锁时，偏向锁撤销次数超过阈值（20）
  JVM 将该类的 epoch+1，使所有该类对象的偏向锁失效
  后续再次偏向新线程时，使用新 epoch（无需逐个撤销）

批量撤销（Bulk Revocation）：
  撤销次数超过 40 次时，JVM 判定该类不适合偏向锁
  直接禁用该类的偏向锁
```

---

## 六、synchronized 的可重入性

同一线程可以多次获取同一个锁，防止死锁：

```java
class ReentrantDemo {
    synchronized void methodA() {
        System.out.println("methodA");
        methodB();  // 同一线程，可以再次获取同一个锁
    }
    
    synchronized void methodB() {
        System.out.println("methodB");
    }
}

// 子类调用父类的同步方法
class Child extends Parent {
    @Override
    synchronized void method() {
        super.method();  // ✅ 可以，锁的是同一个对象
    }
}
```

**实现原理**：Monitor 中维护 `_recursions`（重入计数），每次进入 +1，退出 -1，为 0 时才真正释放锁。

---

## 七、synchronized 与 wait/notify 协作模式

```java
// 等待/通知模式（生产者-消费者）
public class WaitNotifyDemo {
    private final Object lock = new Object();
    private volatile boolean dataReady = false;
    private String data;

    // 消费者线程
    public void consumer() throws InterruptedException {
        synchronized (lock) {
            while (!dataReady) {   // ⚠️ 必须用 while，不能用 if（防止虚假唤醒）
                lock.wait();       // 释放锁，进入 WaitSet 等待
            }
            process(data);
            dataReady = false;
            lock.notifyAll();      // 通知所有等待的生产者
        }
    }

    // 生产者线程
    public void producer(String newData) throws InterruptedException {
        synchronized (lock) {
            while (dataReady) {    // 数据未被消费，生产者等待
                lock.wait();
            }
            data = newData;
            dataReady = true;
            lock.notifyAll();      // 通知消费者
        }
    }
}

// ⚠️ wait 的三个必须：
// 1. 必须在 synchronized 块内调用
// 2. 必须用 while 循环（不能用 if）→ 防止虚假唤醒（spurious wakeup）
// 3. 必须持有 lock 对象的监视器锁（不能 wait 其他对象的锁）
```

---

## 八、synchronized vs ReentrantLock

| 特性 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 来源 | JVM 原生关键字 | JDK 提供的类 |
| 锁获取 | 隐式，进入同步块自动获取 | 显式：`lock.lock()` |
| 锁释放 | 隐式，退出同步块自动释放 | **必须手动** `lock.unlock()`（放 finally）|
| 可中断等待 | 不支持 | `lockInterruptibly()` |
| 超时获取 | 不支持 | `tryLock(time, unit)` |
| 公平锁 | 不支持 | `new ReentrantLock(true)` |
| 多条件变量 | 只有一个 wait/notify | 支持多个 `Condition` |
| 性能 | JDK 6 后与 ReentrantLock 相当 | JDK 6 后与 synchronized 相当 |

**选择建议**：
- 大多数情况用 `synchronized`（更简洁，JVM 可以优化）
- 需要可中断、超时、公平锁、多条件变量时用 `ReentrantLock`

---

## 九、常见误区与陷阱

### 7.1 锁对象不能是 null

```java
Object lock = null;
synchronized(lock) { ... }  // ❌ NullPointerException
```

### 7.2 不要锁字符串字面量或 Integer 常量

```java
synchronized("lock") { ... }  // ❌ 字符串常量池中的对象可能被其他代码共用

synchronized(Integer.valueOf(i)) {  // ❌ -128~127 的 Integer 对象从缓存池返回，可能被其他地方锁住
```

### 7.3 锁粒度要合适

```java
// ❌ 锁粒度过大：整个方法加锁，影响并发度
public synchronized void processOrder(Order order) {
    // 其中只有 order.save() 需要同步
    order.validate();   // 不需要同步
    order.save();       // 需要同步（写数据库）
    order.notify();     // 不需要同步
}

// ✅ 细化锁粒度
public void processOrder(Order order) {
    order.validate();
    synchronized(this) { order.save(); }  // 只锁必要部分
    order.notify();
}
```

---

## 十、面试要点速查

| 问题 | 要点 |
|------|------|
| synchronized 锁的是什么 | 实例方法锁 this；静态方法锁 Class 对象；代码块锁指定对象 |
| 锁升级的流程 | 无锁 → 偏向锁（单线程）→ 轻量级锁（交替）→ 重量级锁（竞争激烈）|
| 偏向锁是什么 | 第一次获取锁时将线程ID记入 Mark Word，后续同线程无需 CAS |
| 轻量级锁是什么 | 在线程栈上创建 Lock Record，CAS 更新 Mark Word，竞争失败自旋 |
| 重量级锁是什么 | 基于 OS 互斥量（Monitor），竞争失败的线程进入阻塞队列 |
| synchronized 可重入原理 | Monitor 维护重入计数（_recursions），同线程每次进入 +1，退出 -1 |
| JDK 6 后 synchronized 的优化 | 偏向锁、轻量级锁（自适应自旋）、锁消除（逃逸分析）、锁粗化 |
| wait 为什么必须用 while | 防止虚假唤醒（OS 可能无条件唤醒 wait 线程）和多消费者场景下的竞争 |
| 锁消除是什么 | JIT 通过逃逸分析，发现锁对象不会逃逸，直接消除无竞争的 synchronized |
| 锁粗化是什么 | JIT 将对同一对象的多次加锁解锁合并为一次，减少加锁开销 |


---

**相关面试题** → [[../../10_Developlanguage/001_Java/03_JavaConcurrencySubject/02、synchronized 关键字|02、synchronized 关键字]]
