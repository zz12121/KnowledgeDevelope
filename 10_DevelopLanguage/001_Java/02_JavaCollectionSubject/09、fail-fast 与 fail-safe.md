###### 1. 什么是 fail-fast 机制？什么是 fail-safe 机制？在 Java 集合中是如何实现的？

[[../../../20_JavaKnowledge/02_集合框架/02、ArrayList深度解析#五、fail-fast 机制|📖]]
这两个机制描述的是集合在**遍历过程中被修改时如何反应**。

**Fail-Fast（快速失败）**：一旦检测到在遍历过程中集合被结构性修改，立即抛出 `ConcurrentModificationException`，让你马上知道出问题了。代表：`ArrayList`、`HashMap`、`HashSet` 等 `java.util` 包下的非线程安全集合都是这个机制。

实现原理是靠**修改计数器 modCount**：集合内部维护一个 modCount 字段，每次结构性修改（add、remove 等）都让它加1。创建迭代器时，迭代器会记下此刻的 modCount 值（叫 expectedModCount）。每次调用 `next()` 前，都检查当前 modCount 是否等于 expectedModCount，不等就说明有人改过集合，立即抛异常。

**Fail-Safe（安全失败）**：迭代器在创建时基于集合的**快照或副本**工作，不会因为原集合被修改而抛异常。代表：`CopyOnWriteArrayList`、`ConcurrentHashMap` 等 JUC 包下的并发集合。

CopyOnWriteArrayList 的迭代器基于创建时的数组快照，所以遍历的是旧数据，不会感知到后续的写操作。ConcurrentHashMap 的迭代器是**弱一致性**的，会尽量反映迭代过程中的修改，但不保证全部可见，也绝不会抛 ConcurrentModificationException。

**两者权衡**：Fail-Fast 强一致性但不允许并发修改；Fail-Safe 允许并发修改但可能读到旧数据。根据场景选择，多线程环境用 Fail-Safe 的并发集合。

###### 2. ConcurrentModificationException 异常是什么？如何避免？

[[../../../20_JavaKnowledge/02_集合框架/10、并发集合全览（CopyOnWriteArrayList等）#三、CopyOnWriteArrayList|📖]]
这个异常是 fail-fast 机制的具体体现，中文叫"并发修改异常"，但单线程也会触发。

**典型触发场景**：

单线程里用 for-each 或显式 Iterator 遍历集合，同时调用集合自身的 add/remove 修改结构：
```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
for (String s : list) {
    if ("B".equals(s)) {
        list.remove(s); // 💥 抛 ConcurrentModificationException
    }
}
```

多线程里一个线程在遍历，另一个线程在修改。

**如何避免**：见下一题的几种解决方案。

###### 3. 如何在遍历集合时删除元素？

[[../../../20_JavaKnowledge/02_集合框架/02、ArrayList深度解析#五、fail-fast 机制|📖]]
这是高频实战题，有5种方式，根据场景选：

**方式一：用 Iterator 的 remove()** — 单线程标准解法。迭代器自己的 remove() 会同步更新 expectedModCount，不会触发异常：
```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if ("B".equals(it.next())) {
        it.remove(); // 安全，同步了 modCount
    }
}
```

**方式二：用 removeIf()（推荐）** — Java 8 的现代写法，最简洁，内部已经处理好了一切：
```java
list.removeIf(item -> "B".equals(item)); // 一行搞定
```

**方式三：用 fail-safe 并发集合** — 多线程场景用 `CopyOnWriteArrayList` 或 `ConcurrentHashMap`，它们的迭代器不会抛 ConcurrentModificationException：
```java
List<String> list = new CopyOnWriteArrayList<>(Arrays.asList("A", "B", "C"));
for (String item : list) {
    if ("B".equals(item)) {
        list.remove(item); // 安全，但本次循环不会立即看到变化
    }
}
```

**方式四：逆向遍历删除（ArrayList 专用技巧）** — 从后往前用索引遍历，删后面的元素不影响前面的索引：
```java
for (int i = list.size() - 1; i >= 0; i--) {
    if ("B".equals(list.get(i))) {
        list.remove(i); // 安全
    }
}
```

**方式五：用 Stream 过滤生成新集合** — 不修改原集合，而是过滤出一个新的：
```java
List<String> newList = originalList.stream()
    .filter(item -> !"B".equals(item))
    .collect(Collectors.toList());
```

**选哪种**：单线程推荐 `removeIf()`，最简单；多线程用并发集合；想要不可变原集合用 Stream 过滤。
