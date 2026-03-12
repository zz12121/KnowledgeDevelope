# 有没有可能两个不相等的对象有相同的 HashCode？

## 一句话说明（白话）
可能。不同对象也可能碰撞出相同 hashCode。

## 它解决什么问题 / 为什么重要
理解哈希冲突，避免误用 hashCode作为唯一标识。

## 核心原理（一步步讲清楚）
- hashCode 是 int，范围有限。
- 不同对象映射到同一值属于正常现象。

##典型使用场景
- HashMap/HashSet 冲突处理。

## 简单例子 /伪代码
- 不同字符串可能出现相同 hashCode。

## 常见坑与误区
- 把 hashCode 当作对象唯一ID。

##关联知识
- equals/hashCode
- 哈希冲突

## 延伸阅读（后续补充）
- Java HashMap 原理
