# HashMap 深度解析

> **核心关键词**：哈希表、拉链法、红黑树、扰动函数、扩容、负载因子、线程不安全

---

## 一、整体架构

### 1.1 数据结构：数组 + 链表 + 红黑树

```
HashMap 内部结构（JDK 8）：

Node<K,V>[] table（数组，长度始终是 2 的幂次）

index 0: [ Node(k1,v1) → Node(k5,v5) → null ]  ← 链表（短）
index 1: [ null ]
index 2: [ TreeNode(k2,v2) ]  ← 红黑树根节点（链表过长后转换）
          /    \
    TreeNode  TreeNode
index 3: [ Node(k3,v3) ]
...
```

### 1.2 核心字段

```java
public class HashMap<K,V> {
    // 存储数据的数组（桶数组），长度始终是 2 的幂次
    transient Node<K,V>[] table;
    
    // 已存储的键值对数量
    transient int size;
    
    // 修改次数（用于 fail-fast 迭代器）
    transient int modCount;
    
    // 下一次扩容的阈值 = capacity × loadFactor
    int threshold;
    
    // 负载因子，默认 0.75
    final float loadFactor;
    
    // 默认初始容量：16（必须是 2 的幂次）
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
    
    // 最大容量：2^30
    static final int MAXIMUM_CAPACITY = 1 << 30;
    
    // 默认负载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
    // 链表转红黑树的阈值：链表长度 >= 8
    static final int TREEIFY_THRESHOLD = 8;
    
    // 红黑树退化为链表的阈值：节点数 <= 6
    static final int UNTREEIFY_THRESHOLD = 6;
    
    // 触发树化的最小数组容量：64（数组容量 < 64 时优先扩容而不是树化）
    static final int MIN_TREEIFY_CAPACITY = 64;
}
```

---

## 二、哈希算法：扰动函数

### 2.1 为什么要扰动？

计算桶索引公式：`index = (n - 1) & hash`

当 `n` 较小（如 16），只有 hash 值的低 4 位参与计算，高位完全无效。如果大量 key 的低位相同（如 Integer key 都是 16 的倍数），会严重堆积。

### 2.2 扰动函数源码

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

**原理**：将 hashCode 的高 16 位与低 16 位做异或，让高位参与低位的计算，减少哈希冲突。

```
假设 hashCode = 0x12345678
高16位：0x1234 = 0001 0010 0011 0100
低16位：0x5678 = 0101 0110 0111 1000
XOR结果：       0100 0100 0100 1100  = 0x444C

最终用于定位的是低 4 位（n=16时），原始 0x8 vs 扰动后 0xC，分布更散
```

---

## 三、put 流程（最核心）

### 3.1 完整流程图

```mermaid
flowchart TD
    A[put key, value] --> B[计算 hash key]
    B --> C{table 是否为空?}
    C -->|是| D[resize 初始化]
    D --> E[计算桶索引 i = n-1 & hash]
    C -->|否| E
    E --> F{table[i] 是否为 null?}
    F -->|是| G[直接放入新 Node]
    F -->|否| H{头节点 key 是否相同?}
    H -->|是| I[覆盖 value]
    H -->|否| J{头节点是否是 TreeNode?}
    J -->|是| K[调用红黑树 putTreeVal]
    J -->|否| L[遍历链表]
    L --> M{找到相同 key?}
    M -->|是| N[覆盖 value]
    M -->|否| O[尾部插入新 Node]
    O --> P{链表长度 >= 8?}
    P -->|是| Q{数组容量 >= 64?}
    Q -->|是| R[treeifyBin 转红黑树]
    Q -->|否| S[resize 扩容]
    P -->|否| T[检查是否需要扩容]
    G --> T
    I --> T
    K --> T
    N --> T
    R --> T
    S --> T
    T --> U{size > threshold?}
    U -->|是| V[resize 扩容]
    U -->|否| W[结束]
    V --> W
```

