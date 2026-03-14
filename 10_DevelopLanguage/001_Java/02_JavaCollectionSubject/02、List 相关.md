###### 1. ArrayList 和 LinkedList 的区别是什么？

[[../../../20_JavaKnowledge/02_集合框架/02、ArrayList深度解析#六、ArrayList vs LinkedList vs Vector|📖]]
这是面试必考题，核心区别就是**底层数据结构不同**，由此带来性能特性的巨大差异。

**ArrayList** 底层是动态数组，内存是连续的。随机访问（按索引取元素）极快，O(1) 直接定位。但在中间插入或删除元素时需要移动后面所有的元素，所以中间插删性能差。

**LinkedList** 底层是双向链表，每个节点存着数据以及前后节点的引用。在任意位置插入或删除只需改指针，O(1) 就搞定。但随机访问得从头或尾遍历，O(n) 的时间复杂度，查询慢。另外因为每个节点多存了两个引用，内存占用比 ArrayList 大。LinkedList 还实现了 `Deque` 接口，可以当栈、队列或双端队列用。

**选择原则**：绝大多数情况用 ArrayList。LinkedList 真正的优势场景是频繁在头尾操作（队列场景），或者需要一个 Deque 的时候。很多人以为"频繁中间插删就用 LinkedList"，实际上因为数组的 CPU 缓存命中率更高，ArrayList 的表现往往不比 LinkedList 差。

###### 2. ArrayList 和 Vector 的区别是什么？

[[../../../20_JavaKnowledge/02_集合框架/02、ArrayList深度解析#六、ArrayList vs LinkedList vs Vector|📖]]
两者底层都是数组，区别主要在线程安全和扩容策略上。

**线程安全**：Vector 的所有公共方法都加了 `synchronized`，是线程安全的，但每次操作都要加锁，性能开销大。ArrayList 没有做同步，是非线程安全的，但性能更好。

**扩容策略**：ArrayList 扩容为原来的 1.5 倍，Vector 默认扩容为原来的 2 倍（可以通过构造参数设置扩容步长）。

**出现时间**：Vector 是 JDK 1.0 的老古董，ArrayList 是 JDK 1.2 才引入的。

**结论**：现在基本不用 Vector 了。不需要线程安全直接用 ArrayList，需要线程安全用 `CopyOnWriteArrayList` 或 `Collections.synchronizedList()`，都比 Vector 强。

###### 3. ArrayList 的扩容机制是怎样的？初始容量是多少？

[[../../../20_JavaKnowledge/02_集合框架/02、ArrayList深度解析#三、扩容机制|📖]]
ArrayList 的扩容设计很巧妙：

**初始容量**：调用无参构造 `new ArrayList()` 的时候，底层数组初始化为一个空数组，**并不是直接创建容量为10的数组**。真正的初始化发生在第一次 add 元素时，此时容量才被设为 10。这种懒加载的设计避免了空集合浪费内存。

**触发扩容**：每次 add 元素前都会检查容量是否够用，不够就触发扩容。

**扩容过程**：核心是 `grow()` 方法，新容量 = 旧容量 + 旧容量 >> 1，也就是**大约增长为原来的 1.5 倍**。然后调用 `Arrays.copyOf()` 把原数组数据复制到新数组。这个复制操作是扩容最耗时的地方，时间复杂度 O(n)。

**最佳实践**：如果能提前知道大概要存多少数据，用 `new ArrayList(initialCapacity)` 指定初始容量，或者调用 `ensureCapacity()` 提前扩容。这样可以避免频繁扩容复制，性能更好。

###### 4. 如何实现数组和 List 之间的转换？

[[../../../20_JavaKnowledge/02_集合框架/02、ArrayList深度解析#四、核心操作|📖]]
**数组 → List**：用 `Arrays.asList(数组)` 方法。但有个大坑要注意：它返回的不是真正的 `java.util.ArrayList`，而是 Arrays 内部的一个固定大小的 List，**不支持 add 和 remove**，调用会报 UnsupportedOperationException。如果需要可变的 List，要包一层：`new ArrayList<>(Arrays.asList(array))`。

**List → 数组**：用 `list.toArray()` 方法。无参版本返回 `Object[]`，一般不用。推荐用泛型版本：`list.toArray(new String[0])`，传个空数组进去，JVM 会自动分配合适大小，这是官方推荐写法。

###### 5. ArrayList 和 LinkedList 的使用场景分别是什么？

[[../../../20_JavaKnowledge/02_集合框架/02、ArrayList深度解析#八、实战场景|📖]]
**绝大多数情况用 ArrayList**。因为数组内存连续，CPU 缓存命中率高，遍历和随机访问都非常快。即便有少量中间插删，ArrayList 也不一定比 LinkedList 慢——数据量小的时候内存复制很快，况且链表还有额外的节点分配开销。

**考虑用 LinkedList 的情况**：
- 需要频繁在头部操作（addFirst、removeFirst），链表头部操作是 O(1)
- 需要把 List 当作栈、队列或双端队列用（它实现了 Deque 接口）

一个容易犯的错误是觉得"我要频繁插入中间，所以用 LinkedList"，但其实找到插入位置本身就需要 O(n) 遍历，LinkedList 并不会快多少。

###### 6. ArrayList 的线程安全问题如何解决？CopyOnWriteArrayList 的原理和应用场景？

[[../../../20_JavaKnowledge/02_集合框架/10、并发集合全览（CopyOnWriteArrayList等）#三、CopyOnWriteArrayList|📖]]
ArrayList 是非线程安全的，多线程同时修改会出问题，解决方案主要三种：

**方案一**：`Collections.synchronizedList(list)` 包装一下，所有方法加 `synchronized` 同步。简单粗暴，但性能差，每次操作都要锁。

**方案二**：用 `CopyOnWriteArrayList`。这是 JUC 包里提供的写时复制 List，原理是：**每次写操作（add、set、remove）都不直接改原数组，而是先复制一份新数组，在新数组上改，改完再把引用指向新数组**。读操作完全无锁，直接读当前数组。

优点是读非常快，不加锁，读读之间不互斥，非常适合**读多写少**的场景，比如配置列表、白名单之类的。

缺点也很明显：每次写都要复制整个数组，内存开销大；而且读到的可能是稍微旧一点的数据（最终一致性，不是强一致性）。写频繁的场景不适合用。

**方案三**：用 `synchronized` 或者 `ReentrantLock` 手动加锁，适合需要精细控制的场景。

###### 7. LinkedList 可以作为栈和队列使用吗？如何使用？

[[../../../20_JavaKnowledge/02_集合框架/03、LinkedList深度解析#三、作为 Deque（双端队列）使用|📖]]
完全可以，LinkedList 实现了 `Deque`（双端队列）接口，天生就是为这种场景设计的。

**当栈用（LIFO，后进先出）**：
- 入栈：`push(e)` 或 `addFirst(e)`
- 出栈：`pop()` 或 `removeFirst()`
- 看栈顶：`peek()` 或 `peekFirst()`

**当队列用（FIFO，先进先出）**：
- 入队：`offer(e)` 或 `addLast(e)`
- 出队：`poll()` 或 `removeFirst()`
- 看队头：`peek()` 或 `peekFirst()`

不过，如果只是需要栈或队列功能，更推荐用 `ArrayDeque`，它基于数组实现，在绝大多数场景比 LinkedList 性能更好，内存占用也更小。

###### 8. subList 方法返回的是什么？使用时需要注意什么？

[[../../../20_JavaKnowledge/02_集合框架/02、ArrayList深度解析#四、核心操作|📖]]
`subList(fromIndex, toIndex)` 返回的是原 List 的一个**视图**，不是独立的新 List。

这个"视图"的意思是：子列表和原列表共享同一份底层数据。对子列表做非结构性修改（比如 `set`）会直接反映到原列表上，反之亦然。

**最大的坑**：如果在拿到子列表之后，对**原列表**做了结构性修改（add、remove），再操作子列表就会抛 `ConcurrentModificationException`。

**最佳实践**：如果想要一份与原列表无关的独立副本，要这样写：`new ArrayList<>(originalList.subList(from, to))`。
