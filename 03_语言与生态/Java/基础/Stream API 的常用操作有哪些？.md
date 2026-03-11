# Stream API 的常用操作有哪些？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
Stream的操作分为两类：**中间操作**（返回Stream，可连接）和**终端操作**（返回具体结果或副作用，触发执行）。
- **常用中间操作**：
    - `filter(Predicate p)`：过滤元素。
    - `map(Function f)`：将元素映射为另一种形式。
    - `sorted()`/ `sorted(Comparator com)`：排序。
    - `distinct()`：去重。
- **常用终端操作**：
    - `forEach(Consumer c)`：遍历每个元素。
    - `collect(Collector c)`：将流转换为集合（如List、Set、Map）。
    - `reduce(...)`：将流中的元素反复组合，得到一个值（如求和、求最大）。
    - `count()`：返回流中元素个数。
    - `findFirst()`/ `anyMatch(Predicate p)`：查找或匹配元素。

##关联知识
- 

## 延伸阅读（后续补充）
- 
