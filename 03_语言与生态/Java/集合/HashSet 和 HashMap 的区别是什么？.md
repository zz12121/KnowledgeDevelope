# HashSet 和 HashMap 的区别是什么？

## 一句话说明（白话）

hashCode 是对象散列标识，用于加速查找。

## 它解决什么问题 / 为什么重要

HashMap/HashSet先用 hashCode 定位桶，再用 equals 判断。

## 核心原理（一步步讲清楚）

equals 相等必须 hashCode 相等。

##典型使用场景

Map key、Set 去重。

## 简单例子 /伪代码

equals 基于 id，hashCode也应基于 id。

## 常见坑与误区

hashCode 相同不代表 equals 相同。

##题库要点（原始材料）
|特性|HashSet|HashMap|
|---|---|---|
|**实现接口**​|实现 `Set`接口|实现 `Map`接口|
|**存储内容**​|存储**单个对象**​|存储**键值对 (Key-Value pairs)**​|
|**添加方法**​|`add(element)`|`put(key, value)`|
|**重复性保证**​|通过存储的**元素本身**保证唯一性|通过 **Key**​ 保证唯一性，**Value**​ 可以重复|
|**底层依赖**​|基于 `HashMap`实现|是独立的实现，是 `HashSet`的基础|

简单来说，**`HashSet`本身就是基于 `HashMap`的一个特化和封装**，它利用 `Map`的 Key 唯一性来实现了 Set 的元素不重复特性。

##关联知识
- 

## 延伸阅读（后续补充）
- 
