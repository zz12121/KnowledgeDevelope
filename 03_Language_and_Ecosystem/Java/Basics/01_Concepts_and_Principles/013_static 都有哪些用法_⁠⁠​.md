# static 都有哪些用法？

## 一句话说明（白话）
static 表示“属于类本身”，不是对象。

## 它解决什么问题 / 为什么重要
减少对象创建成本，方便共享状态或工具方法。

## 核心原理（一步步讲清楚）
- static 成员随类加载一次。
-通过类名访问，不依赖实例。

##典型使用场景
-常量（static final）。
-工具类方法（Math、Collections）。
-单例模式。

## 简单例子 /伪代码
```java
public static final int MAX =100;
public static void util() {}
```

## 常见坑与误区
-过多 static共享导致状态混乱。
- 静态方法无法访问实例成员。

##关联知识
- 类加载
- 单例模式

## 延伸阅读（后续补充）
- Java static关键字
