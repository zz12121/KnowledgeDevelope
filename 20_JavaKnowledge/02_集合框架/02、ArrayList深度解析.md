---
tags:
  - Java/集合框架
  - Java/List
aliases:
  - ArrayList扩容
  - 动态数组
date: 2026-03-18
---

# ArrayList 深度解析

> **核心关键词**：动态数组、扩容机制、RandomAccess、System.arraycopy、fail-fast

---

## 一、整体架构

### 1.1 继承关系

```
Iterable<E>
    └── Collection<E>
            └── List<E>
                    └── AbstractList<E>
                            └── ArrayList<E>
                                    实现了：Serializable, Cloneable, RandomAccess
```

**`RandomAccess` 接口**：标记接口（无方法），表示该集合支持**高效随机访问**（按索引访问是 O(1)）。`Collections.binarySearch()`、`Collections.sort()` 等方法会判断是否实现了 `RandomAccess` 来选择算法。

### 1.2 核心字段

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    
    // 默认初始容量：10
    private static final int DEFAULT_CAPACITY = 10;
    
    // 空数组（用于容量为0时）
    private static final Object[] EMPTY_ELEMENTDATA = {};
    
    // 默认容量的空数组（与 EMPTY_ELEMENTDATA 区分，用于判断第一次 add 时的扩容策略）
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    
    // 实际存储数据的数组（transient：不直接序列化，自定义序列化只保存有效元素）
    transient Object[] elementData;
    
    // 实际存储的元素个数
    private int size;
    
    // 最大数组大小（Integer.MAX_VALUE - 8，-8 是为了存储 header 信息）
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
}
```

---

## 二、初始化策略

```java
// 方式1：默认构造器（推荐，延迟分配）
// 创建时 elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA（空数组）
// 第一次 add 时才分配容量为 10 的数组（懒初始化，节省内存）
ArrayList<String> list1 = new ArrayList<>();

// 方式2：指定初始容量（已知大概数量时推荐，避免扩容）
ArrayList<String> list2 = new ArrayList<>(100);

// 方式3：从集合构造
ArrayList<String> list3 = new ArrayList<>(existingCollection);
```

**为什么区分 `EMPTY_ELEMENTDATA` 和 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA`？**

```java
// add 第一个元素时的扩容决策
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // 默认构造器：第一次扩容到 DEFAULT_CAPACITY(10)，而不是 1
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;  // 指定容量为0时：按需扩容（1→2→3...）
}
```

---

## 三、扩容机制

### 3.1 扩容流程

```mermaid
flowchart TD
    A[add element] --> B{size >= elementData.length?}
    B -->|否| C[直接写入 elementData[size++]]
    B -->|是| D[grow 扩容]
    D --> E[新容量 = 旧容量 * 1.5]
    E --> F{新容量 < minCapacity?}
    F -->|是| G[新容量 = minCapacity]
    F -->|否| H{新容量 > MAX_ARRAY_SIZE?}
    H -->|是| I[hugeCapacity 处理]
    H -->|否| J[Arrays.copyOf 复制数组]
    G --> J
    I --> J
    J --> C
```

### 3.2 关键源码

```java
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    // 新容量 = 旧容量 + 旧容量 / 2 = 旧容量 * 1.5
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    
    // Arrays.copyOf 内部调用 System.arraycopy（本地方法，效率高）
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

**扩容示例**（默认构造器）：

```
初始：[]（DEFAULTCAPACITY_EMPTY_ELEMENTDATA）
add 第1个：扩容到 max(10, 1) = 10
add 到第10个：不扩容
add 第11个：扩容 10 → 15
add 到第15个：不扩容
add 第16个：扩容 15 → 22（15 + 15/2 = 22）
...
```

### 3.3 预设容量的最佳实践

```java
// 已知会存入 1000 个元素，如何避免多次扩容？
// 理想初始容量：1000（正好够用）
// 但更保险：预留 1/3 的余量
ArrayList<String> list = new ArrayList<>(1000);

