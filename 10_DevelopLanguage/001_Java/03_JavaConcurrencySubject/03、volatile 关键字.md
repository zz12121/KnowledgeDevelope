###### 1. 说说 volatile 关键字的理解？
[[../../../20_JavaKnowledge/03_并发编程/04、volatile深度解析#一、volatile 是什么|📖]]
`volatile` 是 Java 提供的**轻量级同步机制**，主要解决两个问题：**变量修改的可见性** 和 **防止指令重排序**。

用 `volatile` 修饰的变量，所有线程每次读取都从主内存读最新值，每次写入都立刻刷回主内存，不会在线程私有的缓存中"藏着"。这解决了多核 CPU 因各自缓存不同步导致的数据不一致问题。

需要特别注意的是，`volatile` **不保证原子性**，所以不是万能的同步工具。

###### 2. volatile 为什么能保证可见性？根本原因是什么？
[[../../../20_JavaKnowledge/03_并发编程/04、volatile深度解析#二、可见性实现原理|📖]]
根本原因在于**现代 CPU 多级缓存架构**。每个 CPU 核心都有自己的 L1/L2 缓存，线程修改了缓存里的变量副本，可能不会立刻同步给其他核心，这就是可见性问题的根源。

`volatile` 通过两个机制来解决这个问题：

一是**强制主内存读写**：写 `volatile` 变量时，JVM 会加 `lock` 前缀指令，强制把新值刷回主内存；读 `volatile` 变量时，线程会让自己缓存里对应的缓存行**失效**，强制重新从主内存加载。

二是**硬件层面的 MESI 协议**：当某个 CPU 核心修改了 `volatile` 变量，其他核心会通过总线嗅探感知到，并把自己缓存中对应的数据标记为无效状态，下次读取时强制去主内存拿最新值。

###### 3. volatile 如何禁止指令重排序？
[[../../../20_JavaKnowledge/03_并发编程/04、volatile深度解析#三、禁止指令重排序|📖]]
通过**内存屏障**（Memory Barrier）来实现。内存屏障是一组 CPU 指令，作用是"划界线"，阻止屏障前后的指令越过它重排序。

JMM 对 `volatile` 的读写操作插入了四种屏障：
- volatile **写前**加 StoreStore 屏障：确保之前的普通写操作先完成
- volatile **写后**加 StoreLoad 屏障：确保写入结果对其他线程立即可见（这是最重的一个屏障）
- volatile **读后**加 LoadLoad 屏障：确保先读完 volatile 变量，再读后面的变量
- volatile **读后**加 LoadStore 屏障：确保先读完 volatile 变量，再执行后面的写操作

###### 4. volatile 能保证原子性吗？为什么？
[[../../../20_JavaKnowledge/03_并发编程/04、volatile深度解析#四、不保证原子性|📖]]
**不能**。

`volatile` 只保证单次读或单次写的可见性，但对**复合操作**无能为力。最典型的例子就是 `i++`，它看起来是一条语句，实际上分三步：读 i 的值 → 加 1 → 写回。这三步之间没有任何原子保护，多个线程可能同时读到同一个旧值，各自加 1 后写回，最终结果少了。

如果需要原子操作，应该用 `synchronized` 或者 `java.util.concurrent.atomic` 包下的原子类（比如 `AtomicInteger`）。

###### 5. volatile 的应用场景有哪些？
[[../../../20_JavaKnowledge/03_并发编程/04、volatile深度解析#五、使用场景|📖]]
**场景一：状态标志位**
用 `volatile boolean` 做线程停止的开关，一个线程改了标志位，另一个线程能立刻感知：
```java
private volatile boolean running = true;
public void run() {
    while (running) { /* 执行任务 */ }
}
public void stop() { running = false; }
```

**场景二：双重检查锁（DCL 单例）**
必须用 `volatile` 防止 `new Singleton()` 的指令重排序（对象创建分三步：分配内存→初始化→赋引用，JIT 可能重排为分配→赋引用→初始化，导致其他线程拿到未初始化完的对象）：
```java
private static volatile Singleton instance;
public static Singleton getInstance() {
    if (instance == null) {
        synchronized (Singleton.class) {
            if (instance == null) {
                instance = new Singleton(); // volatile 防重排序
            }
        }
    }
    return instance;
}
```

**场景三：读多写少的共享变量**
当写操作是**直接赋值**（不依赖当前值）、读操作远多于写操作时，可以用 `volatile` 代替锁，性能更好。

###### 6. volatile 和 synchronized 的区别是什么？
[[../../../20_JavaKnowledge/03_并发编程/04、volatile深度解析#六、volatile vs synchronized|📖]]
两者都能保证可见性和有序性，但差异很明显。

**功能上**：`volatile` 只保证可见性和有序性，不保证原子性；`synchronized` 三样都能保证。

**阻塞上**：`volatile` 完全不会阻塞线程，是无锁的；`synchronized` 在竞争时会让线程阻塞等待，有上下文切换开销。

**适用级别**：`volatile` 只能修饰变量；`synchronized` 可以修饰方法和代码块。

简单记忆：**volatile 管"看得见"，synchronized 管"做完整"。** 复合操作必须用 `synchronized`，简单的状态标志用 `volatile` 就够了。

###### 7. 说说 volatile 与 happens-before 的关系？
[[../../../20_JavaKnowledge/03_并发编程/02、Java内存模型（JMM）#四、happens-before 规则|📖]]
happens-before 是 Java 内存模型（JMM）定义的可见性规则：**如果 A happens-before B，那么 A 的操作结果对 B 一定可见。**

`volatile` 的 happens-before 规则是：**对 volatile 变量的写操作，happens-before 后续对该变量的读操作。**

更厉害的是 happens-before 的**传递性**：如果线程 A 先写了一些普通变量，然后写了 `volatile` 变量 v，线程 B 读到了 v 的最新值，那么 B 也能看到 A 写普通变量的结果。这就是为什么 `volatile` 写之前的所有操作，对 `volatile` 读之后的操作都可见。

###### 8. 什么是 CPU 缓存行伪共享？和 volatile 有什么关系？
[[../../../20_JavaKnowledge/03_并发编程/04、volatile深度解析#七、伪共享（False Sharing）|📖]]
**缓存行**是 CPU 缓存读写的最小单位，通常是 64 字节。

**伪共享**指的是：两个无关的变量 a 和 b 恰好落在同一个缓存行里，线程 A 修改 a 时，整个缓存行被标记为"已修改"，导致线程 B 的对应缓存行失效，即使 B 只关心 b，也要重新从内存加载整个缓存行。两个无关变量互相影响性能，这就是"假共享"。

解决方法：用 `@sun.misc.Contended` 注解（JDK 8+）让 JVM 自动在变量两侧填充字节，使其独占一个缓存行。`LongAdder` 的内部 Cell 数组就用了这个技术。

###### 9. DCL（双重检查锁）为什么一定要加 volatile？
[[../../../20_JavaKnowledge/03_并发编程/04、volatile深度解析#五、使用场景|📖]]
因为 `new Singleton()` 这行代码不是原子的，在字节码层面分三步：
1. 在堆上分配内存
2. 调用构造方法初始化对象
3. 把对象引用赋值给 instance 变量

JIT 或 CPU 可能把步骤 2 和 3 重排序，变成"分配→赋引用→初始化"。这样另一个线程在第一次判断 `instance != null` 时，看到了一个非 null 但还没初始化完的对象，拿去用就会出问题。

加了 `volatile` 之后，写操作前的 StoreStore 屏障确保初始化完成后才赋引用，彻底杜绝了这种重排序。
