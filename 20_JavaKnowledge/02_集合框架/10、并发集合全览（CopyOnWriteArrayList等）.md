---
tags:
  - Java/集合框架
  - Java/并发
aliases:
  - CopyOnWriteArrayList
  - ConcurrentLinkedQueue
  - BlockingQueue
  - 并发安全集合
date: 2026-03-18
---

# 并发集合全览

> **核心关键词**：CopyOnWriteArrayList、ConcurrentLinkedQueue、BlockingQueue、ConcurrentSkipListMap、Collections.synchronizedXxx

---

## 一、为什么需要并发集合

```java
// ArrayList 在多线程下的问题
List<Integer> list = new ArrayList<>();
// 并发 add 可能导致：
//   1. 数据丢失（两个线程同时扩容）
//   2. 数组越界（size++ 非原子，导致覆盖）
//   3. 无限循环（JDK 7 HashMap 并发 resize 的经典 bug）

// 解决方案演进：
// 方案1：Collections.synchronized 包装（粗粒度锁，性能差）
// 方案2：java.util.concurrent 并发集合（专门为高并发设计）
```

---

## 二、并发集合全景图

```
java.util.concurrent（JUC）并发集合：

List 类：
  CopyOnWriteArrayList       → 读多写少，写时复制

Queue/Deque 类：
  ConcurrentLinkedQueue      → 无界非阻塞队列（CAS）
  ConcurrentLinkedDeque      → 无界非阻塞双端队列

BlockingQueue（阻塞队列）：
  ArrayBlockingQueue         → 有界，数组，公平/非公平锁
  LinkedBlockingQueue        → 有界/无界，链表，两把锁（读写分离）
  PriorityBlockingQueue      → 无界，优先级堆
  DelayQueue                 → 无界，延迟队列
  SynchronousQueue           → 零容量，直接传递
  LinkedTransferQueue        → 无界，TransferQueue 语义
  LinkedBlockingDeque        → 有界双端阻塞队列

Map 类：
  ConcurrentHashMap          → 高并发 HashMap（分段/CAS+synchronized）
  ConcurrentSkipListMap      → 有序并发 Map（跳表）

Set 类：
  CopyOnWriteArraySet        → 基于 CopyOnWriteArrayList
  ConcurrentSkipListSet      → 有序并发 Set（基于 ConcurrentSkipListMap）
```

---

## 三、CopyOnWriteArrayList

### 3.1 原理：写时复制

```java
// 核心思想：
// 读操作：直接读原数组，无锁，并发安全
// 写操作：加锁 → 复制整个数组 → 在副本上修改 → 替换引用

// 源码核心（add 方法）
public boolean add(E e) {
    synchronized (lock) {
        Object[] oldArray = getArray();
        int len = oldArray.length;
        Object[] newArray = Arrays.copyOf(oldArray, len + 1);  // 复制新数组
        newArray[len] = e;
        setArray(newArray);  // 用 volatile 写替换引用
        return true;
    }
}

// 读操作（无锁！）
public E get(int index) {
    return elementAt(getArray(), index);  // 读 volatile 数组引用
}
```

### 3.2 适用场景与局限

```java
// 适用：读多写少（订阅者列表、白名单、配置列表）
// 不适用：写频繁（每次写都复制整个数组，内存和 CPU 开销大）

// 注意：弱一致性（Weakly Consistent）
// 遍历时看到的是快照，不反映遍历过程中发生的写操作
// 不会抛 ConcurrentModificationException（与 ArrayList 不同）

CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>();
cowList.addAll(Arrays.asList("a", "b", "c"));

// 安全遍历：迭代器持有快照
for (String s : cowList) {
    cowList.add("new");  // 修改的是新副本，不影响当前迭代
}
```

---

## 四、阻塞队列（BlockingQueue）

### 4.1 核心接口

```java
// 四组操作（同样功能，不同的失败处理方式）
//                   抛异常     返回特殊值   阻塞        超时阻塞
// 插入              add()      offer()      put()       offer(e, time, unit)
// 移除              remove()   poll()       take()      poll(time, unit)
// 检查              element()  peek()       —           —

// put() / take() 是关键：会阻塞直到可以操作
BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);
queue.put("task");         // 队满时阻塞
String task = queue.take(); // 队空时阻塞
```

### 4.2 ArrayBlockingQueue

```java
// 有界、数组、一把 ReentrantLock（notFull + notEmpty 两个 Condition）
ArrayBlockingQueue<Integer> abq = new ArrayBlockingQueue<>(100);
// 可选公平锁（按等待顺序唤醒，吞吐量低但公平）
ArrayBlockingQueue<Integer> fair = new ArrayBlockingQueue<>(100, true);

// 生产者消费者经典场景
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(50);

// 生产者线程
new Thread(() -> {
    while (true) {
        Task task = generateTask();
        queue.put(task);  // 满了就阻塞
    }
}).start();

// 消费者线程
new Thread(() -> {
    while (true) {
        Task task = queue.take();  // 空了就阻塞
        process(task);
    }
}).start();
```

### 4.3 LinkedBlockingQueue

```java
// 链表实现，默认无界（Integer.MAX_VALUE），可指定容量
// 两把锁（putLock + takeLock）：读写可以同时进行，吞吐量高于 ArrayBlockingQueue
LinkedBlockingQueue<Integer> lbq = new LinkedBlockingQueue<>(1000);

// ThreadPoolExecutor 默认使用 LinkedBlockingQueue（无界！）
// Executors.newFixedThreadPool(n) 使用无界 LinkedBlockingQueue
// ⚠️ 生产环境危险：任务堆积可能导致 OOM
ExecutorService pool = new ThreadPoolExecutor(
    5, 10, 60, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000)  // 建议指定容量！
);
```

