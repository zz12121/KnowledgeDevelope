# 两个对象值相同 (x.equals (y) == true)，但却可有不同的 Hash Code，这句话对不对？⁠​

## 一句话说明（白话）

== 比较引用地址，equals 比较内容（类重写后）。

## 它解决什么问题 / 为什么重要

避免把对象地址当内容比较；集合、去重场景很关键。

## 核心原理（一步步讲清楚）

Object.equals 默认等同 ==；重写需配合 hashCode 一致性。

##典型使用场景

字符串、值对象比较。

## 简单例子 /伪代码

String a="x"; String b=new String("x"); a==b false, a.equals(b) true。

## 常见坑与误区

重写 equals 不重写 hashCode。

##题库要点（原始材料）
**不对。这句话严重违反了`equals`和`hashCode`方法的契约。
正确的规定是：如果`x.equals(y) == true`，那么`x.hashCode() == y.hashCode()`必须为`true`。**
反之则不一定：如果两个对象的hashCode相同，它们不一定equals。
如果违反了这条规则，当把这个对象放入基于哈希的集合（如HashSet）时，会导致无法正确找到对象等严重问题。因此，**当你重写`equals`方法时，必须同时重写`hashCode`方法**，以确保契约成立。

##关联知识
- 

## 延伸阅读（后续补充）
- 
