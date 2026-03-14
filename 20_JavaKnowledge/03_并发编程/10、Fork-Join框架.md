# 10、Fork-Join 框架

> **核心思想**：分治 + 工作窃取（Work-Stealing），将大任务拆分为小任务并行执行，空闲线程主动从其他线程队列尾部窃取任务。

---

## 一、整体架构

```
ForkJoinPool
├── WorkQueue[]          ← 每个工作线程拥有一个双端队列
│   ├── 自己 push/pop（LIFO，头部）
│   └── 其他线程 steal（FIFO，尾部）
├── ForkJoinWorkerThread[]
└── 提交队列（外部任务）
```

### 核心类关系

| 类 | 说明 |
|---|---|
| `ForkJoinPool` | 线程池，管理 WorkQueue 和线程 |
| `ForkJoinTask<V>` | 任务抽象基类 |
| `RecursiveTask<V>` | 有返回值的递归任务 |
| `RecursiveAction` | 无返回值的递归任务 |
| `CountedCompleter<T>` | 带完成计数的任务（JDK8+） |

---

## 二、工作窃取算法（Work-Stealing）

### 为什么需要工作窃取？

普通线程池任务队列是共享的，竞争激烈。Fork-Join 每个线程有**私有双端队列**：
- 线程自己：从**头部** push/pop（LIFO），利用缓存局部性
- 窃取线程：从**尾部** steal（FIFO），减少竞争

```
WorkQueue（双端队列）
 [tail] <--- steal ---  任务C | 任务B | 任务A  --- push/pop ---> [head]
                                ↑
                          其他空闲线程窃取尾部
```

### 窃取流程

```
1. 工作线程执行完本队列所有任务
2. 随机选择一个受害者线程的 WorkQueue
3. 从受害者队列尾部（base 端）CAS 窃取一个任务
4. 执行窃取到的任务，若该任务又 fork 出子任务则继续执行
5. 若无可窃取任务，线程进入休眠等待
```

---

## 三、ForkJoinPool 核心参数

```java
ForkJoinPool pool = new ForkJoinPool(
    parallelism,           // 并行度，默认 CPU 核心数
    ForkJoinWorkerThreadFactory factory,
    UncaughtExceptionHandler handler,
    boolean asyncMode      // true=FIFO（适合事件驱动），false=LIFO（默认）
);

// JDK8+ commonPool（全局共享池）
ForkJoinPool.commonPool();  // 并行度 = CPU-1
```

### 关键内部状态字段

```java
// ctl 字段（64位）编码了：活跃线程数、总线程数、等待线程栈顶
volatile long ctl;

// runState 字段
static final int STARTED   = 1 << 0;
static final int STOP      = 1 << 1;
static final int TERMINATED = 1 << 2;
static final int SHUTDOWN  = 1 << 3;
```

---

## 四、RecursiveTask 使用示例

### 经典：并行归并排序

```java
public class MergeSortTask extends RecursiveAction {
    private final int[] arr;
    private final int left, right;
    private static final int THRESHOLD = 1000;

    public MergeSortTask(int[] arr, int left, int right) {
        this.arr = arr;
        this.left = left;
        this.right = right;
    }

    @Override
    protected void compute() {
        if (right - left <= THRESHOLD) {
            // 小数组直接串行排序
            Arrays.sort(arr, left, right + 1);
            return;
        }
        int mid = (left + right) / 2;
        MergeSortTask leftTask = new MergeSortTask(arr, left, mid);
        MergeSortTask rightTask = new MergeSortTask(arr, mid + 1, right);

        // fork 拆分，join 等待
        leftTask.fork();
        rightTask.compute();  // 当前线程直接执行右侧（优化：减少一次入队）
        leftTask.join();

        merge(arr, left, mid, right);
    }
}

// 调用
ForkJoinPool pool = ForkJoinPool.commonPool();
pool.invoke(new MergeSortTask(arr, 0, arr.length - 1));
```

### 并行求和

```java
public class SumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 10_000;
    private final long[] data;
    private final int start, end;

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += data[i];
            return sum;
        }
        int mid = (start + end) / 2;
        SumTask left = new SumTask(data, start, mid);
        SumTask right = new SumTask(data, mid, end);
        left.fork();
        return right.compute() + left.join();
    }
}
```

---

## 五、fork() 与 join() 底层原理

### fork() 源码流程

```java
public final ForkJoinTask<V> fork() {
    Thread t;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        // 工作线程：push 到自己的队列头部
        ((ForkJoinWorkerThread) t).workQueue.push(this);
    else
        // 外部线程：提交到公共提交队列
        ForkJoinPool.common.externalPush(this);
    return this;
}
```

### join() 源码流程