### 4.4 SynchronousQueue

```java
// 零容量队列：put 必须等 take，直接传递，无缓冲
// Executors.newCachedThreadPool() 使用 SynchronousQueue
SynchronousQueue<String> sq = new SynchronousQueue<>();

// 生产者：必须有消费者在等待，否则阻塞
new Thread(() -> sq.put("data")).start();
// 消费者：必须有生产者在等待，否则阻塞
String data = sq.take();
```

### 4.5 DelayQueue

```java
// 元素必须实现 Delayed 接口，只有到期的元素才能被 take
// 应用：定时任务、缓存过期清理、订单超时取消

public class DelayedTask implements Delayed {
    private final long executeAt;  // 执行时间（毫秒时间戳）
    private final String name;

    public DelayedTask(String name, long delayMs) {
        this.name = name;
        this.executeAt = System.currentTimeMillis() + delayMs;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(executeAt - System.currentTimeMillis(),
                            TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed other) {
        return Long.compare(this.executeAt,
                           ((DelayedTask) other).executeAt);
    }
}

DelayQueue<DelayedTask> delayQueue = new DelayQueue<>();
delayQueue.put(new DelayedTask("task1", 5000));  // 5秒后可取出

// 调度线程
new Thread(() -> {
    while (true) {
        DelayedTask task = delayQueue.take();  // 阻塞到有任务到期
        execute(task);
    }
}).start();
```

---

## 五、ConcurrentLinkedQueue

```java
// 无界、非阻塞、CAS 实现（Michael-Scott 无锁队列算法）
// 适合：高并发无界场景，不需要阻塞语义时

ConcurrentLinkedQueue<String> clq = new ConcurrentLinkedQueue<>();
clq.offer("a");          // 非阻塞，始终返回 true
String head = clq.poll(); // 非阻塞，队空返回 null

// 注意：size() 是 O(n) 的！不要在循环中调用
// 判断是否为空：用 isEmpty()，而非 size() == 0
```

---

## 六、ConcurrentSkipListMap / Set

```java
// 跳表实现：有序、无锁（CAS）、O(log n) 增删查
// 并发环境下替代 TreeMap/TreeSet

ConcurrentSkipListMap<String, Integer> skipMap = new ConcurrentSkipListMap<>();
skipMap.put("banana", 2);
skipMap.put("apple", 1);
skipMap.put("cherry", 3);
// 自动按 key 排序

// 导航方法（同 TreeMap）
skipMap.firstKey()          // "apple"
skipMap.lastKey()           // "cherry"
skipMap.headMap("cherry")   // [apple, banana]
skipMap.floorKey("b")       // "banana"

// 使用场景：排行榜、时间线索引、范围查询 + 高并发
```

---

## 七、Collections.synchronizedXxx（不推荐）

```java
// 用 synchronized(mutex) 包装每个方法，粗粒度加锁
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
Map<String, String> syncMap = Collections.synchronizedMap(new HashMap<>());

// 注意：迭代时仍需手动加锁！
synchronized (syncList) {
    for (String s : syncList) { ... }
}

// 与 JUC 集合的对比：
// synchronizedXxx：单锁，并发度 = 1，迭代不安全
// CopyOnWriteArrayList：读无锁，迭代安全（快照），适合读多写少
// ConcurrentHashMap：分段/CAS，并发度高，迭代弱一致性
```

---

## 八、如何选择并发集合

```
场景 → 推荐集合

高并发 Map（读写均衡）    → ConcurrentHashMap
高并发有序 Map            → ConcurrentSkipListMap
读远多于写的 List         → CopyOnWriteArrayList
有界生产者消费者           → ArrayBlockingQueue / LinkedBlockingQueue(有界)
无界非阻塞队列            → ConcurrentLinkedQueue
线程池任务队列            → LinkedBlockingQueue(有界) / SynchronousQueue
延迟/定时任务             → DelayQueue
优先级队列（并发）         → PriorityBlockingQueue
```

---

## 九、面试要点速查

| 问题 | 要点 |
|------|------|
| CopyOnWriteArrayList 原理 | 写时复制：写操作复制整个数组后修改，读操作无锁 |
| CopyOnWriteArrayList 缺点 | 内存开销大（写时复制全量）；弱一致性（迭代看快照）；不适合写频繁场景 |
| ArrayBlockingQueue vs LinkedBlockingQueue | ABQ：有界数组单锁；LBQ：链表双锁（读写分离），吞吐更高，默认无界（OOM风险）|
| SynchronousQueue 特点 | 零容量，直接传递，CachedThreadPool 使用它 |
| DelayQueue 原理 | PriorityQueue + ReentrantLock，take 阻塞到堆顶元素到期 |
| ConcurrentLinkedQueue vs LinkedBlockingQueue | CLQ：CAS 无锁，非阻塞；LBQ：ReentrantLock，支持阻塞 put/take |
| 为什么不用 Collections.synchronized | 粗粒度单锁，并发性能差；迭代还需额外手动加锁 |


---

**相关面试题** → [[../../10_Developlanguage/001_Java/02_JavaCollectionSubject/07、并发集合|02集合-07、并发集合]] | [[../../10_Developlanguage/001_Java/03_JavaConcurrencySubject/08、并发集合|并发-08、并发集合]]
