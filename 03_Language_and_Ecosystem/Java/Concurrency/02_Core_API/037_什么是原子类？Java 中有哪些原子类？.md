# 什么是原子类？Java 中有哪些原子类？

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
原子类通过**硬件级别的CAS（Compare-And-Swap）操作**和 **`volatile`变量**来保证单个共享变量操作的原子性、可见性和有序性。即使在多线程环境下，对原子类实例的每一次操作（如自增、比较并交换）都是不可分割的，从而避免了数据不一致的问题。
Java中的原子类主要位于 `java.util.concurrent.atomic`包下，可以根据其作用对象分为以下几大类：

|类别|核心类|说明|
|---|---|---|
|**基本类型**​|`AtomicInteger`, `AtomicLong`, `AtomicBoolean`|用于原子性操作对应的基本数据类型。|
|**数组类型**​|`AtomicIntegerArray`, `AtomicLongArray`, `AtomicReferenceArray`|原子性地更新数组中的单个元素。|
|**引用类型**​|`AtomicReference`, `AtomicStampedReference`, `AtomicMarkableReference`|用于原子性更新对象引用。后两者可解决ABA问题。|
|**字段更新器**​|`AtomicIntegerFieldUpdater`, `AtomicLongFieldUpdater`, `AtomicReferenceFieldUpdater`|以线程安全的方式更新已有类中的volatile字段。|
|**累加器**​|`LongAdder`, `DoubleAdder`, `LongAccumulator`, `DoubleAccumulator`|高并发场景下专为求和、求最大值等操作设计，性能通常优于AtomicLong。|

##关联知识
- 

## 延伸阅读（后续补充）
- 
