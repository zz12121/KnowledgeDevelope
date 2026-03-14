# ConcurrentHashMap 深度解析

> **核心关键词**：分段锁（JDK7）、synchronized+CAS（JDK8）、ForwardingNode、扩容协助、并发安全

---

## 一、演进历史

### 1.1 JDK 7：分段锁（Segment）

```
ConcurrentHashMap（JDK 7）

Segment[0]  Segment[1]  ...  Segment[15]  （默认 16 个段，继承 ReentrantLock）
    │            │                │
 HashEntry[]  HashEntry[]     HashEntry[]  （每个段内部是独立的哈希表）
```

**特点**：
- 锁的粒度是 Segment（包含多个桶）
- 同一时刻最多 16 个线程可以并发写（16个段互不干扰）
- 读操作不加锁（HashEntry 的 value 是 volatile）

**缺陷**：
- 初始化时确定并发度（默认16），之后不可修改
- Segment 是重量级数据结构，内存占用较大

### 1.2 JDK 8：synchronized + CAS

```
ConcurrentHashMap（JDK 8）

Node<K,V>[] table（数组）

index 0: [Node] → [Node]  ← 链表（synchronized 锁头节点）
index 1: [null]
index 2: [TreeNode]       ← 红黑树（synchronized 锁根节点）
...
```

**核心改进**：
- **抛弃 Segment**，直接锁**桶（链表头/树根）**，锁粒度更细
- 结合 **CAS** 处理无竞争场景，进一步降低开销
- 并发度最高可达数组长度（不再固定）
- 引入红黑树（与 HashMap JDK8 一致）

---

## 二、JDK 8 核心结构

### 2.1 关键字段

```java
public class ConcurrentHashMap<K,V> {
    // 桶数组，volatile 保证可见性
    transient volatile Node<K,V>[] table;
    
    // 扩容时的新数组（迁移未完成时不为 null）
    private transient volatile Node<K,V>[] nextTable;
    
    // 控制字段（多用途）：
    // -1：初始化中
    // -N：表示有 N-1 个线程正在进行扩容
    // 0：table 未初始化
    // 正数：下次扩容的阈值（= capacity × loadFactor）
    private transient volatile int sizeCtl;
    
    // 已存储的键值对数量（分布式计数，避免竞争）
    private transient volatile long baseCount;
    private transient volatile CounterCell[] counterCells;
}
```

### 2.2 Node 节点类型

```java
// 普通节点
static class Node<K,V> {
    final int hash;
    final K key;
    volatile V val;        // val 是 volatile，读不加锁
    volatile Node<K,V> next; // next 是 volatile
}

// 树节点（红黑树）
static final class TreeNode<K,V> extends Node<K,V> { ... }

// 树的代理节点（桶上挂的是 TreeBin，不是 TreeNode）
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;
    volatile TreeNode<K,V> first;
    volatile Thread waiter;
    volatile int lockState;  // TreeBin 有自己的读写锁
}

// 扩容期间的转发节点（标记该桶已迁移完毕）
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);  // hash = MOVED(-1)
        this.nextTable = tab;
    }
}
```

---

## 三、put 流程（JDK 8）

```mermaid
flowchart TD
    A[put key,value] --> B[计算 hash key 扰动处理]
    B --> C{table 是否为空?}
    C -->|是| D[initTable CAS 初始化]
    D --> E[计算桶索引 i = n-1 & hash]
    C -->|否| E
    E --> F{table[i] 是否为 null?}
    F -->|是| G[CAS 直接写入新 Node 无锁]
    F -->|否| H{hash == MOVED?}
    H -->|是| I[helpTransfer 协助扩容]
    I --> E
    H -->|否| J[synchronized table[i] 加锁]
    J --> K{是链表节点?}
    K -->|是| L[遍历链表 查找或尾插]
    K -->|否| M[是TreeBin? 走红黑树插入]
    L --> N{链表长度 >= 8?}
    N -->|是| O[treeifyBin 树化]
    G --> P[addCount +1 检查扩容]
    O --> P
    M --> P
    P --> Q{需要扩容?}
    Q -->|是| R[transfer 扩容迁移]
    Q -->|否| S[完成]
    R --> S
```

### 3.1 关键源码分析

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());  // 扰动：(h ^ (h >>> 16)) & HASH_BITS
    int binCount = 0;
    
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        
        // 1. 表为空，初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        
        // 2. 目标桶为空，CAS 写入（无锁）
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;  // CAS 成功，不需要加锁
        }
        
        // 3. 桶头是 ForwardingNode，说明正在扩容，协助迁移
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        
        // 4. 真正的写操作，加锁
        else {
            V oldVal = null;
            synchronized (f) {  // 锁住桶的头节点
                if (tabAt(tab, i) == f) {  // double-check，防止并发变化
                    if (fh >= 0) {  // 链表节点（hash >= 0）
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash && ((ek = e.key) == key || 
                                                   (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent) e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {  // 尾部插入
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {  // 红黑树
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent) p.val = value;
                        }
                    }
                }
            }
            // 链表长度 >= 8，树化
            if (binCount >= TREEIFY_THRESHOLD)
                treeifyBin(tab, i);
            if (oldVal != null) return oldVal;
            break;
        }
    }
    addCount(1L, binCount);  // 计数 +1，并检查是否需要扩容
    return null;
}
```

---

## 四、get 流程（无锁）

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;  // 头节点命中
        }
        else if (eh < 0)
            // hash < 0：ForwardingNode（-1）或 TreeBin（-2）
            // ForwardingNode.find() 会去 nextTable 中查找
            return (p = e.find(h, key)) != null ? p.val : null;
        
        while ((e = e.next) != null) {  // 遍历链表
            if (e.hash == h && ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

**为什么 get 不需要加锁？**
- `Node.val` 和 `Node.next` 都是 `volatile`，读到的值一定是最新的
- `table` 数组本身也是 `volatile`
- 扩容时使用 `ForwardingNode`，`find()` 方法会去新表查找，不会读到脏数据

---

## 五、并发扩容（多线程协助迁移）

### 5.1 扩容触发

```java
// addCount 中的扩容检查
if (s >= (long)(sc = sizeCtl) && ... && U.compareAndSwapInt(this, SIZECTL, sc, ...) )
    transfer(tab, null);  // 触发扩容
