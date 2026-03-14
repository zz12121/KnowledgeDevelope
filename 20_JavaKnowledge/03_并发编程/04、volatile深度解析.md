# volatile 深度解析

> **核心关键词**：可见性、有序性、内存屏障、happens-before、不保证原子性、DCL、CPU缓存行、MESI协议

---

## 零、CPU 缓存架构与可见性问题根源

理解 volatile，先要理解**为什么会有可见性问题**。

```
现代 CPU 缓存架构（以 Intel 四核为例）：

核心0            核心1            核心2            核心3
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  L1 Cache│    │  L1 Cache│    │  L1 Cache│    │  L1 Cache│  (32KB, ~4ns)
│  L2 Cache│    │  L2 Cache│    │  L2 Cache│    │  L2 Cache│  (256KB, ~12ns)
└────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
     └────────────────┴─────────────┴────────────────┘
                         L3 Cache（共享，~40ns）
                              主内存（DRAM，~100ns）

问题：
  线程A（核心0）修改了变量 x=1 → 写入 L1/L2 Cache（还没刷主内存）
  线程B（核心1）读取变量 x   → 从自己的 L1/L2 Cache 读，得到旧值 x=0
  → 可见性问题！
```

### 0.1 MESI 缓存一致性协议

CPU 通过 **MESI 协议**保证多核缓存一致性：

```
MESI 四种状态：
  M（Modified）：数据只在本缓存，且已被修改（脏）
  E（Exclusive）：数据只在本缓存，未修改（干净）
  S（Shared）：数据在多个缓存中，未修改
  I（Invalid）：缓存行无效

volatile 写的工作原理：
  1. 线程A 修改 volatile 变量
  2. MESI 协议发出"Invalid"消息，使其他核心的缓存行失效（I 状态）
  3. 其他线程读取该变量时，缓存行无效 → 强制从主内存重新加载
  → 可见性保证的底层机制
```

### 0.2 Store Buffer 与 Invalidate Queue

```
硬件优化带来新问题：
  写操作先进 Store Buffer（异步刷主内存），不等待其他核心确认 Invalid
  读操作先查 Invalidate Queue（延迟处理 Invalid 消息），可能读到脏数据

volatile 的内存屏障作用：
  StoreLoad 屏障：刷空 Store Buffer，等待 Invalid 确认 → 确保写可见
  LoadLoad 屏障：刷空 Invalidate Queue，重新读 → 确保读到最新值
```

---

## 一、volatile 的两大保证

### 1.1 可见性

被 `volatile` 修饰的变量，每次写操作会**立即刷新到主内存**，每次读操作会**从主内存重新加载**，不使用线程工作内存的副本。

```java
// ❌ 没有 volatile：线程B 可能一直读到 flag=false（工作内存中的过期副本）
private boolean flag = false;

// ✅ 有 volatile：线程A 修改 flag 后，线程B 立即能看到最新值
private volatile boolean flag = false;

// 线程A
public void stop() {
    flag = true;  // volatile 写，立即刷到主内存
}

// 线程B
public void run() {
    while (!flag) {  // volatile 读，每次从主内存读
        doWork();
    }
}
```

### 1.2 有序性（禁止指令重排序）

`volatile` 通过**内存屏障**禁止相关操作的重排序：

```
volatile 写：
  StoreStore 屏障（其前的写不能排到 volatile 写后面）
  [volatile 写]
  StoreLoad 屏障（volatile 写不能排到其后的读/写前面）← 最关键

volatile 读：
  [volatile 读]
  LoadLoad 屏障（volatile 读不能排到其后的读前面）
  LoadStore 屏障（volatile 读不能排到其后的写前面）
```

---

## 二、volatile 不保证原子性

`volatile` **不能**保证复合操作（如 i++）的原子性：

