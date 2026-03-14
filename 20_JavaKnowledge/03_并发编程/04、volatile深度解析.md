# volatile 深度解析

> **核心关键词**：可见性、有序性、内存屏障、happens-before、不保证原子性、DCL

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

## 四、volatile 与 synchronized 对比

| 特性 | volatile | synchronized |
|------|----------|-------------|
| 原子性 | ❌ 不保证 | ✅ 保证 |
| 可见性 | ✅ 保证 | ✅ 保证 |
| 有序性 | ✅ 禁止重排序 | ✅ 保证有序性 |
| 锁阻塞 | 无锁，不会阻塞 | 会阻塞等待锁 |
| 适用场景 | 状态标志、发布对象、DCL | 复合操作、临界区 |

---

## 五、面试要点速查

| 问题 | 要点 |
|------|------|
| volatile 能保证原子性吗 | 不能！只保证可见性和有序性。i++ 等复合操作不安全 |
| volatile 的原理 | 写操作加 StoreLoad 屏障刷到主内存；读操作加 LoadLoad 屏障从主内存读 |
| DCL 为什么必须用 volatile | 防止 `new Singleton()` 的指令重排序（分配→赋值→初始化 变成 分配→初始化→赋值），避免其他线程拿到未初始化对象 |
| volatile 和 synchronized 的区别 | volatile 轻量（无锁），只保证可见性+有序性；synchronized 重量（有锁），同时保证原子性 |
| long/double 为什么要用 volatile | 32位JVM 中 64位变量读写非原子，volatile 保证原子读写 |
