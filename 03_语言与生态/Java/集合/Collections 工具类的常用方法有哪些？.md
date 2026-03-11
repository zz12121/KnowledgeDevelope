# Collections 工具类的常用方法有哪些？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
`java.util.Collections`是一个纯粹的工具类，无法实例化，其所有方法都是静态的。它主要提供以下几类操作：

|功能类别|核心方法|作用描述|
|---|---|---|
|**排序与洗牌**​|`sort(List<T> list)`, `sort(list, Comparator)`|对列表排序（自然序或定制序）|
||`reverse(List<?> list)`|反转列表中元素的顺序|
||`shuffle(List<?> list)`|随机打乱列表元素顺序（常用于测试或游戏）|
|**查找与极值**​|`binarySearch(List, key)`|在**已排序**的列表中使用二分法查找，效率高（O(log n)）|
||`max(Collection)`, `min(Collection)`|返回集合中的最大/最小元素（基于自然顺序或比较器）|
||`frequency(Collection, Object)`|统计指定元素在集合中出现的次数|
|**修改与替换**​|`swap(List, int i, int j)`|交换列表中指定位置的元素|
||`replaceAll(List<T> list, T oldVal, T newVal)`|将列表中所有出现的某一指定值替换为另一值|
||`fill(List<? super T> list, T obj)`|使用指定元素填充列表的所有位置（**注意：会覆盖原有元素**）|
|**线程安全包装**​|`synchronizedList/Set/Map(...)`|返回一个线程安全的集合包装器。**注意：迭代时仍需手动同步**​|
|**不可变包装**​|`unmodifiableList/Set/Map(...)`|返回一个只读的集合视图。任何修改操作都会抛出 `UnsupportedOperationException`|
|**特殊集合创建**​|`emptyList()/emptySet()/emptyMap()`|返回一个不可变的空集合实例，避免返回`null`，推荐作为空结果返回|
||`singleton(T o)`, `singletonList(T o)`, `singletonMap(K, V)`|返回一个包含且仅包含一个指定元素的不可变集合，避免创建集合的开销|
|**批量添加**​|`addAll(Collection<? super T> c, T... elements)`|将所有指定元素添加到指定 collection 中，比多次调用`add`更方便|

**使用示例：**
```java
// 排序与查找
List<Integer> numbers = new ArrayList<>(Arrays.asList(3, 1, 4, 1, 5));
Collections.sort(numbers); // 列表变为 [1, 1, 3, 4, 5]
int index = Collections.binarySearch(numbers, 4); // 返回 3
int count = Collections.frequency(numbers, 1); // 返回 2

// 创建不可变空集合和单元素集合（安全返回的好习惯）
public List<String> getData() {
    // ... 如果没数据，返回空集合而非null，避免调用方空指针
    return Collections.emptyList();
}
public Set<String> getDefaultConfig() {
    return Collections.singleton("default_value");
}
```

##关联知识
- 

## 延伸阅读（后续补充）
- 