```java
public class Counter {
    volatile int count = 0;
    
    public void increment() {
        count++;  // ❌ 分三步：读-加-写，不是原子的
        // 即使 count 是 volatile，多线程并发 increment() 仍然不安全
    }
}

// 10个线程各执行 1000 次 increment()，结果不是 10000！
// 因为：
// 线程A 读 count=5，线程B 读 count=5
// 线程A 写 count=6，线程B 写 count=6
// → 两次 increment 只增加了 1

// ✅ 解决方案
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();  // CAS 保证原子性

// ✅ 或者使用 synchronized
synchronized(this) { count++; }
```

---

## 三、典型使用场景

### 3.1 状态标志位（最常见）

```java
// 停止线程的标志
class Worker extends Thread {
    private volatile boolean stopped = false;
    
    @Override
    public void run() {
        while (!stopped) {
            doWork();
        }
    }
    
    public void shutdown() {
        stopped = true;
    }
}
```

### 3.2 双重检查锁单例（DCL）

```java
public class Singleton {
    // volatile 防止 instance = new Singleton() 的指令重排序
    private static volatile Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {              // 第一次检查（不加锁，高效）
            synchronized (Singleton.class) {
                if (instance == null) {      // 第二次检查（加锁后再检查）
                    instance = new Singleton(); // 三步：分配内存→初始化→赋值引用
                    // 没有 volatile：步骤可能变成：分配内存→赋值引用→初始化
                    // 另一线程第一次检查 instance != null 进入 return
                    // 但 instance 未初始化完成！→ 拿到了半初始化的对象
                }
            }
        }
        return instance;
    }
}
```

### 3.3 发布不可变对象

```java
class DataPublisher {
    private volatile Data data;  // volatile 保证对象安全发布
    
    public void publish(Data newData) {
        this.data = newData;  // volatile 写，发布后对其他线程可见
    }
    
    public Data getLatest() {
        return data;  // volatile 读，始终拿到最新发布的对象
    }
}
```

---

## 四、volatile 与 happens-before

**happens-before** 是 Java 内存模型（JMM）的核心可见性规则：如果 A happens-before B，则 A 的结果对 B 可见。

### 4.1 volatile 的 happens-before 规则

```
volatile 变量规则：
  对 volatile 变量的写操作 happens-before 后续对该变量的读操作

传递性：
  A 写入普通变量 x
  A 写入 volatile 变量 v（触发 StoreStore + StoreLoad 屏障）
  B 读取 volatile 变量 v
  B 读取普通变量 x
  → A 写 x happens-before B 读 x（通过 volatile v 传递）
```

```java
// 关键理解：volatile 不只保证自身变量的可见性
// 而是保证 volatile 写之前的所有写操作对 volatile 读之后的操作可见

int normalVar = 0;
volatile boolean ready = false;

// 线程A
normalVar = 42;     // 普通写
ready = true;       // volatile 写（StoreStore 屏障确保前面的写先刷）

// 线程B
if (ready) {        // volatile 读（LoadLoad 屏障确保后面的读从主内存读）
    // 能看到 normalVar = 42（happens-before 传递性）
    use(normalVar);
}
```

### 4.2 完整的 happens-before 规则体系

| 规则 | 说明 |
|------|------|
| 程序顺序规则 | 单线程内，前面的操作 hb 后面的操作 |
| volatile 规则 | volatile 写 hb 后续 volatile 读 |
| 监视器规则 | unlock hb 后续 lock（synchronized 的可见性保证）|
| 线程启动规则 | `Thread.start()` hb 该线程的每个操作 |
| 线程终止规则 | 线程所有操作 hb `Thread.join()` 返回 |
| 中断规则 | `Thread.interrupt()` hb 被中断线程检测到中断 |
| 对象终结规则 | 构造函数完成 hb `finalize()` 开始 |
| 传递性 | A hb B，B hb C → A hb C |

---

## 五、volatile 与 synchronized 对比

