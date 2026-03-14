# CAS 与原子类

> **核心关键词**：Compare-And-Swap、Unsafe、AtomicInteger、LongAdder、ABA问题、CAS自旋

---

## 一、CAS 原理

### 1.1 什么是 CAS？

**Compare-And-Swap（比较并交换）** 是一条 CPU 原子指令（x86: `CMPXCHG`）：

```
CAS(内存地址V, 预期旧值A, 新值B)：
  如果 V 的当前值 == A：
      原子地将 V 更新为 B，返回 true
  否则：
      不做任何改变，返回 false
```

```java
// Java 中的 CAS 通过 Unsafe 实现
// AtomicInteger.compareAndSet 源码
public final boolean compareAndSet(int expectedValue, int newValue) {
    return U.compareAndSetInt(this, VALUE, expectedValue, newValue);
    //  U = sun.misc.Unsafe 单例
    //  VALUE = value 字段的内存偏移量
}
```

### 1.2 CAS 的优势

相比 `synchronized`，CAS 是**乐观锁**：
- 假设不会有冲突，不加锁
- 操作失败时自旋重试（忙等待）
- 适合竞争不激烈的场景（冲突少，自旋次数少）

```java
// CAS 自旋的典型实现
public final int getAndIncrement() {
    for (;;) {
        int current = get();         // 读当前值
        int next = current + 1;
        if (compareAndSet(current, next))  // CAS 尝试更新
            return current;          // 成功则返回旧值
        // 失败则自旋重试（其他线程抢先修改了值）
    }
}
```

---

## 二、CAS 的三大问题

### 2.1 ABA 问题

```
线程A 读到 V = A
线程B 修改 V = B
线程B 再改回 V = A
线程A CAS 判断 V == A → 成功（但中间值变了，A 感知不到）
```

**危险场景**：链表操作中，ABA 可能导致链表节点被错误引用。

**解决方案**：**版本号（Stamp）机制**

```java
// AtomicStampedReference：带版本号的原子引用
AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);

// 读取时同时读版本号
int[] stampHolder = new int[1];
String value = ref.get(stampHolder);  // value="A", stamp=0

// CAS 时需要值和版本号都匹配
ref.compareAndSet("A", "B", 0, 1);  // 旧值,新值,旧版本号,新版本号

// AtomicMarkableReference：简化版，用 boolean 标记
AtomicMarkableReference<String> markRef = new AtomicMarkableReference<>("A", false);
markRef.compareAndSet("A", "B", false, true);
```

### 2.2 自旋开销

竞争激烈时，自旋次数增多，CPU 空转严重：

```java
// ❌ 高并发写入场景下，大量线程自旋竞争同一个 AtomicLong
AtomicLong counter = new AtomicLong(0);
// 10000个线程同时 counter.incrementAndGet() → 大量 CAS 失败，自旋

// ✅ 高并发场景用 LongAdder（JDK 8+）
LongAdder adder = new LongAdder();
adder.increment();  // 化单点竞争为多点
long sum = adder.sum();
```

### 2.3 只能保证单个变量的原子性

CAS 无法保证多个变量组合的原子性：

```java
// ❌ 想原子地更新两个变量（账户余额+流水号）
// 两次 CAS 之间可能被其他线程插入

// ✅ 解决：AtomicReference 将多个变量封装成一个对象
AtomicReference<AccountState> stateRef = new AtomicReference<>(new AccountState(100, 0));
AccountState current = stateRef.get();
AccountState newState = new AccountState(current.balance - 50, current.version + 1);
stateRef.compareAndSet(current, newState);  // 原子更新整个状态对象
```

---

## 三、Java 原子类体系

### 3.1 基本类型原子类

```java
AtomicInteger    ai = new AtomicInteger(0);
AtomicLong       al = new AtomicLong(0L);
AtomicBoolean    ab = new AtomicBoolean(false);

// 常用方法
ai.get()                    // 读取当前值
ai.set(10)                  // 设置值（volatile 写）
ai.getAndSet(10)            // 原子设置并返回旧值
ai.getAndIncrement()        // i++ 原子版，返回旧值
ai.incrementAndGet()        // ++i 原子版，返回新值
ai.getAndAdd(5)             // i+=5 原子版，返回旧值
ai.addAndGet(5)             // i+=5 原子版，返回新值
ai.compareAndSet(0, 1)      // CAS：期望值0，新值1
ai.compareAndExchange(0, 1) // JDK 9+：CAS，返回旧值（不是boolean）

// JDK 8 新增函数式更新
ai.updateAndGet(x -> x * 2)         // 原子地 x = x*2，返回新值
ai.accumulateAndGet(5, Integer::sum) // 原子地 x = x+5，返回新值
```