```

### 5.2 协助扩容机制

```
扩容期间：
1. 线程A 触发扩容，开始迁移桶（从后向前，每次认领一段）
2. 线程B put 时发现桶头是 ForwardingNode（hash=-1），调用 helpTransfer 协助迁移
3. 迁移完的桶被 ForwardingNode 占位，后续 get/put 会根据 ForwardingNode.nextTable 转发

每个线程每次认领的最小迁移步长（stride）：
min(n/8/NCPU, 16)  ← 保证每个线程有足够工作量，避免频繁 CAS 竞争
```

### 5.3 迁移完成标记

```java
// 桶迁移完毕后
setTabAt(tab, i, fwd);  // 将旧数组的桶设为 ForwardingNode
// 其他线程 get 时：ForwardingNode.find() → 在 nextTable 中查找
// 其他线程 put 时：helpTransfer → 协助迁移 → 然后在 nextTable 中 put
```

---

## 六、计数器设计（分布式计数）

### 6.1 为什么不用一个 AtomicLong？

高并发下，所有线程争抢同一个 `AtomicLong`，CAS 竞争激烈，性能差。

### 6.2 LongAdder 思想

```
线程1 → CAS baseCount  失败 → 写入 counterCells[1]
线程2 → CAS baseCount  成功 → baseCount++
线程3 → CAS baseCount  失败 → 写入 counterCells[3]

size = baseCount + sum(counterCells)
```

类似于 JDK 8 的 `LongAdder`：化单点竞争为多点竞争，大幅提升并发写性能，代价是 `size()` 需要汇总所有 cell（近似值）。

---

## 七、JDK 7 vs JDK 8 对比

| 对比项 | JDK 7 | JDK 8 |
|--------|-------|-------|
| 数据结构 | Segment 数组 + HashEntry 链表 | Node 数组 + 链表/红黑树 |
| 锁机制 | ReentrantLock 锁 Segment | synchronized 锁桶头节点 |
| 锁粒度 | Segment（含多个桶）| 单个桶（链表头/树根）|
| 并发度 | 固定（默认16），创建后不可改 | 动态，最高 = 数组长度 |
| 读操作 | 不加锁（volatile）| 不加锁（volatile）|
| 扩容 | 单线程扩容 | **多线程协助扩容** |
| 计数 | 全局计数（竞争激烈）| 分段计数（LongAdder 思想）|
| null 键值 | 不支持 | 不支持 |

---

## 八、与 HashMap 的主要区别

| 对比项 | HashMap | ConcurrentHashMap |
|--------|---------|-------------------|
| 线程安全 | ❌ | ✅ |
| null 键/值 | ✅ 允许 | ❌ **不允许**（会抛 NPE）|
| 性能（单线程）| 稍高 | 略有 CAS/volatile 开销 |
| 性能（多线程）| 不安全，不可用 | **高并发下优秀** |
| 迭代器 | fail-fast | 弱一致性（不抛 ConcurrentModificationException）|

**为什么不允许 null 键/值？**
> 多线程场景下，`map.get(key) == null` 无法区分"key 不存在"还是"key 对应的 value 是 null"，会引发歧义。HashMap 是单线程，可以用 `containsKey` 区分，但多线程下 `get` + `containsKey` 不是原子操作。

---

## 九、面试要点速查

| 问题 | 要点 |
|------|------|
| JDK8 相比 JDK7 的改进 | 抛弃 Segment，锁粒度降至单桶；多线程协助扩容；分段计数 |
| get 为什么不加锁 | val/next/table 都是 volatile；ForwardingNode 处理扩容中的查找 |
| put 的锁策略 | 桶为空用 CAS；桶非空用 synchronized 锁头节点 |
| ForwardingNode 的作用 | 扩容迁移完的桶的占位符，读操作转发到新表 |
| 为什么不允许 null | 多线程环境下 get(null)==null 有歧义（不存在 vs 值为 null）|
| size() 准确吗 | 近似值，通过 baseCount + counterCells 汇总，不是强一致 |
| 如何实现精确计数 | 使用 `mappingCount()`（返回 long）比 `size()`（返回 int）更准确，但仍是近似 |


---

**相关面试题** → [[../../10_Developlanguage/001_java/02_JavaCollectionSubject/04、Map 相关|04、Map 相关]] | [[../../10_Developlanguage/001_java/02_JavaCollectionSubject/07、并发集合|07、并发集合]] | [[../../10_Developlanguage/001_java/03_JavaConcurrencySubject/08、并发集合|并发-08、并发集合]]
