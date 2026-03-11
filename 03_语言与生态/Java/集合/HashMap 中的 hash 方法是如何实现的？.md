# HashMap 中的 hash 方法是如何实现的？

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
`hash(Object key)`方法并非直接使用 key 的 `hashCode()`，而是进行了一次“扰动处理”：
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
- **右移16位并异或**：将 `hashCode`的高16位与低16位进行异或运算。目的是**将高位的特征混合到低位中**。
- **原因**：因为计算索引时是 `(n-1) & hash`，当 n 较小时，实际上只有哈希值的低位参与了运算。如果直接使用 `hashCode`，高位的变化完全不影响索引，容易造成冲突。扰动后，高位的特征也参与了进来，从而**降低了哈希冲突的概率**​。

##关联知识
- 

## 延伸阅读（后续补充）
- 
