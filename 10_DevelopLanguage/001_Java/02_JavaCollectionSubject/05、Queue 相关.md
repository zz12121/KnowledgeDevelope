###### 1. Queue 接口的常用方法有哪些？

[[../../../20_JavaKnowledge/02_集合框架/09、ArrayDeque与PriorityQueue#一、队列体系概览|📖]]
Queue 接口的方法都是成对出现的，同一个操作有两个版本：一个在失败时抛异常，另一个在失败时返回特殊值（null 或 false）。

**添加元素**：
- `add(e)`：队列满时抛 `IllegalStateException`
- `offer(e)`：队列满时返回 false（更常用，不用 try-catch）

**移除并返回队头**：
- `remove()`：队列空时抛 `NoSuchElementException`
- `poll()`：队列空时返回 null（更常用）

**查看队头（不移除）**：
- `element()`：队列空时抛 `NoSuchElementException`
- `peek()`：队列空时返回 null（更常用）

**记忆口诀**：`offer/poll/peek` 这三个不抛异常，日常开发优先用这组。

###### 2. PriorityQueue 的实现原理是什么？

[[../../../20_JavaKnowledge/02_集合框架/09、ArrayDeque与PriorityQueue#三、PriorityQueue 深度解析|📖]]
PriorityQueue 是基于**最小堆**的优先级队列，出队顺序不是插入顺序，而是**优先级顺序**（默认最小元素先出）。

**底层结构**：用一个动态数组来模拟完全二叉树。数组下标 `i` 的节点，其父节点在 `(i-1)/2`，左孩子在 `2i+1`，右孩子在 `2i+2`。

**核心操作**：
- **入队（offer）**：新元素放到数组末尾，然后执行**上浮（siftUp）**——不断和父节点比较，如果比父节点优先级高就交换，直到堆性质恢复。O(log n)。
- **出队（poll）**：弹出堆顶（最高优先级），把末尾元素移到堆顶，然后执行**下沉（siftDown）**——不断和优先级更高的子节点交换，直到堆性质恢复。O(log n)。

**注意**：PriorityQueue 的 `iterator()` 遍历**不保证按优先级顺序**，只有反复调用 `poll()` 才能得到有序输出。常用于任务调度、Top K 问题等场景。

###### 3. ArrayDeque 和 LinkedList 作为队列有什么区别？

[[../../../20_JavaKnowledge/02_集合框架/09、ArrayDeque与PriorityQueue#二、ArrayDeque 深度解析|📖]]
两者都实现了 `Deque` 接口，都能当队列、栈或双端队列用，但底层实现和性能差异明显。

**ArrayDeque** 底层是动态循环数组，内存连续，CPU 缓存命中率高，性能更好，内存占用更小。**不允许存 null**。

**LinkedList** 底层是双向链表，每次插入删除都要创建新节点对象，内存开销相对大。允许存 null。还实现了 `List` 接口，支持按索引访问。

**结论**：单纯需要队列或栈功能，**优先用 ArrayDeque**，性能更好，内存更省。只有需要 List 功能（按索引访问）或者需要存 null 时，才考虑 LinkedList。

###### 4. BlockingQueue 有哪些实现类？各有什么特点？

[[../../../20_JavaKnowledge/02_集合框架/10、并发集合全览（CopyOnWriteArrayList等）#四、阻塞队列（BlockingQueue）|📖]]
BlockingQueue 是 `java.util.concurrent` 里的阻塞队列接口，天生线程安全，是实现**生产者-消费者模式**的利器。主要实现类：

**ArrayBlockingQueue**：有界队列，基于数组，构造时必须指定容量。内部用单锁（或可选分离锁），生产消费会竞争同一把锁。适合需要明确边界、防止资源耗尽的场景。

**LinkedBlockingQueue**：默认无界（Integer.MAX_VALUE），也可以指定容量。基于链表，采用两把锁（putLock 和 takeLock），生产者和消费者可以并发操作，高并发吞吐量更好。是线程池默认的工作队列（`Executors.newFixedThreadPool` 用的就是它）。

**PriorityBlockingQueue**：无界优先级阻塞队列，是 PriorityQueue 的线程安全版本。队列为空时 take 会阻塞，但不会因为满而阻塞（无界）。适合需要按优先级处理任务的场景。

**SynchronousQueue**：容量为0，不存储任何元素。每次 put 都必须等一个 take，直接"手递手"传递，吞吐量极高。`Executors.newCachedThreadPool()` 的工作队列就是它。

**DelayQueue**：无界队列，元素必须实现 `Delayed` 接口。只有到了指定延迟时间后，元素才能被 take 出来。适合定时任务、缓存过期、会话超时等场景。

###### 5. ArrayBlockingQueue 和 LinkedBlockingQueue 的区别？

[[../../../20_JavaKnowledge/02_集合框架/10、并发集合全览（CopyOnWriteArrayList等）#四、阻塞队列（BlockingQueue）|📖]]
这两个是最常用的两种阻塞队列，区别有几个维度：

**容量**：ArrayBlockingQueue 是有界的，创建时必须指定大小；LinkedBlockingQueue 默认无界（但可以指定），不指定的话上限是 Integer.MAX_VALUE，生产速度远超消费时可能 OOM。

**底层存储**：ArrayBlockingQueue 用数组，内存连续，预先分配；LinkedBlockingQueue 用链表，动态创建节点，每次插入有额外对象分配开销。

**锁机制**：ArrayBlockingQueue 用单把锁控制生产和消费，两者会竞争；LinkedBlockingQueue 用两把锁（putLock + takeLock），生产者和消费者互不干扰，高并发下吞吐量更高。

**选择建议**：需要明确边界（防止内存撑爆）用 ArrayBlockingQueue；不需要边界且并发量大用 LinkedBlockingQueue。

###### 6. DelayQueue 的应用场景是什么？

[[../../../20_JavaKnowledge/02_集合框架/09、ArrayDeque与PriorityQueue#三、PriorityQueue 深度解析|📖]]
DelayQueue 包装了一个 PriorityQueue，要求所有元素实现 `Delayed` 接口（需要实现 `getDelay()` 方法返回剩余延迟时间）。队列按延迟时间排序，最快到期的在队首，到期之前 take 会阻塞。

典型应用场景：
- **定时任务调度**：把任务和执行时间封装成 Delayed 对象放进去，工作线程 take 到就执行
- **缓存过期清理**：把缓存项和过期时间放进去，后台线程不断 take 过期的并清除
- **用户会话超时**：用户登录时放入 DelayQueue，超时则自动踢下线
- **订单超时取消**：下单时放入，30分钟后取出，如果还没支付就取消订单

###### 7. ConcurrentLinkedQueue 和 LinkedBlockingQueue 的区别？

[[../../../20_JavaKnowledge/02_集合框架/10、并发集合全览（CopyOnWriteArrayList等）#五、ConcurrentLinkedQueue|📖]]
这两个都是线程安全的队列，但设计理念完全不同：

**ConcurrentLinkedQueue** 是基于 **CAS 无锁算法**的非阻塞队列，无界。offer 和 poll 不会阻塞线程，高并发下吞吐量非常高。但没有阻塞机制，消费者发现队列空了需要自己轮询，会浪费 CPU（忙等）。适合**生产消费速度大致匹配**的高并发场景。

**LinkedBlockingQueue** 是基于**锁机制**的阻塞队列，可有界可无界。队列为空时 take 会挂起线程；队列满时 put 会挂起线程，不浪费 CPU。有完善的阻塞 API，天然适合**生产者-消费者模型**，协调节奏靠阻塞而不是轮询。
