# main 方法为什么必须是 public static void main？

## 一句话说明（白话）
JVM需要一个固定入口，才能反射调用。

## 它解决什么问题 / 为什么重要
保证 Java 程序可被启动。

## 核心原理（一步步讲清楚）
- public：JVM 可访问。
- static：无需实例化对象。
- void：无返回值。
- main：约定名称。

##典型使用场景
- 应用启动入口。

## 简单例子 /伪代码
```java
public static void main(String[] args) {}
```

## 常见坑与误区
- 签名不正确导致无法启动。

##关联知识
- JVM 启动流程

## 延伸阅读（后续补充）
- Java 程序入口
