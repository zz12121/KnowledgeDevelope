###### 1. 什么是 fail-fast 机制？什么是 fail-safe 机制？在 Java 集合中是如何实现的？
为了快速把握核心区别，下表对比了这两种机制的关键特性：

| 特性         | Fail-Fast（快速失败）                                             | Fail-Safe（安全失败）                                                            |
| ---------- | ----------------------------------------------------------- | -------------------------------------------------------------------------- |
| **核心原则**​  | 一旦检测到并发修改，立即抛出`ConcurrentModificationException`异常，使程序快速失败。  | 允许在遍历过程中修改原集合，不会抛出异常。迭代器基于创建时的集合**副本**或**快照**工作，因此可能无法反映遍历开始后对原集合的所有修改。    |
| **实现基础**​  | 基于集合内部的**修改计数器**（`modCount`）实现。                             | 基于**写时复制**（如`CopyOnWriteArrayList`）或**弱一致性迭代器**（如`ConcurrentHashMap`）实现。   |
| **性能特点**​  | 检测开销小，但迭代过程中集合不能被结构性修改。                                     | 写操作（如添加、删除）通常有额外开销（如复制数组），但读操作性能很高。                                        |
| **数据一致性**​ | 强一致性，迭代器总能访问到最新的修改。                                         | 弱一致性，迭代器反映的是创建时刻的集合状态，可能读取到过时数据。                                           |
| **代表集合**​  | `ArrayList`, `HashMap`, `HashSet`等大部分`java.util`包下的非线程安全集合。 | `CopyOnWriteArrayList`, `ConcurrentHashMap`等`java.util.concurrent`包下的并发集合。 |

**Fail-Fast 的实现原理**
以`ArrayList`为例，其内部维护了一个 `modCount`（修改次数）字段。任何会改变集合结构的操作（如`add`, `remove`）都会使`modCount`加1。
当您通过`list.iterator()`获取迭代器时，迭代器会记录下当前集合的`modCount`值，存为`expectedModCount`。在迭代过程中，每次调用`next()`方法时，它都会检查当前的`modCount`是否与`expectedModCount`相等。如果不相等，则说明在迭代期间集合的结构被意外修改了，此时迭代器会立即抛出`ConcurrentModificationException`异常。
###### 2. ConcurrentModificationException 异常是什么？如何避免？
这个异常是**Fail-Fast机制的具体表现**，它是一个运行时异常，在迭代器正常遍历集合的过程中，集合的结构被改变了（但不是通过迭代器自身的`remove`方法）。
**触发该异常的典型场景包括：**
- **单线程环境**：在增强for循环（其底层也是使用迭代器）或显式使用迭代器遍历集合时，直接调用集合自身的`add`, `remove`等方法修改集合结构
```java
    List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
    for (String s : list) {
        if ("B".equals(s)) {
            list.remove(s); // 触发 ConcurrentModificationException
        }
    }
    ```
- **多线程环境**：一个线程正在迭代集合，另一个线程修改了该集合的结构。
###### 3. 如何在遍历集合时删除元素？
了解了异常产生的原因，我们就可以有针对性地避免它。以下是几种安全地在遍历时删除元素的方法，各有其适用场景。
**1. 使用迭代器自身的 remove() 方法**
这是**在单线程环境下，对`ArrayList`、`HashSet`等Fail-Fast集合进行遍历删除时的标准做法和最佳实践**。迭代器的`remove()`方法会在删除元素后，自动更新它内部记录的`expectedModCount`，从而与集合的`modCount`保持同步，避免了异常的发生。
```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
Iterator<String> iterator = list.iterator();
while (iterator.hasNext()) {
    String item = iterator.next();
    if ("B".equals(item)) {
        iterator.remove(); // 安全删除！
    }
}
System.out.println(list); // 输出: [A, C]
```
**2. 使用 Java 8 的 removeIf() 方法**
这个方法非常简洁高效，它内部已经处理了迭代和删除的所有细节，是**单线程环境下更现代、更推荐的做法**​。
```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
list.removeIf(item -> "B".equals(item)); // 一行代码搞定
System.out.println(list); // 输出: [A, C]
```
**3. 使用 Fail-Safe 并发集合（适用于多线程场景）**
如果你的应用场景是**多线程高并发**的，那么最根本的解决方案是使用`java.util.concurrent`包下的线程安全集合。
- **`CopyOnWriteArrayList`**：适用于**读多写少**的场景。每次写操作（如add, remove）都会复制底层数组的一个新副本，在副本上修改，然后替换掉旧的数组引用。它的迭代器基于创建时的旧数组快照工作，因此不会抛出异常，但也无法感知迭代开始后的新修改。
```java
    List<String> list = new CopyOnWriteArrayList<>(Arrays.asList("A", "B", "C"));
    for (String item : list) { // 即使在遍历时...
        if ("B".equals(item)) {
            list.remove(item); // ...删除是安全的，但本次循环不会立即看到变化
        }
    }
    ```
- **`ConcurrentHashMap`**：采用更复杂的锁机制（如分段锁或CAS操作）来保证并发安全，其迭代器是**弱一致性**的，它会尽力反映迭代开始后集合的更新，但不保证能反映所有修改，也绝不会抛出`ConcurrentModificationException`。
**4. 逆向遍历并删除（适用于ArrayList）**
这是一种针对`ArrayList`的特性（基于数组，尾部删除效率高）的**单线程场景下的技巧**。从后往前遍历，然后使用集合的`remove(int index)`方法删除，可以避免因为元素移动导致的索引错乱问题，且不会触发Fast-Fail（因为未使用迭代器）。
```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
for (int i = list.size() - 1; i >= 0; i--) {
    if ("B".equals(list.get(i))) {
        list.remove(i);
    }
}
```
**5. 使用 Stream API 过滤（适用于创建新集合）**
如果你想**创建一个不包含特定元素的新集合**，而不是修改原集合，Java 8的Stream API提供了非常优雅的函数式解决方案。
```java
List<String> originalList = new ArrayList<>(Arrays.asList("A", "B", "C"));
List<String> newList = originalList.stream()
                                  .filter(item -> !"B".equals(item))
                                  .collect(Collectors.toList()); // 创建新集合
System.out.println(newList); // 输出: [A, C]
```