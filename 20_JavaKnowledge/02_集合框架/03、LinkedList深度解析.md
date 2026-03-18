---
tags:
  - Java/集合框架
  - Java/List
aliases:
  - LinkedList双向链表
  - Deque实现
date: 2026-03-18
---

# LinkedList 深度解析

> **核心关键词**：双向链表、Deque、头尾操作、内存结构、与 ArrayList 对比

---

## 一、整体架构

### 1.1 继承关系与实现接口

```
AbstractSequentialList<E>
    └── LinkedList<E>
          实现：List<E>, Deque<E>, Cloneable, Serializable

LinkedList 同时实现了 List 和 Deque 接口：
- 作为 List：有序、可重复、支持按索引访问
- 作为 Deque：双端队列，支持头尾高效操作，可用作 Stack 和 Queue
```

### 1.2 双向链表结构

```
LinkedList 内部结构：

null ← [prev|item|next] ↔ [prev|item|next] ↔ [prev|item|next] → null
         ↑ first（head）                         ↑ last（tail）

核心字段：
  int size          // 节点数量
  Node<E> first     // 指向第一个节点（头节点）
  Node<E> last      // 指向最后一个节点（尾节点）
```

### 1.3 Node 节点

```java
private static class Node<E> {
    E item;
    Node<E> next;   // 指向后继节点
    Node<E> prev;   // 指向前驱节点
    
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

---

## 二、核心操作与源码

### 2.1 头部操作（O(1)）

```java
// ===== 作为 List 的头部操作 =====
list.add(0, element);        // 等价于 addFirst
list.remove(0);              // 等价于 removeFirst

// ===== 作为 Deque 的头部操作（推荐使用这些方法）=====
void addFirst(E e);          // 头部插入
E removeFirst();             // 移除并返回头节点（队列空时抛 NoSuchElementException）
E pollFirst();               // 移除并返回头节点（队列空时返回 null）
E getFirst();                // 返回头节点（不删除，队列空时抛异常）
E peekFirst();               // 返回头节点（不删除，队列空时返回 null）

// 头部插入源码
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);  // prev=null, next=旧head
    first = newNode;
    if (f == null)
        last = newNode;   // 链表原来为空，头尾都指向新节点
    else
        f.prev = newNode; // 旧头节点的 prev 指向新节点
    size++;
    modCount++;
}
```

### 2.2 尾部操作（O(1)）

```java
// ===== Deque 尾部操作 =====
void addLast(E e);           // 尾部插入（默认 add(E e) 等价于此）
E removeLast();              // 移除并返回尾节点
E pollLast();                // 移除尾节点（为空返回 null）
E getLast();                 // 返回尾节点（不删除）
E peekLast();                // 返回尾节点（不删除，为空返回 null）

// 尾部插入源码
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);  // prev=旧tail, next=null
    last = newNode;
    if (l == null)
        first = newNode;    // 链表原来为空
    else
        l.next = newNode;   // 旧尾节点的 next 指向新节点
    size++;
    modCount++;
}
```

### 2.3 按索引访问（O(n)）——LinkedList 的短板

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

// 折半查找：根据 index 与 size/2 的关系，从头或尾开始遍历
Node<E> node(int index) {
    if (index < (size >> 1)) {  // index 在前半段，从头遍历
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {                     // index 在后半段，从尾遍历
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

> ⚠️ 即使做了折半优化，最坏情况仍是 O(n/2) = O(n)，远不如 ArrayList 的 O(1)。**不适合频繁按索引访问**。

### 2.4 中间插入/删除（O(n)）

```java
// 指定位置插入：先 O(n) 找到节点，再 O(1) 插入
list.add(5, "element");

// unlink：删除节点（O(1)，但前提是已经拿到节点引用）
E unlink(Node<E> x) {
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;    // 删除头节点
    } else {
        prev.next = next;
        x.prev = null;   // help GC
    }

    if (next == null) {
        last = prev;     // 删除尾节点
    } else {
        next.prev = prev;
        x.next = null;   // help GC
    }

    x.item = null;       // help GC
    size--;
    modCount++;
    return element;
}
```

---

## 三、作为 Deque（双端队列）使用

### 3.1 Deque 方法对照表

| 操作 | Deque 方法（推荐）| Queue 方法（等价）| 异常行为 |
|------|-----------------|-----------------|---------|
| 头部插入 | `addFirst` / `offerFirst` | — | add 失败抛异常；offer 返回 false |
| 尾部插入 | `addLast` / `offerLast` | `add` / `offer` | — |
| 头部取出（删） | `removeFirst` / `pollFirst` | `remove` / `poll` | remove 失败抛异常；poll 返回 null |
| 头部查看（不删）| `getFirst` / `peekFirst` | `element` / `peek` | get/element 失败抛异常；peek 返回 null |
| 尾部取出（删） | `removeLast` / `pollLast` | — | — |
| 尾部查看（不删）| `getLast` / `peekLast` | — | — |

### 3.2 作为栈（Stack）使用

```java
// LinkedList 实现了 Deque，可以当栈用（比 java.util.Stack 更推荐）
Deque<String> stack = new LinkedList<>();

stack.push("a");   // 等价于 addFirst（入栈）
stack.push("b");
stack.push("c");

stack.pop();       // 返回 "c"，等价于 removeFirst（出栈）
stack.peek();      // 返回 "b"，不删除（查看栈顶）

