# HashCode 的作用是什么？

## 一句话说明（白话）
hashCode 用于快速定位对象在哈希结构中的位置。

## 它解决什么问题 / 为什么重要
提高 HashMap/HashSet 的查找性能。

## 核心原理（一步步讲清楚）
- 哈希值决定桶位置。
- 冲突时再用 equals 判等。

##典型使用场景
- HashMap/HashSet。

## 简单例子 /伪代码
- `map.put(key, value)`先看 hashCode。

## 常见坑与误区
-只重写 equals 不重写 hashCode。

##关联知识
- equals/hashCode

## 延伸阅读（后续补充）
- HashMap 原理
