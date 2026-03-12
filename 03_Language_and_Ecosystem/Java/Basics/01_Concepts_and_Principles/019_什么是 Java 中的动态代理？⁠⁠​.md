# 什么是 Java 中的动态代理？

## 一句话说明（白话）
动态代理是在运行时生成代理类，用来增强方法调用。

## 它解决什么问题 / 为什么重要
实现 AOP、日志、事务等横切逻辑而不侵入业务代码。

## 核心原理（一步步讲清楚）
- JDK 动态代理基于接口和反射。
- CGLIB 基于字节码生成子类。

##典型使用场景
- Spring AOP、RPC 框架。

## 简单例子 /伪代码
```java
Proxy.newProxyInstance(loader, interfaces, handler);
```

## 常见坑与误区
- JDK代理只能代理接口。
- CGLIB不能代理 final 类/方法。

##关联知识
-反射
- CGLIB
- AOP

## 延伸阅读（后续补充）
- Java 动态代理机制