### 3.2 关键源码分析

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // 1. 数组为空时初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // 2. 计算桶索引，桶为空直接放入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        
        // 3. 头节点 key 相同，标记待覆盖
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        
        // 4. 红黑树节点，走树插入
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        
        // 5. 链表遍历插入
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null); // 尾插法！（JDK8 改为尾插）
                    if (binCount >= TREEIFY_THRESHOLD - 1)     // binCount >= 7，即链表长度 = 8
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        
        // 6. 覆盖旧值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            return oldValue;
        }
    }
    
    ++modCount;
    // 7. 超出阈值触发扩容
    if (++size > threshold)
        resize();
    return null;
}
```

> ⚠️ **JDK 7 vs JDK 8 插入方式**：JDK 7 用**头插法**（新节点插入链表头部），并发扩容时可能形成**环形链表**导致死循环；JDK 8 改为**尾插法**，解决了环形链表问题（但仍然线程不安全）。

---

## 四、扩容机制（resize）

### 4.1 触发时机

`size > threshold`，其中 `threshold = capacity × loadFactor`（默认 16 × 0.75 = 12）

### 4.2 扩容策略

- **新容量 = 旧容量 × 2**（始终保持 2 的幂次）
- **新阈值 = 旧阈值 × 2**

### 4.3 高效重哈希（核心优化）

由于容量始终是 2 的幂次，扩容后每个元素的新位置只有两种可能：

```
扩容前：n = 16，二进制 10000，n-1 = 01111
扩容后：n = 32，二进制 100000，n-1 = 011111

元素的新索引 = hash & (newCap - 1)
            = hash & 011111

对比旧索引   = hash & 01111

新旧索引的区别只在第5位（原容量对应的那一位）：
- 如果 hash 的第5位是 0 → 新索引 = 旧索引（位置不变）
- 如果 hash 的第5位是 1 → 新索引 = 旧索引 + oldCap（位置偏移旧容量）
```

```java
// 源码中的关键判断（e.hash & oldCap 就是检查那一位）
if ((e.hash & oldCap) == 0)
    // 新位置 = 原位置（放入 lo 链）
else
    // 新位置 = 原位置 + oldCap（放入 hi 链）
```

这个设计让扩容时**无需重新计算 hash**，极大提升了效率。

---

## 五、红黑树转换

### 5.1 链表转红黑树的条件

**同时满足两个条件**：
1. 链表长度 ≥ `TREEIFY_THRESHOLD`（8）
2. 数组容量 ≥ `MIN_TREEIFY_CAPACITY`（64）

条件2的意义：如果数组容量还小，说明哈希冲突是因为容量不够导致的，应该优先扩容来分散元素，而不是直接树化。

### 5.2 红黑树退化为链表的条件

- `resize()` 时，如果树的元素 ≤ `UNTREEIFY_THRESHOLD`（6），退化为链表
- **为什么退化阈值是 6 而不是 7？** 避免在 7/8 附近来回反复转换（迟滞效应）

### 5.3 为什么选红黑树而不是 AVL 树？

| 特性 | 红黑树 | AVL 树 |
|------|--------|--------|
| 平衡性 | 弱平衡（最长路径 ≤ 最短路径的2倍）| 严格平衡（高度差 ≤ 1）|
| 查找性能 | O(log n)，略差于 AVL | O(log n)，更优 |
| 插入/删除旋转 | 旋转次数少，更快 | 旋转次数多 |

HashMap 的主要操作是**插入和删除**，红黑树的旋转代价更小，所以选红黑树。

---

## 六、负载因子为什么是 0.75？

这是**时间与空间的权衡**：

| 负载因子 | 空间利用率 | 哈希冲突 | 查询性能 |
|---------|-----------|---------|---------|
| 过低（如 0.5）| 低（桶大量空闲）| 少 | 快 |
| 过高（如 1.0）| 高 | 多 | 慢 |
| **0.75** | **适中** | **适中** | **均衡** |

泊松分布分析：在负载因子 0.75 时，链表长度达到 8 的概率仅约 **0.00000006**（极低），这也是为什么树化阈值设为 8。

---

## 七、线程不安全问题

### 7.1 数据覆盖（JDK 7 & 8）

```
线程A 和 线程B 同时 put 不同 key，但计算出相同桶索引 i

