# TreeSet 的排序规则是什么？

## 一句话说明（白话）

这是一个 Java关键概念/特性，用于解释语言规则或运行机制。

## 它解决什么问题 / 为什么重要

帮助理解规范与最佳实践，避免常见错误。

## 核心原理（一步步讲清楚）

说明语法/机制，再解释运行时表现与影响。

##典型使用场景

面试常问点、日常开发高频使用。

## 简单例子 /伪代码

给出最小示例说明用法。

## 常见坑与误区

列出1-2个易错点。

##题库要点（原始材料）
`TreeSet`的排序规则主要有两种：
1. **自然排序 (Natural Ordering)**：
    - 添加到 `TreeSet`中的元素类必须实现 `java.lang.Comparable`接口。
    - `TreeSet`会通过调用元素的 `compareTo(Object o)`方法来比较元素的大小，并据此进行排序。
    - 如果尝试添加未实现 `Comparable`接口的对象，会在运行时抛出 `ClassCastException`。
2. **定制排序 (Custom Ordering)**：
    - 在创建 `TreeSet`时，可以传入一个实现了 `java.util.Comparator`接口的比较器对象。
    - `TreeSet`会使用这个 `Comparator`的 `compare(Object o1, Object o2)`方法来比较元素，而不再要求元素类实现 `Comparable`接口。这为排序规则提供了更大的灵活性。
**重要原则**：为了保证排序的准确性，**`compareTo`（或 `compare`）方法与 `equals`方法的逻辑必须保持一致**。即当 `x.compareTo(y) == 0`时，应有 `x.equals(y)`为 `true`。

##关联知识
- 

## 延伸阅读（后续补充）
- 
