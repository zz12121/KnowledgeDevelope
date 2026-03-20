# TreeSet集合

## 概述

TreeSet是Java集合框架中的一个实现类，基于红黑树（Red-Black Tree）实现，实现了SortedSet接口，可以对元素进行自然排序或自定义排序。

## 核心特性

1. **有序**：元素按照自然顺序或Comparator顺序排列
2. **唯一**：不允许重复元素
3. **非同步**：线程不安全
4. **底层实现**：红黑树

## 工作原理

### 红黑树

TreeSet底层使用红黑树实现，是一种自平衡的二叉搜索树：
- 每个节点要么是红色，要么是黑色
- 根节点是黑色
- 叶子节点（NIL）是黑色
- 红色节点的子节点必须是黑色
- 从任一节点到其每个叶子的路径上黑色节点数量相同

### 操作复杂度

| 操作 | 时间复杂度 |
|------|------------|
| 添加 | O(log n) |
| 删除 | O(log n) |
| 查询 | O(log n) |
| 遍历 | O(n) |

## 使用示例

### 自然排序

```java
TreeSet<Integer> set = new TreeSet<>();
set.add(5);
set.add(2);
set.add(8);
// 输出: [2, 5, 8]
System.out.println(set);
```

### 自定义排序

```java
TreeSet<String> set = new TreeSet<>(new Comparator<String>() {
    @Override
    public int compare(String s1, String s2) {
        return s2.compareTo(s1); // 降序
    }
});
```

## 常用方法

```java
// 获取第一个元素
set.first();

// 获取最后一个元素
set.last();

// 获取小于等于指定元素的最大的元素
set.floor(e);

// 获取大于等于指定元素的最小的元素
set.ceiling(e);

// 获取子集
set.subSet(fromElement, toElement);
```

## 与HashSet对比

| 特性 | HashSet | TreeSet |
|------|---------|---------|
| 排序 | 无序 | 有序 |
| 时间复杂度 | O(1) | O(log n) |
| 底层 | 哈希表 | 红黑树 |
| 内存占用 | 较低 | 较高 |

## 适用场景

1. 需要元素自动排序
2. 需要快速查找、插入、删除
3. 需要范围查询
4. 不需要线程安全

## 总结

TreeSet通过红黑树实现了元素的有序存储和高效查找，适合需要排序和范围查询的场景。
