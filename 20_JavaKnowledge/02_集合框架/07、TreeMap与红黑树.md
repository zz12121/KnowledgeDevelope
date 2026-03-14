# TreeMap 与红黑树

> **核心关键词**：红黑树、有序Map、NavigableMap、floor/ceiling/subMap、compareTo

---

## 一、整体架构

### 1.1 继承关系

```
Map<K,V>
    └── SortedMap<K,V>（保证键有序）
            └── NavigableMap<K,V>（支持导航操作：floor/ceiling/higher/lower）
                    └── TreeMap<K,V>（基于红黑树实现）
```

### 1.2 核心字段

```java
public class TreeMap<K,V> extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable {
    
    // 键的比较器（为 null 时使用键的自然排序 Comparable）
    private final Comparator<? super K> comparator;
    
    // 红黑树的根节点
    private transient Entry<K,V> root;
    
    // 树中键值对数量
    private transient int size = 0;
    
    // 结构修改次数（用于 fail-fast）
    private transient int modCount = 0;
}
```

### 1.3 Entry 节点（红黑树节点）

```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;    // 左子节点
    Entry<K,V> right;   // 右子节点
    Entry<K,V> parent;  // 父节点
    boolean color = BLACK;  // 节点颜色（红/黑）
}
```

---

## 二、红黑树原理

### 2.1 红黑树的五条性质

```
1. 每个节点要么是红色，要么是黑色
2. 根节点是黑色
3. 每个叶节点（NIL 空节点）是黑色
4. 如果一个节点是红色，则其两个子节点都是黑色（不能有相邻的红节点）
5. 从任一节点到其所有叶节点的路径上，经过的黑色节点数目相同（黑高相等）
```

**由这五条性质推导出：**
- 红黑树是**近似平衡**的（弱平衡），不像 AVL 树要求严格平衡
- 最长路径（红黑交替）≤ 最短路径（全黑）的 2 倍
- 树的高度 h ≤ 2 * log₂(n+1)，确保所有操作 O(log n)

### 2.2 旋转操作

```
左旋（以 X 为轴）：
    X                 Y
   / \               / \
  A   Y    →        X   C
     / \           / \
    B   C         A   B

右旋（以 Y 为轴）：
    Y                 X
   / \               / \
  X   C    →        A   Y
 / \                   / \
A   B                 B   C

旋转不改变树的有序性（中序遍历结果不变）
```

### 2.3 插入后的修复（变色 + 旋转）

插入新节点时，新节点着红色，可能破坏性质4（父红子红），需要修复：

```
情况1：叔父节点是红色
  → 父节点和叔父节点变黑，祖父变红，继续向上调整

情况2：叔父节点是黑色，新节点是父节点的右子（内侧）
  → 对父节点左旋，转化为情况3

情况3：叔父节点是黑色，新节点是父节点的左子（外侧）
  → 父节点变黑，祖父变红，对祖父右旋
```

---

## 三、TreeMap 核心操作

### 3.1 put 操作

```java
public V put(K key, V value) {
    Entry<K,V> t = root;
    
    // 树为空，创建根节点
    if (t == null) {
        compare(key, key);  // 类型检查
        root = new Entry<>(key, value, null);
        size = 1;
        return null;
    }
    
    int cmp;
    Entry<K,V> parent;
    Comparator<? super K> cpr = comparator;
    
    // BST 查找插入位置（二叉搜索树性质）
    if (cpr != null) {
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0) t = t.left;
            else if (cmp > 0) t = t.right;
            else return t.setValue(value);  // 键已存在，更新 value
        } while (t != null);
    } else {
        // 使用 Comparable 自然排序
        Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0) t = t.left;
            else if (cmp > 0) t = t.right;
            else return t.setValue(value);
        } while (t != null);
    }
    
    // 创建新节点并插入
    Entry<K,V> e = new Entry<>(key, value, parent);
    if (cmp < 0) parent.left = e;
    else parent.right = e;
    
    // 修复红黑树性质
    fixAfterInsertion(e);
    size++;
    return null;
}
```

### 3.2 get 操作（BST 查找）

```java
final Entry<K,V> getEntry(Object key) {
    if (comparator != null)
        return getEntryUsingComparator(key);
    
    Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)       p = p.left;
        else if (cmp > 0)  p = p.right;
        else               return p;  // 找到
    }
    return null;
}
```

---

## 四、NavigableMap 导航操作（TreeMap 独有优势）

### 4.1 边界查找

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(1, "a"); map.put(3, "c"); map.put(5, "e"); map.put(7, "g");

// floorKey(key)：返回 <= key 的最大键
map.floorKey(4);    // → 3（小于等于4的最大键）
map.floorKey(3);    // → 3（等于）
map.floorKey(0);    // → null（没有）

// ceilingKey(key)：返回 >= key 的最小键
map.ceilingKey(4);  // → 5（大于等于4的最小键）
map.ceilingKey(5);  // → 5（等于）
map.ceilingKey(8);  // → null（没有）

// lowerKey(key)：严格小于 key 的最大键
map.lowerKey(5);    // → 3（严格小于5）

