# HashMap 如何解决哈希冲突？

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
HashMap 主要使用**拉链法**（Separate Chaining）来解决哈希冲突。当多个键映射到同一个桶时，它们会以链表的形式存储在该桶中。当链表过长影响查询效率时，会进一步转换为红黑树以提升性能。

##关联知识
- 

## 延伸阅读（后续补充）
- 
