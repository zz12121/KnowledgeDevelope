# Optional 类的作用是什么？如何使用？

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
`Optional`的核心思想不是替换所有的`null`，而是作为一种通信工具，明确表示一个返回值可能为空，强制调用方进行处理。
- **创建Optional**：
    - `Optional.of(value)`：创建一个非空值的Optional。
    - `Optional.ofNullable(value)`：创建一个可能为`null`的Optional。
    - `Optional.empty()`：创建一个空的Optional。
- **使用Optional**（应避免直接调用 `get()`，因其在为空时会抛异常）：
    - `isPresent()`：判断值是否存在。
    - `ifPresent(Consumer c)`：如果值存在，则执行给定的操作。
    - `orElse(T other)`：如果值存在则返回，否则返回指定的默认值。
    - `orElseGet(Supplier other)`：延迟版本的`orElse`，只有在值为空时才调用Supplier。
    - `orElseThrow(Supplier exceptionSupplier)`：如果值为空，则抛出指定的异常。

##关联知识
- 

## 延伸阅读（后续补充）
- 
