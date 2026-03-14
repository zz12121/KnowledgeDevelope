# Java 内存模型（JMM）

> **核心关键词**：主内存、工作内存、happens-before、可见性、有序性、原子性

---

## 一、为什么需要 JMM？

### 1.1 硬件层面的问题

现代 CPU 有多级缓存（L1/L2/L3）。每个核心有自己的缓存，修改后不立即回写主内存，导致**多个核心看到的数据不一致**：

```
CPU 核心1 → L1 Cache → L2 Cache → L3 Cache → 主内存
CPU 核心2 → L1 Cache → L2 Cache → L3 Cache ↗

核心1 修改了变量 x=1，仅在自己的 L1 Cache 中
核心2 读取 x，从主内存读到 x=0 → 可见性问题
```

CPU 还会对指令进行**乱序执行（Out-of-Order Execution）**以提升效率，导致代码执行顺序与源码不一致 → 有序性问题。

### 1.2 JMM 的目标

Java 内存模型（JMM）是一个**规范**，定义了：
1. **多线程如何访问共享变量**（主内存 vs 工作内存）
2. **操作的原子性、可见性、有序性**的保证规则
3. **happens-before** 规则：程序员可以依赖的内存语义

---

## 二、主内存与工作内存

### 2.1 抽象模型

```
┌──────────────────────────────────────────────┐
│               主内存 (Main Memory)            │
│  [共享变量 x=0]  [共享变量 y=1]  ...           │
└────────────────┬────────────────┬─────────────┘
                 │  read/write    │  read/write
    ┌────────────▼───┐      ┌─────▼──────────┐
    │ 线程1工作内存   │      │  线程2工作内存  │
    │  x=0（副本）   │      │  x=0（副本）   │
    └───────────────┘      └────────────────┘
```

**规则**：
- 线程不能直接读写主内存中的共享变量，必须先拷贝到**工作内存（副本）**
- 工作内存对其他线程不可见
- 线程间通信必须经过主内存：线程A 修改 → 写回主内存 → 线程B 读取

### 2.2 8 种内存操作

JMM 定义了以下原子操作（JDK 8 后简化为 lock/read/write/unlock）：

| 操作 | 作用对象 | 含义 |
|------|---------|------|
| `lock` | 主内存变量 | 将变量标记为线程独占 |
| `unlock` | 主内存变量 | 释放独占标记 |
| `read` | 主内存变量 | 将变量值从主内存传输到工作内存 |
| `load` | 工作内存变量 | 将 read 得到的值放入工作内存副本 |
| `use` | 工作内存变量 | 将工作内存值传给执行引擎（读操作） |
| `assign` | 工作内存变量 | 接收执行引擎赋值，放入工作内存（写操作） |
| `store` | 工作内存变量 | 将工作内存值传输到主内存 |
| `write` | 主内存变量 | 将 store 传来的值写入主内存 |

---

## 三、三大特性

### 3.1 原子性（Atomicity）

操作不可被打断，要么全部完成，要么不执行。

**JMM 保证的原子操作**：对基本类型（除 long/double 外）的 read/load/use/assign/store/write 是原子的。

```java
// ❌ i++ 不是原子操作！分为三步：读 → 加1 → 写
int i = 0;
i++;  // 线程A 读到 0，线程B 读到 0，各自加1写回 → 结果是 1 而不是 2

// ✅ 解决方案
AtomicInteger ai = new AtomicInteger(0);
ai.incrementAndGet();  // CAS 保证原子性

// ✅ synchronized 解决
synchronized(this) { i++; }
```

> **long/double 的特殊性**：在 32 位 JVM 中，64 位的 long/double 的读写可能被分为两次 32 位操作，不保证原子性。但主流 64 位 JVM 已将其作为原子操作，实践中几乎不是问题。

### 3.2 可见性（Visibility）

一个线程修改了共享变量的值，其他线程能立即看到最新值。

**保证可见性的手段**：

```java
// 1. volatile：强制刷新到主内存 + 读取时从主内存获取最新值
volatile boolean flag = false;

// 2. synchronized：unlock 前将修改刷新到主内存
synchronized(lock) {
    sharedVar = newValue;
}

// 3. final：final 字段在构造器中初始化后，对其他线程可见（不可修改）
class ImmutablePoint {
    final int x, y;
    ImmutablePoint(int x, int y) { this.x = x; this.y = y; }
}
```

### 3.3 有序性（Ordering）

程序执行顺序与代码顺序一致（禁止指令重排序）。

**JVM 中的重排序**：
1. **编译器优化重排序**：不改变单线程语义的前提下，调整指令顺序
2. **指令级并行重排序**：CPU 流水线并行执行多条不相关指令
3. **内存系统重排序**：缓存和写缓冲区导致读写操作顺序变化

```java
// 经典重排序导致的问题：双重检查锁的错误写法
class Singleton {
    private static Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();  
                    // ❌ 问题：这行实际上有三步：
                    // 1. 分配内存
                    // 2. 初始化对象
                    // 3. 将引用指向内存
                    // 重排序可能变成 1→3→2，另一个线程在步骤3后、步骤2前访问，得到未初始化的对象
                }
            }
        }
        return instance;
    }
}

// ✅ 正确写法：volatile 禁止重排序
private static volatile Singleton instance;
```