// 更推荐：ArrayDeque 作为栈（无锁，比 LinkedList 快）
Deque<String> fastStack = new ArrayDeque<>();
```

### 3.3 作为队列（Queue）使用

```java
// 先进先出（FIFO）
Queue<String> queue = new LinkedList<>();

queue.offer("a");   // 尾部入队
queue.offer("b");
queue.offer("c");

queue.poll();       // 返回 "a"，头部出队
queue.peek();       // 返回 "b"，不删除

// 更推荐：ArrayDeque（性能更好）或 BlockingQueue（需要阻塞语义）
```

---

## 四、内存占用分析

```
ArrayList（存储 N 个对象引用）：
  Object[] 数组 = N * 4 字节（引用）+ 少量对象头/元数据
  内存紧凑

LinkedList（存储 N 个节点）：
  每个 Node = item（引用 4B）+ prev（引用 4B）+ next（引用 4B）+ 对象头（16B）
            ≈ 28~32 字节
  N 个节点 ≈ 28N ~ 32N 字节
  
对比：
  10000 个 Integer 对象：
  ArrayList:  40KB（引用数组）
  LinkedList: 280KB~320KB（每个节点额外 24B 开销）
  
LinkedList 内存开销约是 ArrayList 的 7-8 倍！
```

---

## 五、ArrayList vs LinkedList 深度对比

| 对比维度 | ArrayList | LinkedList |
|--------|-----------|------------|
| 底层结构 | 动态数组 | 双向链表 |
| 随机访问 get(i) | **O(1)** | O(n)（折半后 O(n/2)）|
| 头部插入/删除 | O(n)（移动元素）| **O(1)** |
| 尾部插入/删除 | O(1) 均摊 | **O(1)** |
| 中间插入/删除 | O(n) | O(n)（查找）+ O(1)（操作）|
| 内存占用 | **低**（仅引用数组）| 高（每节点额外 prev/next）|
| 缓存友好性 | **好**（数组连续内存）| 差（节点分散）|
| 实现 Deque | ❌ | **✅** |
| 扩容开销 | 偶发 O(n)（复制数组）| 无扩容 |
| 适用场景 | **读多写少，按索引访问** | **频繁头尾操作，实现队列/栈** |

**实际建议**：
- 99% 的场景用 **ArrayList**（内存更省，缓存友好，随机访问快）
- 需要双端队列时用 **ArrayDeque**（比 LinkedList 还快，内存也更省）
- LinkedList 适用场景：需要在**已知节点引用**的情况下频繁插入/删除

---

## 六、实战场景

### 场景1：实现 LRU 缓存（用 LinkedHashMap 更优）

```java
// 用 LinkedList 实现简单的 LRU（仅作演示，实际推荐用 LinkedHashMap）
class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K,V>> map = new HashMap<>();
    private final LinkedList<Node<K,V>> list = new LinkedList<>();
    
    record Node<K,V>(K key, V value) {}
    
    LRUCache(int capacity) { this.capacity = capacity; }
    
    V get(K key) {
        Node<K,V> node = map.get(key);
        if (node == null) return null;
        list.remove(node);
        list.addFirst(node);  // 移到头部（最近使用）
        return node.value();
    }
    
    void put(K key, V value) {
        Node<K,V> existing = map.get(key);
        if (existing != null) {
            list.remove(existing);
        } else if (list.size() == capacity) {
            Node<K,V> lru = list.removeLast();  // 移除最久未使用
            map.remove(lru.key());
        }
        Node<K,V> node = new Node<>(key, value);
        list.addFirst(node);
        map.put(key, node);
    }
}
```

### 场景2：undo/redo 操作栈

```java
// 编辑器的撤销/重做
Deque<String> undoStack = new LinkedList<>();
Deque<String> redoStack = new LinkedList<>();

void doAction(String action) {
    undoStack.push(action);
    redoStack.clear();  // 新操作清空 redo 栈
}

void undo() {
    if (!undoStack.isEmpty()) {
        String action = undoStack.pop();
        redoStack.push(action);
        // 执行撤销逻辑...
    }
}

void redo() {
    if (!redoStack.isEmpty()) {
        String action = redoStack.pop();
        undoStack.push(action);
        // 执行重做逻辑...
    }
}
```

---

## 七、面试要点速查

| 问题 | 要点 |
|------|------|
| LinkedList 底层结构 | 双向链表，每个节点含 item/prev/next |
| LinkedList 的时间复杂度 | 头尾操作 O(1)；按索引访问/中间插入 O(n) |
| LinkedList 和 ArrayList 如何选 | 读多用 ArrayList；频繁头尾操作用 Deque/ArrayDeque；很少选 LinkedList |
| LinkedList 实现了哪些接口 | List 和 Deque（双端队列） |
| node(index) 的优化 | 折半查找，index < size/2 从头遍历，否则从尾遍历，最多遍历 n/2 步 |
| 为什么不推荐用 LinkedList 作队列 | ArrayDeque 无锁 + 数组存储，缓存友好，性能更好 |
| LinkedList 的内存开销 | 每个节点约 28~32 字节，是 ArrayList 的 7-8 倍 |


---

**相关面试题** → [[../../10_Developlanguage/001_Java/02_JavaCollectionSubject/02、List 相关|02、List 相关]]
