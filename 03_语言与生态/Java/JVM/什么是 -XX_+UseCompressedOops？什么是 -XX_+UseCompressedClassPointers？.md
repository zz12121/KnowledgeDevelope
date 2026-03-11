# 什么是 -XX:+UseCompressedOops？什么是 -XX:+UseCompressedClassPointers？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
指针压缩技术这是 64 位 JVM 上减少内存占用的重要技术。
- **-XX:+UseCompressedOops**：启用**普通对象指针压缩**。在 64 位系统中，一个引用指针原本占 8 字节，开启后压缩为 4 字节，显著节省堆内存。该参数在 JDK 6 之后默认开启。
- **-XX:+UseCompressedClassPointers**：启用**类指针压缩**。压缩对象头中指向类元数据的指针。**注意**：此参数依赖于 `UseCompressedOops`，必须在 `UseCompressedOops`开启时才能生效。

##关联知识
- 

## 延伸阅读（后续补充）
- 
