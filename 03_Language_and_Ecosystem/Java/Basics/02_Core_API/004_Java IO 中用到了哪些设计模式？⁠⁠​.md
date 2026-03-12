# Java IO 常用的设计模式有哪些？

## 一句话说明（白话）
装饰器模式最常见，配合适配器/桥接。

## 它解决什么问题 / 为什么重要
让 IO组合灵活、可扩展。

## 核心原理（一步步讲清楚）
- InputStream/Reader体系层层包装。
-通过装饰器增强功能。

##典型使用场景
- BufferedInputStream 包装 FileInputStream。

## 简单例子 /伪代码
```java
new BufferedInputStream(new FileInputStream(f))
```

## 常见坑与误区
-过度包装导致性能问题。

##关联知识
- 装饰器模式

## 延伸阅读（后续补充）
- Head First设计模式
