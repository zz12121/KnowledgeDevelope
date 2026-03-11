# 为什么不推荐使用 Vector 和 Hashtable？

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
`Vector`和 `Hashtable`是Java早期的线程安全集合类，它们通过在几乎所有方法上添加 `synchronized`关键字来实现线程安全。这种方式虽然保证了线程安全，但是一种**粗粒度的锁策略**，导致每次只允许一个线程访问集合，严重限制了并发性能。在高并发场景下，这种同步开销巨大，性能远低于JUC包中的并发集合（如 `ConcurrentHashMap`）。

##关联知识
- 

## 延伸阅读（后续补充）
- 
