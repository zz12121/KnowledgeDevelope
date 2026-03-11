# HashMap 和 LinkedHashMap 的区别？

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
`LinkedHashMap`是 `HashMap`的子类。
- **`HashMap`**：不保证迭代顺序。
- **`LinkedHashMap`**：**维护着一个贯穿所有条目的双向链表**。这使得它可以保持元素的**插入顺序**（order of insertion）或**访问顺序**（order of access，即 LRU - Least Recently Used 算法的体现）。因此，在迭代 `LinkedHashMap`时，顺序是可以预测的。

##关联知识
- 

## 延伸阅读（后续补充）
- 
