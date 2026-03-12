# transient关键字的作用是什么？

## 一句话说明（白话）
transient 表示该字段不参与序列化。

## 它解决什么问题 / 为什么重要
避免敏感字段或临时字段被持久化。

## 核心原理（一步步讲清楚）
- Java 序列化会忽略 transient 字段。
-反序列化后该字段为默认值。

##典型使用场景
-密码、缓存、临时状态。

## 简单例子 /伪代码
```java
transient String password;
```

## 常见坑与误区
- transient不能阻止 JSON 序列化库（只对 Java 序列化生效）。

##关联知识
- Serializable
- 序列化/反序列化

## 延伸阅读（后续补充）
- Java 序列化机制
