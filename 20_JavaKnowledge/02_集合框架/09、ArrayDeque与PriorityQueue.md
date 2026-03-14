# ArrayDeque 与 PriorityQueue

> **核心关键词**：双端队列、栈、队列、堆、优先级队列、最大堆、最小堆、Deque、offer/poll/peek

---

## 一、队列体系概览

```
Queue（队列，FIFO）
  ├── LinkedList          → 链表实现，允许 null，通用
  ├── ArrayDeque          → 数组实现双端队列，不允许 null，性能更好
  │     （同时可用作 Stack 替代品）
  └── PriorityQueue       → 堆实现，按优先级出队（不保证全局顺序）

Deque（双端队列，两端都可入队出队）
  ├── ArrayDeque
  └── LinkedList
```

---

## 二、ArrayDeque 深度解析

### 2.1 底层结构

```java
// 底层：循环数组（circular array / ring buffer）
// 用 head 和 tail 两个指针标记头尾
// 扩容：容量翻倍，复制数组

// 初始容量：16（指定容量时向上取 2 的幂）
// 不允许存 null（用 null 作为空槽标记）

ArrayDeque<Integer> deque = new ArrayDeque<>();
// 或指定初始容量（减少扩容）
ArrayDeque<Integer> deque2 = new ArrayDeque<>(64);
```

### 2.2 作为队列（Queue）使用

```java
Queue<String> queue = new ArrayDeque<>();

// 入队
queue.offer("first");   // 推荐：队满时返回 false（ArrayDeque 不限容，实际不会失败）
queue.add("second");    // 队满时抛 IllegalStateException

// 出队
String head = queue.poll();   // 推荐：队空时返回 null
String head2 = queue.remove(); // 队空时抛 NoSuchElementException

// 查看队头（不移除）
String peek = queue.peek();   // 队空返回 null
String peek2 = queue.element(); // 队空抛异常

// offer/poll/peek 是推荐 API（不抛异常，更安全）
```

### 2.3 作为双端队列（Deque）使用

```java
Deque<String> deque = new ArrayDeque<>();

// 队头操作
deque.offerFirst("a");   // 从头部入队
deque.addFirst("b");
deque.pollFirst();       // 从头部出队
deque.peekFirst();       // 查看头部

// 队尾操作
deque.offerLast("c");    // 从尾部入队（等同于 offer）
deque.pollLast();        // 从尾部出队
deque.peekLast();        // 查看尾部

// 应用：实现滑动窗口（单调双端队列）
// 维护单调递减双端队列，记录窗口内最大值的索引
```

### 2.4 作为栈（Stack）使用——替代 java.util.Stack

```java
// 官方推荐用 ArrayDeque 代替 Stack（Stack 是线程安全的，有加锁开销）
Deque<String> stack = new ArrayDeque<>();

stack.push("a");    // = addFirst，压栈
stack.push("b");
stack.push("c");

stack.pop();        // = removeFirst，弹栈 → "c"
stack.peek();       // = peekFirst，查看栈顶 → "b"

System.out.println(stack);  // [b, a]（头部是栈顶）
```

### 2.5 ArrayDeque vs LinkedList

| 对比点 | ArrayDeque | LinkedList |
|--------|-----------|-----------|
| 底层 | 循环数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 头尾插删 | O(1)（均摊） | O(1) |
| 缓存友好 | ✅ 数组连续 | ❌ 节点分散 |
| null 支持 | ❌ | ✅ |
| 内存占用 | 较小 | 较大（每节点额外指针） |
| **推荐使用** | **队列/栈首选** | 需要 null 或中间插删 |

---

## 三、PriorityQueue 深度解析

### 3.1 底层结构：二叉堆

```java
// PriorityQueue 底层是 最小堆（min-heap）
// poll() 始终返回最小元素
// 不保证全局有序，只保证堆顶最小
// 不允许 null

// 默认：最小堆（自然升序）
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(5);
minHeap.offer(2);
minHeap.offer(8);
minHeap.offer(1);
System.out.println(minHeap.peek());   // 1（最小值）
System.out.println(minHeap.poll());   // 1
System.out.println(minHeap.poll());   // 2

// 自定义比较器：最大堆
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.offer(5);
maxHeap.offer(2);
maxHeap.offer(8);
System.out.println(maxHeap.peek());   // 8（最大值）
```

### 3.2 堆操作原理（sift up / sift down）