```java
public final V join() {
    int s;
    if ((s = doJoin() & DONE_MASK) != NORMAL)
        reportException(s);
    return getRawResult();
}

// doJoin 核心逻辑
private int doJoin() {
    // 1. 任务已完成？直接返回
    // 2. 当前线程是 ForkJoinWorkerThread？
    //    → tryUnpush（尝试从队列头取回自己 fork 的任务直接执行）
    //    → helpStealer（帮助窃取了此任务的线程执行其任务，避免阻塞）
    // 3. 都不行，awaitJoin 阻塞等待
}
```

**关键优化**：`join()` 不会傻等，而是去帮助其他线程干活（compensation），充分利用 CPU。

---

## 六、CountedCompleter（JDK8+）

适合**触发式**场景（如遍历树、异步流水线），不需要返回值汇聚：

```java
public class TreeTraversal extends CountedCompleter<Void> {
    private final TreeNode node;

    @Override
    public void compute() {
        for (TreeNode child : node.children) {
            addToPendingCount(1);
            new TreeTraversal(this, child).fork();
        }
        processNode(node);
        tryComplete();  // 每完成一个，减少 pending count，全部完成后通知父任务
    }
}
```

---

## 七、与 Stream 并行流的关系

```java
// 并行流底层默认使用 ForkJoinPool.commonPool()
list.parallelStream().map(...).collect(...)

// 指定自定义 ForkJoinPool（避免污染公共池）
ForkJoinPool customPool = new ForkJoinPool(4);
customPool.submit(() ->
    list.parallelStream().map(...).collect(...)
).get();
```

**注意**：`commonPool` 默认并行度 = `CPU核心数 - 1`，IO密集型任务慎用，会阻塞整个公共池。

---

## 八、实战场景与踩坑

### 场景1：大文件并行处理

```java
// 将文件按行分块，并行处理
long lineCount = Files.lines(path)
    .parallel()
    .filter(line -> line.contains("ERROR"))
    .count();
```

### 场景2：树形结构并行遍历

适合组织架构、文件目录、BOM 清单等树形数据的并行聚合。

### 踩坑1：阈值（THRESHOLD）设置不当

- 阈值太小：任务拆分过多，调度开销 > 计算收益
- 阈值太大：无法充分并行
- **经验值**：计算密集型 1000~10000 元素；IO 场景不适合 Fork-Join

### 踩坑2：join() 嵌套死锁

```java
// 错误：在 compute 里同步等待外部线程提交的任务
ForkJoinTask<?> task = pool.submit(...);
task.join(); // 若 pool 线程全部阻塞，会死锁！
```

### 踩坑3：在 compute() 中执行阻塞 IO

Fork-Join 是 CPU 密集型框架，IO 阻塞会导致工作线程全部挂起，吞吐暴跌。

### 踩坑4：异常处理

```java
ForkJoinTask<Integer> task = pool.submit(new SumTask(...));
try {
    task.get();  // 抛出 ExecutionException 包装的原始异常
} catch (ExecutionException e) {
    Throwable cause = e.getCause();
}
// 或
task.join();  // 直接抛出原始异常（RuntimeException/Error）
```

---

## 九、性能调优建议

| 场景 | 建议 |
|---|---|
| CPU密集型 | 并行度 = CPU核心数，使用 commonPool |
| IO密集型 | 不推荐 Fork-Join，用 CompletableFuture + 自定义线程池 |
| 避免共享状态 | 子任务间不共享可变对象，否则引入锁竞争 |
| 合理设置 THRESHOLD | 通过 JMH 基准测试找最优值 |
| 监控 | `pool.getStealCount()` 监控窃取次数，评估并行效率 |

---

## 十、面试要点

**Q：Fork-Join 与普通线程池的区别？**
> Fork-Join 每个线程有独立 WorkQueue，工作窃取减少竞争；适合递归分治计算密集型任务。普通线程池共享任务队列，适合独立任务。

**Q：工作窃取为什么从尾部窃取？**
> 自己从头部 LIFO 执行（利用缓存局部性），别人从尾部 FIFO 窃取（减少与所有者的竞争），实现无锁或低竞争的并发访问。

**Q：join() 阻塞时线程会死等吗？**
> 不会。`doJoin()` 会尝试 `tryUnpush` 取回任务自己执行，或 `helpStealer` 帮助其他线程完成任务，从而推进整体进度，避免线程浪费。

**Q：什么时候用 CountedCompleter 而不是 RecursiveTask？**
> 不需要汇聚返回值、树形结构遍历、异步流水线等场景。CountedCompleter 基于完成计数触发，比 RecursiveTask 更灵活。

**Q：并行流默认线程数是多少？**
> `ForkJoinPool.commonPool` 的并行度 = `Runtime.getRuntime().availableProcessors() - 1`，可通过 `-Djava.util.concurrent.ForkJoinPool.common.parallelism=N` 调整。