---

## 四、happens-before 规则

### 4.1 什么是 happens-before？

如果操作 A happens-before 操作 B，则 A 的结果对 B 可见，且 A 的执行顺序在 B 之前。

**注意**：happens-before 不一定意味着 A 在时间上比 B 先执行，只是保证了内存可见性和顺序语义。

### 4.2 八大 happens-before 规则

```java
// 1. 程序顺序规则
// 同一线程内，前面的操作 happens-before 后面的操作
int x = 1;  // A
int y = x;  // B  → A happens-before B，y = 1 是可见的

// 2. 监视器锁规则
// unlock 操作 happens-before 后续的 lock 操作
synchronized(lock) { x = 42; }  // unlock
// ...
synchronized(lock) { use(x); }  // lock → x = 42 可见

// 3. volatile 变量规则
// volatile 写操作 happens-before 后续对该变量的读操作
volatile int v = 0;
v = 1;     // 线程A：volatile 写
// ...
int r = v; // 线程B：volatile 读 → r = 1 可见

// 4. 线程启动规则
// Thread.start() happens-before 该线程的所有操作
x = 10;
Thread t = new Thread(() -> System.out.println(x));
t.start();  // start happens-before t 内部所有操作 → 线程内能看到 x=10

// 5. 线程终止规则
// 线程内的所有操作 happens-before Thread.join() 返回
Thread t = new Thread(() -> { x = 42; });
t.start();
t.join();   // join 返回后，主线程能看到 x=42

// 6. 线程中断规则
// 对线程调用 interrupt() happens-before 该线程检测到中断

// 7. 对象终结规则
// 构造器完成 happens-before finalize() 开始

// 8. 传递性
// A hb B，B hb C → A hb C
```

### 4.3 happens-before 的实际意义

```java
// 实际案例：正确发布对象
class SafePublish {
    private int x;
    private volatile boolean ready;
    
    // 线程A：初始化
    public void writer() {
        x = 42;          // ① 写 x
        ready = true;    // ② volatile 写 ready
    }
    
    // 线程B：读取
    public void reader() {
        if (ready) {     // ③ volatile 读 ready
            use(x);      // ④ 读 x
        }
    }
    
    // happens-before 链：
    // ① hb ②（程序顺序规则）
    // ② hb ③（volatile 规则，B 读到 true）
    // ③ hb ④（程序顺序规则）
    // 传递性：① hb ④ → 线程B 在 ready=true 时能看到 x=42
}
```

---

## 五、内存屏障（Memory Barrier）

JMM 通过**内存屏障**指令实现有序性，屏障两侧的操作不能被重排序越过屏障：

| 屏障类型 | 描述 |
|---------|------|
| `LoadLoad`  | 屏障前的 Load 先于屏障后的 Load |
| `StoreStore`| 屏障前的 Store 先于屏障后的 Store |
| `LoadStore` | 屏障前的 Load 先于屏障后的 Store |
| `StoreLoad` | 屏障前的 Store 先于屏障后的 Load（**最强，开销最大**）|

**volatile 的内存屏障策略**：
```
volatile 写：
    [普通写]
    StoreStore 屏障  ← 禁止 volatile 写与上面的写重排序
    [volatile 写]
    StoreLoad 屏障   ← 禁止 volatile 写与下面的读/写重排序（最重要）

volatile 读：
    [volatile 读]
    LoadLoad 屏障    ← 禁止 volatile 读与下面的读重排序
    LoadStore 屏障   ← 禁止 volatile 读与下面的写重排序
    [普通读/写]
```

---

## 六、实战场景

### 场景1：正确的单例（DCL）

```java
public class Singleton {
    // volatile 禁止实例化对象时的指令重排序
    private static volatile Singleton INSTANCE;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (INSTANCE == null) {                   // 第一次检查：减少同步开销
            synchronized (Singleton.class) {
                if (INSTANCE == null) {           // 第二次检查：防止重复创建
                    INSTANCE = new Singleton();   // volatile 保证安全发布
                }
            }
        }
        return INSTANCE;
    }
}
```

### 场景2：停止线程标志位

```java
// ❌ 不可靠：flag 的修改对其他线程可能不可见
private boolean flag = false;

// ✅ volatile 保证可见性
private volatile boolean flag = false;

public void stop() { flag = false; }
public void run() {
    while (flag) {
        // 工作中...
    }
}
```

---

## 七、面试要点速查

| 问题 | 要点 |
|------|------|
| JMM 是什么 | Java 多线程内存访问规范，解决可见性/有序性/原子性问题 |
| 三大特性 | 原子性（synchronized/Atomic）、可见性（volatile/synchronized）、有序性（volatile/happens-before）|
| happens-before 是什么 | 如果 A hb B，则 A 的结果对 B 可见且 A 排在 B 之前 |
| volatile 能保证原子性吗 | 不能！只保证可见性和有序性（禁止重排序）；i++ 这类复合操作需要 synchronized 或原子类 |
| DCL 为什么需要 volatile | 防止 `instance = new Singleton()` 的指令重排序，避免其他线程拿到未初始化对象 |
| 重排序有哪几种 | 编译器优化重排、CPU 指令并行重排、内存系统（缓存）重排 |