线程A：判断 table[i] == null → 准备写入，但被切换
线程B：判断 table[i] == null → 写入成功
线程A：恢复执行 → 直接写入 table[i]，覆盖了线程B的数据！
```

### 7.2 环形链表（仅 JDK 7，头插法导致）

并发扩容时，头插法会将链表中的节点顺序反转，两个线程各自扩容后可能形成环形链表，导致 `get` 操作死循环。

JDK 8 改为尾插法后，不再有此问题，但**数据覆盖问题依然存在**。

### 7.3 解决方案

```java
// 方案1：使用 ConcurrentHashMap（推荐）
Map<String, String> map = new ConcurrentHashMap<>();

// 方案2：Collections.synchronizedMap（性能差，全表加锁）
Map<String, String> map = Collections.synchronizedMap(new HashMap<>());

// 方案3：Hashtable（过时，不推荐）
```

---

## 八、实战场景与踩坑

### 场景1：初始容量的合理设置

```java
// 如果已知会存入 1000 个元素，初始容量应该设多少？
// 避免扩容：size / loadFactor = 1000 / 0.75 ≈ 1334
// 取大于 1334 的最小 2 的幂次：2048
Map<String, Object> map = new HashMap<>(2048);

// JDK 的计算工具（Guava）：
// Maps.newHashMapWithExpectedSize(1000) 内部自动计算
```

### 场景2：key 的 equals 和 hashCode 规范

```java
// ✅ 使用自定义对象作为 key，必须正确实现 equals 和 hashCode
// 如果 equals 相等，hashCode 必须相等
// 如果 hashCode 相等，equals 不一定相等（允许哈希碰撞）

class Point {
    int x, y;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return x == p.x && y == p.y;
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(x, y);  // JDK 7+ 推荐写法
    }
}

// ❌ 常见错误：只重写 equals 不重写 hashCode
// 结果：equals 相等的两个 Point 对象会落在不同的桶里，找不到！
```

### 场景3：遍历时不能修改

```java
Map<String, Integer> map = new HashMap<>();
// ...填入数据...

// ❌ 遍历时删除 → ConcurrentModificationException
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    if (entry.getValue() < 0) {
        map.remove(entry.getKey());  // 修改 modCount，迭代器检测到抛异常
    }
}

// ✅ 用迭代器的 remove 方法
Iterator<Map.Entry<String, Integer>> it = map.entrySet().iterator();
while (it.hasNext()) {
    Map.Entry<String, Integer> entry = it.next();
    if (entry.getValue() < 0) {
        it.remove();  // 迭代器自身的 remove，安全
    }
}

// ✅ JDK 8：removeIf（推荐）
map.entrySet().removeIf(e -> e.getValue() < 0);
```

---

## 九、面试要点速查

| 问题 | 要点 |
|------|------|
| HashMap 底层结构 | 数组 + 链表 + 红黑树（JDK 8），数组长度始终是 2 的幂次 |
| 哈希冲突如何解决 | 拉链法（链表），超过阈值树化为红黑树 |
| 扰动函数的作用 | 高16位异或低16位，让高位参与桶索引计算，减少冲突 |
| 为什么容量是 2 的幂次 | `(n-1) & hash` 等价取模，高效；扩容时重哈希只需判断一位 |
| 默认负载因子 0.75 | 时间与空间的均衡，泊松分布下链表长度 ≥ 8 概率极低 |
| 链表转红黑树的条件 | 链表长度 ≥ 8 **且** 数组容量 ≥ 64（否则优先扩容）|
| JDK 7 头插法的问题 | 并发扩容时链表顺序反转可能形成环，导致死循环 |
| 线程不安全的根本原因 | 复合操作非原子（判断+写入非原子），多线程会数据覆盖 |
| 如何线程安全 | ConcurrentHashMap（推荐）|
