# subList 方法返回的是什么？使用时需要注意什么？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
`List`的 `subList(int fromIndex, int toIndex)`方法返回的是原列表的一个**视图**（View），而非一个新的独立List对象。
这意味着：
- 对子列表的非结构性修改（如`set`方法）会反映到原列表上。
- 对原列表的结构性修改（如直接调用`add`, `remove`）会导致后续对子列表的任何操作（包括简单的遍历）抛出 `ConcurrentModificationException`异常。
**最佳实践**：如果你需要一份与原列表无关的副本，应该这样创建：`new ArrayList<>(originalList.subList(from, to))`。

##关联知识
- 

## 延伸阅读（后续补充）
- 
