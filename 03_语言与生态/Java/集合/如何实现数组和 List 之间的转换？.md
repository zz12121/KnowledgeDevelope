# 如何实现数组和 List 之间的转换？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
- **数组 转 List**：
    - 使用 `Arrays.asList(T... a)`方法。但需要注意，返回的List并非 `java.util.ArrayList`，而是一个固定大小的视图，不支持添加(`add`)或删除(`remove`)操作，但可以修改元素。
    - 如果需要可变的List，可以这样创建：`new ArrayList<>(Arrays.asList(array))`。
- **List 转 数组**：
    - 使用 `List`的 `toArray()`方法。无参重载返回的是 `Object[]`类型。
    - 使用 `toArray(T[] a)`可以指定返回数组的类型，更常用。例如：`list.toArray(new String[0])`或 `list.toArray(new String[list.size()])`。

##关联知识
- 

## 延伸阅读（后续补充）
- 