### 3.2 引用类型原子类

```java
// AtomicReference：原子引用
AtomicReference<User> userRef = new AtomicReference<>(new User("张三"));
userRef.compareAndSet(currentUser, newUser);  // 原子更新引用

// AtomicStampedReference：带版本号（解决ABA）
AtomicStampedReference<String> ref = new AtomicStampedReference<>("v1", 0);

// AtomicMarkableReference：带标记位（简化版版本号）
AtomicMarkableReference<Node> nodeRef = new AtomicMarkableReference<>(node, false);
```

### 3.3 数组原子类

```java
// 原子更新数组中的元素
AtomicIntegerArray    arr = new AtomicIntegerArray(new int[]{1, 2, 3});
AtomicLongArray       larr = new AtomicLongArray(10);
AtomicReferenceArray<String> rarr = new AtomicReferenceArray<>(new String[10]);

arr.getAndIncrement(0);           // 原子地 arr[0]++，返回旧值
arr.compareAndSet(1, 2, 3);      // 原子地：如果arr[1]==2，则arr[1]=3
```

### 3.4 字段更新原子类

```java
// 不修改类定义，原子地更新某个类的 volatile 字段
// 字段必须是 volatile 修饰的
class Node {
    volatile int value;
}

AtomicIntegerFieldUpdater<Node> updater = 
    AtomicIntegerFieldUpdater.newUpdater(Node.class, "value");

Node node = new Node();
updater.compareAndSet(node, 0, 1);  // 原子更新 node.value
```

---

## 四、LongAdder vs AtomicLong

**AtomicLong 的瓶颈**：

```
高并发写入：
线程1 → CAS(0, 1) 成功
线程2 → CAS(0, 1) 失败 → 自旋 → CAS(1, 2) 成功
线程3 → CAS(0, 1) 失败 → 自旋 → CAS(0, 1) 失败 → 自旋 → CAS(2, 3) 成功
所有线程竞争同一个变量，自旋 CPU 空转
```

**LongAdder 的解决方案**（类似 ConcurrentHashMap 的分段计数）：

```
cells 数组（每个 Cell 独立计数）：
线程1 → cells[0] += 1  （无竞争）
线程2 → cells[1] += 1  （无竞争）
线程3 → cells[0] += 1  （轻微竞争）
线程4 → base += 1

sum() = base + cells[0] + cells[1] + ...
```

```java
// 性能对比（高并发场景）
// AtomicLong：大量 CAS 失败，吞吐量下降
// LongAdder：近似线性扩展，高并发写性能大幅提升

// 适用场景
// AtomicLong：需要精确值 + 中等并发（如 ID 生成器）
// LongAdder：高并发写 + 最终结果即可（如统计计数、监控指标）

LongAdder adder = new LongAdder();
adder.increment();                  // +1
adder.add(5);                       // +5
long total = adder.sum();           // 近似总和
long totalAndReset = adder.sumThenReset(); // 获取并重置（原子操作）
```

## 五、Unsafe 类深度解析

`Unsafe` 是 Java 实现 CAS 的底层入口，位于 `sun.misc.Unsafe`（JDK 9+ 迁移到 `jdk.internal.misc.Unsafe`）。

### 5.1 如何获取 Unsafe

```java
// 方法1：反射获取（测试/框架层用）
Field f = Unsafe.class.getDeclaredField("theUnsafe");
f.setAccessible(true);
Unsafe unsafe = (Unsafe) f.get(null);

// 方法2：JDK 9+ 用 VarHandle（推荐，见下节）
```

### 5.2 Unsafe 的核心能力

