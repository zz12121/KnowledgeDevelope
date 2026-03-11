# ConcurrentHashMap 的实现原理？

## 一句话说明（白话）

hashCode 是对象散列标识，用于加速查找。

## 它解决什么问题 / 为什么重要

HashMap/HashSet先用 hashCode 定位桶，再用 equals 判断。

## 核心原理（一步步讲清楚）

equals 相等必须 hashCode 相等。

##典型使用场景

Map key、Set 去重。

## 简单例子 /伪代码

equals 基于 id，hashCode也应基于 id。

## 常见坑与误区

hashCode 相同不代表 equals 相同。

##题库要点（原始材料）
`ConcurrentHashMap`是 HashMap 的线程安全版本。
- **JDK 1.7**：采用 **分段锁（Segment）​机制。容器里包含一个 `Segment`数组，每个 `Segment`本身就是一个线程安全的哈希表。锁的粒度是段，不同的段上的操作可以并发进行。
- **JDK 1.8 及之后**：进行了重大优化，放弃了 `Segment`，改用 **`synchronized`锁桶数组的头节点**（链表头或红黑树根节点）的方式实现同步。同时结合 CAS（Compare-And-Swap） 操作，使得锁的粒度更细，并发性能更高。

##关联知识
- 

## 延伸阅读（后续补充）
- 