// 使用 Guava 的计算工具
// Lists.newArrayListWithCapacity(1000)
// Lists.newArrayListWithExpectedSize(1000)  → 内部做了缓冲计算
```

---

## 四、核心操作

### 4.1 add 操作

```java
// 尾部添加：O(1) 均摊（偶发扩容时 O(n)）
list.add("element");

// 指定位置插入：O(n)（需要移动后续元素）
list.add(2, "insert");
// 内部：System.arraycopy(elementData, 2, elementData, 3, size - 2);
//       将 index 2 之后的元素整体向右移一位
```

### 4.2 remove 操作

```java
// 按索引删除：O(n)（需要移动后续元素）
list.remove(2);
// 内部：System.arraycopy(elementData, 3, elementData, 2, numMoved);
//       然后 elementData[--size] = null;  ← 清除尾部引用，帮助 GC

// 按值删除：O(n)（先线性查找，再移动）
list.remove("element");  // ⚠️ 删除第一个匹配的元素

// ⚠️ 陷阱：删除 Integer 时的歧义
List<Integer> nums = new ArrayList<>(Arrays.asList(1, 2, 3));
nums.remove(1);            // ← 按索引删除，删除的是下标1的元素（值为2）！
nums.remove(Integer.valueOf(1)); // ← 按值删除，删除值为1的元素
```

### 4.3 get/set 操作

```java
// get：O(1)，直接数组随机访问
E element = list.get(index);  // → elementData[index]（带范围检查）

// set：O(1)，直接数组写入
E old = list.set(index, newElement);  // → elementData[index] = element

// 内部实现（包含边界检查）
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

### 4.4 批量操作

```java
// addAll：批量添加（一次扩容 + 一次 System.arraycopy，比多次 add 高效）
list.addAll(anotherCollection);
list.addAll(2, anotherCollection);  // 从指定位置插入

// removeAll：删除所有在给定集合中的元素
list.removeAll(anotherCollection);  // 内部做了优化：一次遍历过滤

// retainAll：只保留在给定集合中的元素
list.retainAll(anotherCollection);
```

---

## 五、fail-fast 机制

### 5.1 什么是 fail-fast？

使用迭代器遍历时，如果集合在迭代过程中被**结构性修改**（增加/删除元素），会立即抛出 `ConcurrentModificationException`：

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));

// ❌ 触发 ConcurrentModificationException
for (String s : list) {
    if (s.equals("b")) {
        list.remove(s);  // 修改了 modCount，迭代器检测到不一致
    }
}
```

### 5.2 实现原理

```java
// ArrayList 维护 modCount（继承自 AbstractList）
// 每次结构性修改（add/remove/clear 等）都会 ++modCount

// 迭代器在创建时记录 expectedModCount = modCount
// 每次 next() 和 remove() 时检查 modCount == expectedModCount
private class Itr implements Iterator<E> {
    int cursor;
    int lastRet = -1;
    int expectedModCount = modCount;  // 记录快照
    
    public E next() {
        checkForComodification();     // ← 检查
        // ...
    }
    
    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
    
    // 迭代器自身的 remove 是安全的（会同步更新 expectedModCount）
    public void remove() {
        // ...
        ArrayList.this.remove(lastRet);
        expectedModCount = modCount;  // ← 同步更新，不会抛异常
    }
}
```

### 5.3 安全的遍历删除

```java
// ✅ 方式1：迭代器的 remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) {
        it.remove();  // 安全
    }
}

// ✅ 方式2：倒序遍历（for 循环）
for (int i = list.size() - 1; i >= 0; i--) {
    if (list.get(i).equals("b")) {
        list.remove(i);  // 删除后不影响前面的索引
    }
}

// ✅ 方式3：removeIf（JDK 8+，推荐）
list.removeIf(s -> s.equals("b"));

