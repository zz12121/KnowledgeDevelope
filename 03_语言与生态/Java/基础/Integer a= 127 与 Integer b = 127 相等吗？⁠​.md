# Integer a= 127 与 Integer b = 127 相等吗？⁠​

## 一句话说明（白话）

基本类型和包装类型行为不同，装箱拆箱有开销。

## 它解决什么问题 / 为什么重要

避免自动装箱导致的性能问题或 NPE。

## 核心原理（一步步讲清楚）

包装类型是对象，可为 null；== 在对象上比较引用。

##典型使用场景

集合只能存对象，需要使用包装类。

## 简单例子 /伪代码

Integer a=127,b=127; a==b true（缓存）；Integer a=128,b=128; a==b false。

## 常见坑与误区

对 null进行自动拆箱会抛 NPE。

##题库要点（原始材料）
**相等（使用 `==`比较结果为 `true`）。**
**原因是 Integer 类的缓存机制（Integer Cache）**。默认情况下，Integer 类会缓存 **-128 到 127**之间的所有整数对象。
当通过自动装箱（即直接赋值）或调用 Integer.valueOf(int i) 方法创建 Integer 对象时，如果数值在这个范围内，就会直接返回缓存池中已存在的同一个对象的引用。
```java
Integer a = 127;// 相当于 Integer.valueOf(127)
Integer b = 127;// 相当于 Integer.valueOf(127)
System.out.println(a == b);// true，因为 a 和 b 指向缓存中的同一个对象
```
**但是，如果数值超出缓存范围，结果就不同了**：
```java
Integer c = 128;
Integer d = 128;
System.out.println(c == d);// false，因为 128 超出了默认缓存范围，会创建新的 Integer 对象
```
**要点**：
- 这种缓存机制是一种性能优化，避免频繁创建和销毁小整数对象。
- ==使用 `new Integer(int)`构造器会强制创建新对象，不会使用缓存==。
- 其他包装类也有类似的缓存机制，如 `Byte`缓存所有值（范围是-128 到 127），`Short`、`Long`缓存 -128 到 127，`Character`缓存 0 到 127，`Boolean`缓存 `TRUE`和 `FALSE`。
- 可以使用 `XX:AutoBoxCacheMax=<size>`JVM 参数来调整 Integer 缓存的上限 。

##关联知识
- 

## 延伸阅读（后续补充）
- 