```
最小堆特性：父节点 ≤ 子节点（对所有节点成立）
数组存储：index i 的左子节点 = 2i+1，右子节点 = 2i+2，父节点 = (i-1)/2

offer(e)：
  1. 将元素添加到数组末尾
  2. sift up（上浮）：与父节点比较，如果更小则交换，直到满足堆性质

poll()：
  1. 取出堆顶（index 0）
  2. 将最后一个元素放到堆顶
  3. sift down（下沉）：与较小的子节点比较，如果更大则交换

时间复杂度：
  offer/poll：O(log n)
  peek：O(1)
  contains/remove（任意元素）：O(n)
```

### 3.3 初始化与扩容

```java
// 初始容量 11，扩容：
//   容量 < 64 时：翻倍 + 2
//   容量 ≥ 64 时：扩容 50%

// 从集合初始化（使用 heapify，O(n) 时间，比逐个 offer 的 O(n log n) 快）
List<Integer> list = Arrays.asList(5, 2, 8, 1, 9);
PriorityQueue<Integer> pq = new PriorityQueue<>(list);  // O(n) heapify
```

### 3.4 自定义对象的优先队列

```java
// 方式1：元素实现 Comparable
public class Task implements Comparable<Task> {
    int priority;
    String name;

    @Override
    public int compareTo(Task other) {
        return Integer.compare(this.priority, other.priority);  // 小优先级先出
    }
}

// 方式2：构造时传入 Comparator（更灵活）
PriorityQueue<Task> taskQueue = new PriorityQueue<>(
    Comparator.comparingInt((Task t) -> t.priority).reversed()  // 大优先级先出
);
```

### 3.5 经典算法应用

```java
// 应用1：Top K 最大元素（维护大小为 K 的最小堆）
public int[] topKFrequent(int[] nums, int k) {
    // 用最小堆维护 K 个最大值
    PriorityQueue<Integer> minHeap = new PriorityQueue<>(k);
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) minHeap.poll();  // 弹出最小的
    }
    return minHeap.stream().mapToInt(i -> i).toArray();
}

// 应用2：合并 K 个有序链表（最小堆）
PriorityQueue<ListNode> pq = new PriorityQueue<>(
    Comparator.comparingInt(n -> n.val));
for (ListNode head : lists) {
    if (head != null) pq.offer(head);
}
ListNode dummy = new ListNode(0), cur = dummy;
while (!pq.isEmpty()) {
    ListNode node = pq.poll();
    cur = cur.next = node;
    if (node.next != null) pq.offer(node.next);
}

// 应用3：Dijkstra 最短路径
PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[0]));
// [距离, 节点]
pq.offer(new int[]{0, src});

// 应用4：延迟任务调度（按执行时间排序）
PriorityQueue<Task> scheduler = new PriorityQueue<>(
    Comparator.comparingLong(t -> t.executeAt));
```

---

## 四、单调队列（Monotonic Deque）— 进阶

```java
// 经典问题：滑动窗口最大值
// 维护单调递减双端队列，队头始终是窗口内最大值

public int[] maxSlidingWindow(int[] nums, int k) {
    Deque<Integer> deque = new ArrayDeque<>();  // 存索引
    int[] result = new int[nums.length - k + 1];

    for (int i = 0; i < nums.length; i++) {
        // 移除窗口外的元素（队头超出窗口左边界）
        while (!deque.isEmpty() && deque.peekFirst() < i - k + 1) {
            deque.pollFirst();
        }
        // 维护单调递减：移除所有比当前元素小的（它们不可能成为最大值）
        while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
            deque.pollLast();
        }
        deque.offerLast(i);
        // 窗口形成后记录结果
        if (i >= k - 1) result[i - k + 1] = nums[deque.peekFirst()];
    }
    return result;
}
```

---

## 五、面试要点速查

| 问题 | 要点 |
|------|------|
| ArrayDeque vs Stack | ArrayDeque 无锁，性能更好；Stack 继承 Vector，方法加 synchronized |
| ArrayDeque vs LinkedList | ArrayDeque 缓存友好，内存效率高；推荐优先用 ArrayDeque |
| PriorityQueue 底层 | 二叉最小堆，数组存储，offer/poll O(log n) |
| PriorityQueue 保证全局有序吗 | 不保证，只保证每次 poll 返回最小值（堆顶） |
| 最大堆如何实现 | 传入 `Comparator.reverseOrder()` |
| Top K 问题用什么堆 | 求最大 K 个 → 用大小为 K 的最小堆；求最小 K 个 → 用大小为 K 的最大堆 |
| 滑动窗口最大值算法 | 单调递减双端队列（ArrayDeque），O(n) 时间 |


---

**相关面试题** → [[../../10_Developlanguage/001_java/02_JavaCollectionSubject/05、Queue 相关|05、Queue 相关]]