// ✅ 方式4：先收集要删除的元素，再 removeAll
List<String> toRemove = list.stream()
                             .filter(s -> s.equals("b"))
                             .collect(Collectors.toList());
list.removeAll(toRemove);

// ❌ 方式5：普通 for-each（会触发 CME）
for (String s : list) {
    if (...) list.remove(s);  // ❌
}
```

---

## 六、ArrayList vs LinkedList vs Vector

| 特性 | ArrayList | LinkedList | Vector |
|------|-----------|------------|--------|
| 底层结构 | **动态数组** | 双向链表 | 动态数组 |
| 随机访问 | **O(1)** | O(n) | O(1) |
| 头部插入/删除 | O(n)（移动元素）| **O(1)** | O(n) |
| 尾部插入 | O(1) 均摊 | O(1) | O(1) 均摊 |
| 中间插入/删除 | O(n) | O(n)（查找）+O(1)（操作）| O(n) |
| 内存占用 | 低（只存数据）| 高（每节点含 prev/next 指针）| 低 |
| 线程安全 | ❌ | ❌ | ✅（方法 synchronized，性能差）|
| 扩容 | 1.5 倍 | 不涉及 | 2 倍 |
| 适用场景 | **大多数场景（读多）**| 频繁头尾操作 | 不推荐（用 CopyOnWriteArrayList）|

---

## 七、序列化优化

```java
// elementData 用 transient 修饰，不直接序列化
// 自定义 writeObject/readObject，只序列化有效元素（避免序列化 null 元素）

private void writeObject(java.io.ObjectOutputStream s) throws java.io.IOException {
    int expectedModCount = modCount;
    s.defaultWriteObject();  // 序列化非 transient 字段（size）
    s.writeInt(size);        // 写入容量
    
    // 只写入有效元素（size 个），不写空槽
    for (int i = 0; i < size; i++) {
        s.writeObject(elementData[i]);
    }
    // ...
}
```

---

## 八、实战场景

### 场景1：大数据量初始化

```java
// ❌ 逐个 add，可能触发多次扩容
List<String> list = new ArrayList<>();
for (String item : largeDataSet) {
    list.add(item);
}

// ✅ 预设容量
List<String> list = new ArrayList<>(largeDataSet.size());
list.addAll(largeDataSet);  // 一次性 addAll，内部一次扩容 + 一次 arraycopy
```

### 场景2：线程安全的 List

```java
// ❌ ArrayList 线程不安全

// ✅ CopyOnWriteArrayList（读多写少场景）
List<String> list = new CopyOnWriteArrayList<>();
// 写时复制：每次修改都复制整个数组，写开销大但读完全无锁

// ✅ Collections.synchronizedList（全量加锁，性能差）
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
// 注意：迭代时仍需手动同步
synchronized (syncList) {
    for (String s : syncList) { ... }
}
```

---

## 九、面试要点速查

| 问题 | 要点 |
|------|------|
| ArrayList 底层 | `Object[]` 数组，支持随机访问 O(1) |
| 默认初始容量 | 默认构造器创建时是空数组，第一次 add 时扩容到 10 |
| 扩容倍数 | 1.5 倍（`newCapacity = oldCapacity + oldCapacity >> 1`）|
| 插入到中间的时间复杂度 | O(n)（需要移动后续元素）|
| fail-fast 机制 | modCount 快照 + 迭代时检查，结构修改则抛 CME |
| 遍历时安全删除 | 迭代器 remove / removeIf / 倒序 for 循环 |
| ArrayList vs LinkedList | 随机访问用 ArrayList（O(1)）；频繁头尾操作用 LinkedList |
| 为什么 elementData 是 transient | 自定义序列化，只序列化有效元素，不序列化 null 槽 |


---

**相关面试题** → [[../../10_Developlanguage/001_Java/02_JavaCollectionSubject/02、List 相关|02、List 相关]]