| 特性 | volatile | synchronized |
|------|----------|-------------|
| 原子性 | ❌ 不保证 | ✅ 保证 |
| 可见性 | ✅ 保证 | ✅ 保证 |
| 有序性 | ✅ 禁止重排序 | ✅ 保证有序性 |
| 锁阻塞 | 无锁，不会阻塞 | 会阻塞等待锁 |
| 适用场景 | 状态标志、发布对象、DCL | 复合操作、临界区 |

---

## 五、volatile 与 synchronized 对比

| 特性 | volatile | synchronized |
|------|----------|-------------|
| 原子性 | ❌ 不保证 | ✅ 保证 |
| 可见性 | ✅ 保证 | ✅ 保证 |
| 有序性 | ✅ 禁止重排序 | ✅ 保证有序性 |
| 锁阻塞 | 无锁，不会阻塞 | 会阻塞等待锁 |
| 适用场景 | 状态标志、发布对象、DCL | 复合操作、临界区 |

---

## 六、CPU 缓存行伪共享（False Sharing）与 @Contended

```
缓存行（Cache Line）：CPU 缓存的最小读写单位，通常 64 字节

伪共享问题：
  两个变量 a、b 位于同一缓存行
  线程A 写 a → 整个缓存行标记为 Modified → 线程B 的缓存行 Invalid
  线程B 写 b → 整个缓存行标记为 Modified → 线程A 的缓存行 Invalid
  → a、b 互不相关，却互相影响（"假共享"，性能急剧下降）

a [ 8字节 ] b [ 8字节 ] ...... [ 同一64字节缓存行 ]
```

```java
// 解决方案1：手动填充（JDK 8 之前）
class PaddedLong {
    volatile long value;
    // 填充 7 个 long，使 value 独占一个缓存行（7*8 + 8 = 64字节）
    long p1, p2, p3, p4, p5, p6, p7;
}

// 解决方案2：@sun.misc.Contended（JDK 8+，推荐）
// 需要 JVM 参数：-XX:-RestrictContended
@sun.misc.Contended
class ContendedLong {
    volatile long value;  // JVM 自动在两侧添加 128 字节填充
}

// 实战案例：LongAdder 的 Cell 类就用了 @Contended
// 防止多个 Cell 落在同一缓存行，保证高并发下的并行更新
@sun.misc.Contended static final class Cell {
    volatile long value;
}
```

---

## 七、面试要点速查

## 七、面试要点速查

| 问题 | 要点 |
|------|------|
| volatile 能保证原子性吗 | 不能！只保证可见性和有序性。i++ 等复合操作不安全 |
| volatile 的原理 | 写操作加 StoreLoad 屏障刷到主内存；读操作加 LoadLoad 屏障从主内存读 |
| DCL 为什么必须用 volatile | 防止 `new Singleton()` 的指令重排序（分配→赋值→初始化 变成 分配→初始化→赋值），避免其他线程拿到未初始化对象 |
| volatile 和 synchronized 的区别 | volatile 轻量（无锁），只保证可见性+有序性；synchronized 重量（有锁），同时保证原子性 |
| long/double 为什么要用 volatile | 32位JVM 中 64位变量读写非原子，volatile 保证原子读写 |
| 可见性问题的根源 | CPU 多级缓存（L1/L2/L3），线程各自读写私有缓存副本，MESI 协议靠 volatile 触发缓存失效 |
| happens-before 是什么 | JMM 的可见性规则：A happens-before B，则 A 的结果对 B 可见。volatile 写 hb volatile 读 |
| 什么是伪共享（False Sharing） | 两个无关变量落在同一缓存行，相互触发缓存失效。用 @Contended 或手动填充解决 |


---

**相关面试题** → [[../../10_Developlanguage/001_java/03_JavaConcurrencySubject/03、volatile 关键字|03、volatile 关键字]] | [[../../10_Developlanguage/001_java/03_JavaConcurrencySubject/11、内存模型与可见性|11、内存模型与可见性]]