// higherKey(key)：严格大于 key 的最小键
map.higherKey(5);   // → 7（严格大于5）

// firstKey / lastKey
map.firstKey();     // → 1（最小键）
map.lastKey();      // → 7（最大键）
```

### 4.2 子映射（范围视图）

```java
// subMap：[fromKey, toKey)  前闭后开
SortedMap<Integer, String> sub = map.subMap(2, 6);
// 包含 3, 5（不包含 2, 6）

// headMap：< toKey
SortedMap<Integer, String> head = map.headMap(5);
// 包含 1, 3（不包含 5）

// tailMap：>= fromKey
SortedMap<Integer, String> tail = map.tailMap(3);
// 包含 3, 5, 7

// 支持包含端点的版本（NavigableMap）
NavigableMap<Integer, String> sub2 = map.subMap(3, true, 5, true);
// 包含 3, 5（前后都包含）

// 这些都是视图（View），对视图的修改会反映到原 Map！
sub2.put(4, "d");  // 原 map 也会有 4="d"
```

### 4.3 倒序视图

```java
// descendingMap()：返回逆序排列的视图
NavigableMap<Integer, String> reversed = map.descendingMap();
// 迭代顺序：7, 5, 3, 1

// descendingKeySet()：倒序的键集合
NavigableSet<Integer> reversedKeys = map.descendingKeySet();
// 7, 5, 3, 1
```

---

## 五、TreeSet 原理

`TreeSet` 完全基于 `TreeMap` 实现：

```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable {
    
    // 底层就是 TreeMap，value 统一用 PRESENT 占位
    private transient NavigableMap<E,Object> m;
    private static final Object PRESENT = new Object();
    
    public boolean add(E e) {
        return m.put(e, PRESENT) == null;
    }
    
    public boolean contains(Object o) {
        return m.containsKey(o);
    }
}
```

---

## 六、HashMap vs TreeMap vs LinkedHashMap 完整对比

| 对比项 | HashMap | TreeMap | LinkedHashMap |
|--------|---------|---------|---------------|
| 底层实现 | 数组+链表+红黑树 | 红黑树 | HashMap+双向链表 |
| 键的顺序 | 无序 | **按键排序** | 插入/访问顺序 |
| get/put 时间复杂度 | O(1) | **O(log n)** | O(1) |
| 遍历效率 | 中（遍历数组）| 低（树的遍历）| **高**（链表顺序）|
| null 键 | 允许（1个）| **不允许** | 允许（1个）|
| 内存占用 | 低 | 中（树节点）| 中（链表指针）|
| 适用场景 | 快速存取 | **有序Map、范围查询** | 有序迭代/LRU |

---

## 七、实战场景

### 场景1：排行榜/区间统计

```java
// 统计成绩区间人数
TreeMap<Integer, List<String>> scoreMap = new TreeMap<>();
// 按分数有序存储学生

// 查询 80-90 分的学生
int count = scoreMap.subMap(80, 90).values()
                    .stream()
                    .mapToInt(List::size)
                    .sum();

// 查询最高分
int topScore = scoreMap.lastKey();
```

### 场景2：时间序列数据

```java
// 按时间有序存储日志
TreeMap<Long, String> logs = new TreeMap<>();
logs.put(System.currentTimeMillis(), "事件1");

// 查询某个时间点之后的所有日志
long since = ...;
SortedMap<Long, String> recent = logs.tailMap(since);

// 查询最近 N 条（从尾部取）
NavigableMap<Long, String> last5 = new TreeMap<>(logs).descendingMap();
// 取前5个键...
```

### 场景3：IP 段路由查找

```java
// 路由表（IP 段到出口的映射）
TreeMap<Long, String> routeTable = new TreeMap<>();
routeTable.put(ipToLong("192.168.0.0"), "网关A");
routeTable.put(ipToLong("10.0.0.0"), "网关B");

// 查找 192.168.1.100 应走哪个网关（最长前缀匹配）
Long targetIP = ipToLong("192.168.1.100");
Map.Entry<Long, String> route = routeTable.floorEntry(targetIP);
// 找到 <= 目标IP 的最大条目
```

---

## 八、面试要点速查

| 问题 | 要点 |
|------|------|
| TreeMap 底层实现 | 红黑树（自平衡二叉搜索树），所有操作 O(log n) |
| TreeMap 如何保证有序 | 键实现 Comparable，或构造时传入 Comparator |
| TreeMap 为什么不允许 null 键 | 键需要进行比较（compareTo），null 无法比较 |
| 红黑树的五条性质 | 节点红/黑；根黑；叶(NIL)黑；红节点子节点黑；任意节点到叶路径黑高相等 |
| 红黑树 vs AVL 树 | 红黑树弱平衡，旋转次数少，适合频繁增删；AVL 严格平衡，查找略快，适合读多 |
| NavigableMap 的优势 | 提供 floor/ceiling/higher/lower、subMap/headMap/tailMap 等范围操作 |
| TreeMap 适合什么场景 | 有序遍历、范围查询（如统计某区间的数据）、排行榜 |
