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

---

## 五、面试要点速查

| 问题 | 要点 |
|------|------|
| CAS 的原理 | 比较内存值与预期值，相等则原子更新为新值，基于 CPU 原子指令 CMPXCHG |
| CAS 的三大问题 | ABA问题（版本号解决）、自旋开销（LongAdder解决）、只能保证单变量（AtomicReference解决）|
| ABA 问题怎么解决 | `AtomicStampedReference` 带版本号；`AtomicMarkableReference` 带标记位 |
| AtomicLong 和 LongAdder 的区别 | LongAdder 分散竞争到多个 Cell，高并发写性能更好；但 sum() 是近似值 |
| Unsafe 的作用 | 提供直接操作内存的能力，CAS 操作底层都依赖 Unsafe.compareAndSwapXxx |
| CAS 和 synchronized 怎么选 | 竞争不激烈选 CAS（乐观锁，无阻塞）；竞争激烈选 synchronized（避免大量自旋）|
