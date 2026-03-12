# Object 的常用方法有哪些？

## 一句话说明（白话）
Object 提供基础方法：equals/hashCode/toString 等。

## 它解决什么问题 / 为什么重要
所有类都继承，定义对象基本行为。

## 核心原理（一步步讲清楚）
- `equals` 比较。
- `hashCode` 哈希。
- `toString` 字符串表示。
- `wait/notify`线程通信。

##典型使用场景
- 重写 equals/hashCode/toString。

## 简单例子 /伪代码
```java
@Override public String toString(){...}
```

## 常见坑与误区
- 重写 equals 不重写 hashCode。

##关联知识
- equals/hashCode
-线程通信

## 延伸阅读（后续补充）
- Java Object 方法