```java
// 1. CAS 操作（AtomicXxx 的基础）
unsafe.compareAndSwapInt(obj, offset, expected, update);
unsafe.compareAndSwapLong(obj, offset, expected, update);
unsafe.compareAndSwapObject(obj, offset, expected, update);

// 2. 内存操作（直接读写内存）
unsafe.allocateMemory(size);          // 堆外内存分配
unsafe.freeMemory(address);           // 释放堆外内存
unsafe.putLong(address, value);       // 写入堆外内存
unsafe.getLong(address);              // 读取堆外内存

// 3. 获取字段内存偏移量
long offset = unsafe.objectFieldOffset(MyClass.class.getDeclaredField("value"));
// AtomicInteger.VALUE 就是通过 objectFieldOffset 获取的

// 4. park/unpark（LockSupport 的底层）
unsafe.park(false, 0L);        // 挂起当前线程
unsafe.unpark(thread);         // 恢复线程

// 5. 绕过构造函数创建对象（序列化框架用）
MyClass obj = (MyClass) unsafe.allocateInstance(MyClass.class);
```

### 5.3 VarHandle（JDK 9+，Unsafe 的官方替代）

```java
import java.lang.invoke.MethodHandles;
import java.lang.invoke.VarHandle;

class Counter {
    volatile int value = 0;
    
    // 获取 VarHandle
    private static final VarHandle VALUE_HANDLE;
    static {
        try {
            VALUE_HANDLE = MethodHandles.lookup()
                .findVarHandle(Counter.class, "value", int.class);
        } catch (Exception e) {
            throw new Error(e);
        }
    }
    
    // CAS 操作（等价于 AtomicInteger.compareAndSet）
    public boolean cas(int expected, int update) {
        return VALUE_HANDLE.compareAndSet(this, expected, update);
    }
    
    // getAndAdd
    public int getAndAdd(int delta) {
        return (int) VALUE_HANDLE.getAndAdd(this, delta);
    }
}

// VarHandle 相比 Unsafe 的优势：
// 1. 类型安全（编译期检查）
// 2. 模块系统友好（JDK 9 强模块化后 Unsafe 访问受限）
// 3. 支持 volatile/plain/opaque/acquire/release 多种内存访问语义
```

---

## 六、LongAdder 内部实现细节

```java
// LongAdder 继承自 Striped64，核心数据结构：
abstract class Striped64 {
    @Contended static final class Cell {
        volatile long value;
        // @Contended：防止伪共享，每个 Cell 独占一个缓存行
    }
    
    transient volatile Cell[] cells;  // Cell 数组，大小是 2 的幂
    transient volatile long base;     // 无竞争时的基础值
    transient volatile int cellsBusy; // 初始化/扩容 Cell 数组时的自旋锁
}
```

```
LongAdder.add(x) 流程：

1. 尝试 CAS 更新 base（无竞争路径，最快）
   → 成功：返回
   → 失败（有竞争）：进入 cells 路径

2. 获取当前线程的 probe（线程本地探针哈希值）
   计算 cells[probe & (cells.length - 1)] → 目标 Cell

3. 尝试 CAS 更新目标 Cell.value
   → 成功：返回
   → 失败（此 Cell 竞争激烈）：

4. 扩容 cells 数组（double size），更换 probe 重新散列
   → 直到 CAS 成功

sum() = base + Σ cells[i].value
  ⚠️ sum() 是近似值！计算期间其他线程可能还在 add()
  sumThenReset() 则先获取总值再清零（也非严格原子）
```

---

## 七、面试要点速查

| 问题 | 要点 |
|------|------|
| CAS 的原理 | 比较内存值与预期值，相等则原子更新为新值，基于 CPU 原子指令 CMPXCHG |
| CAS 的三大问题 | ABA问题（版本号解决）、自旋开销（LongAdder解决）、只能保证单变量（AtomicReference解决）|
| ABA 问题怎么解决 | `AtomicStampedReference` 带版本号；`AtomicMarkableReference` 带标记位 |
| AtomicLong 和 LongAdder 的区别 | LongAdder 分散竞争到多个 Cell，高并发写性能更好；但 sum() 是近似值 |
| Unsafe 的作用 | 提供直接操作内存的能力，CAS 操作底层都依赖 Unsafe.compareAndSwapXxx |
| CAS 和 synchronized 怎么选 | 竞争不激烈选 CAS（乐观锁，无阻塞）；竞争激烈选 synchronized（避免大量自旋）|


---

**相关面试题** → [[../../10_Developlanguage/001_java/03_JavaConcurrencySubject/09、原子类|09、原子类]]
