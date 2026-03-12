# 什么是 happens-before 原则？

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
happens-before 是 JMM 中定义的两个操作之间的偏序关系，用于描述**操作之间的可见性**​。如果操作 A happens-before 操作 B，那么 A 操作的结果对 B 操作是**可见的**。
常见的 happens-before 规则包括：
- **程序次序规则**：同一线程内，书写在前面的操作 happens-before 后面的操作。
- **管程锁定规则**：对一个锁的 unlock 操作 happens-before 后面对同一个锁的 lock 操作。
- **volatile 变量规则**：对一个 volatile 变量的写操作 happens-before 后面对这个变量的读操作。
- **线程启动规则**：Thread 对象的 start() 方法调用 happens-before 此线程的每一个动作。
- **线程终止规则**：线程中的所有操作都 happens-before 对此线程的终止检测（如 join() 方法返回）。
- **传递性**：如果 A happens-before B，且 B happens-before C，那么 A happens-before C。

##关联知识
- 

## 延伸阅读（后续补充）
- 
