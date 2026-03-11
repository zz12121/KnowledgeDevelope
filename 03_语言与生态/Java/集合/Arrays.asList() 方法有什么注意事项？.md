# Arrays.asList() 方法有什么注意事项？

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
这个方法是将数组转换为List的便捷方式，但有一些重要的限制：
1. **固定大小**：返回的列表是基于原数组的**视图**，是一个固定大小的列表。你不能对该列表进行**添加(`add`)或删除(`remove`)​ 操作，否则会抛出 `UnsupportedOperationException`。
2. **反映修改**：对返回列表的修改（如`set`方法）会**直接反映到原数组**上，因为它们共享存储。
3. **基本类型数组问题**：如果传入一个基本数据类型（如`int[]`）的数组，`Arrays.asList()`会将其视为单个元素，返回`List<int[]>`，而不是期望的`List<Integer>`。应使用包装类数组。
**正确使用示例：**
```java
String[] strArray = {"Hello", "World"};
List<String> list = Arrays.asList(strArray); // ok
// list.add("Java"); // 错误！不支持添加操作
list.set(0, "Hi"); // 正确，修改元素，同时strArray[0]也会变成"Hi"

// 对于基本类型数组，应使用如下方式
int[] intArray = {1, 2, 3};
List<Integer> realList = Arrays.stream(intArray).boxed().collect(Collectors.toList());
```

##关联知识
- 

## 延伸阅读（后续补充）
- 
