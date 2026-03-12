# Integer a=127 与 Integer b=127 相等吗？

## 一句话说明（白话）
`Integer` 有缓存，`==` 比较的是引用；127 在缓存范围内，所以 `a==b` 通常为 `true`。

## 它解决什么问题 / 为什么重要
避免对包装类型误用 `==`，导致偶发性逻辑错误。

## 核心原理（一步步讲清楚）
- `Integer` 是对象，`==` 比较**引用地址**。
- `Integer.valueOf()` 对 -128~127 做了缓存，装箱会复用缓存对象。
- 超出范围会创建新对象，因此 `==`可能为 `false`。

##典型使用场景
-需要把基本类型放入集合。
-处理可能为 `null` 的数值。

## 简单例子 /伪代码
```java
Integer a =127, b =127;
System.out.println(a == b); // true（缓存）

Integer c =128, d =128;
System.out.println(c == d); // false

System.out.println(a.equals(b)); // true
```

## 常见坑与误区
- 对 `Integer` 使用 `==`进行值比较。
- 自动拆箱遇到 `null`触发 NPE。

##关联知识
- 自动装箱/拆箱
- equals vs ==
- IntegerCache

## 延伸阅读（后续补充）
- Java 包装类型缓存机制
