# LinkedList集合

## 概述

LinkedList是Java集合框架中的双向链表实现，实现了List接口和Deque接口，既可以当作列表使用，也可以当作队列或栈使用。

## 核心特性

1. **双向链表**：每个节点包含前驱和后继指针
2. **插入删除高效**：在头部或尾部操作时间复杂度O(1)
3. **随机访问较慢**：需要遍历链表，时间复杂度O(n)
4. **非同步**：线程不安全

## 数据结构

```
Head <-> Node1 <-> Node2 <-> Node3 <-> Tail
       [prev|data|next]   [prev|data|next]
```

每个节点包含：
- **data**：存储的数据
- **prev**：前驱节点引用
- **next**：后继节点引用

## 操作复杂度

| 操作 | 时间复杂度 | 说明 |
|------|------------|------|
| addFirst | O(1) | 头部插入 |
| addLast | O(1) | 尾部插入 |
| removeFirst | O(1) | 头部删除 |
| removeLast | O(1) | 尾部删除 |
| get(index) | O(n) | 随机访问 |
| add(index, e) | O(n) | 指定位置插入 |

## 使用示例

```java
LinkedList<String> list = new LinkedList<>();

// 添加元素
list.add("A");
list.addFirst("B"); // 头部插入
list.addLast("C");  // 尾部插入

// 获取元素
String first = list.getFirst();
String last = list.getLast();

// 删除元素
list.removeFirst();
list.removeLast();

// 当作栈使用
list.push("X"); // 头部插入
String pop = list.pop(); // 头部删除

// 当作队列使用
list.offer("Y"); // 尾部插入
String poll = list.poll(); // 头部删除
```

## 与ArrayList对比

| 特性 | ArrayList | LinkedList |
|------|-----------|------------|
| 底层 | 动态数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 头部插入/删除 | O(n) | O(1) |
| 内存占用 | 较低（连续空间） | 较高（节点+指针） |
| 缓存命中率 | 高 | 低 |

## 适用场景

1. **队列/栈**：需要频繁在头尾操作
2. **随机访问少**：不需要频繁随机访问元素
3. **迭代为主**：主要是顺序遍历
4. **内存充足**：需要额外内存存储指针

## 不适用场景

1. **随机访问频繁**：大量get(index)操作
2. **内存敏感**：对内存占用要求高

## 总结

LinkedList通过双向链表实现了高效的插入和删除操作，适合队列、栈等场景。但随机访问性能较差，不适合需要频繁随机访问的场景。
