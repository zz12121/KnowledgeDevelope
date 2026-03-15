### 七、Netty 高性能设计

###### 1. Netty 高性能表现在哪些方面？

Netty 的高性能体现在四个维度：

- **高吞吐量**：支撑数十万乃至百万级并发连接，同时保持高数据处理速度。依靠高效的 Reactor 线程模型、无锁串行设计和零拷贝技术实现
- **低延迟**：数据从到达网络栈到应用层处理完毕的端到端延迟极低。通过减少线程上下文切换、避免数据拷贝、精心优化的任务调度实现
- **低资源消耗**：
  - CPU 高效：事件驱动模型避免无效轮询，无锁串行设计让 CPU 时间花在真正有用的工作上
  - 内存高效：内存池（PooledByteBufAllocator）+ 对象池（Recycler），大幅降低 GC 频率和停顿时间
  - 线程高效：少量固定 EventLoop 线程处理海量连接，避免"一连接一线程"的 BIO 式浪费
- **高可扩展与稳定性**：Pipeline 模块化设计使添加协议支持、业务逻辑、监控探针都不影响核心数据路径；writeBufferWaterMark 等机制防止 OOM

---

###### 2. Netty 是怎么实现高性能设计的？

Netty 的高性能是系统工程，贯穿架构各层：

**1. 异步非阻塞 I/O + Reactor 线程模型**

基于 Java NIO，利用 OS 的 epoll/kqueue 多路复用，单线程管理数百个连接。主从 Reactor 将 accept（bossGroup）和 I/O 读写（workerGroup）分离，避免相互阻塞。

**2. 串行化处理与无锁化设计**

一个 Channel 的所有 I/O 事件都在同一个 EventLoop 线程中顺序执行，ChannelHandler 不需要加任何锁。非 EventLoop 线程提交的操作自动封装为任务入队，由 EventLoop 线程顺序执行。

EventLoop 的任务队列使用 **MpscQueue**（多生产者单消费者队列，基于 JCTools），入队几乎无锁。

**3. 高效内存管理**

- **内存池**：PooledByteBufAllocator 预分配大块内存，按 Tiny/Small/Normal 三级分配，从 PoolThreadCache（线程本地缓存）优先分配，避免竞争全局 Arena，大幅减少 new byte[] 和 ByteBuffer.allocateDirect() 的调用
- **引用计数**：ByteBuf 基于 ReferenceCounted，精准确定性地释放内存，不依赖 GC
- **堆外内存优先**：Socket I/O 默认使用 DirectByteBuf，避免堆内到堆外的额外拷贝

**4. 零拷贝优化**

- CompositeByteBuf：多个 ByteBuf 逻辑合并，不拷贝数据
- FileRegion：文件传输走 sendfile 系统调用，数据不经过用户空间
- slice()/duplicate()：创建共享底层数据的视图，不拷贝

**5. 精心优化的组件**

- **FastThreadLocal**：替代 JDK ThreadLocal，通过数组下标直接访问，O(1) 且无哈希冲突
- **HashedWheelTimer**：高效时间轮算法处理定时任务
- **epoll bug 修复**：计数检测 + 重建 Selector 规避 CPU 100% 问题

---

###### 3. 简单说说 Netty 的零拷贝实现

Netty 的零拷贝主要指**减少或避免数据在用户空间（JVM）与内核空间之间，或内存内部的冗余拷贝**。

**1. 堆外内存（Direct Buffer）**

使用 DirectByteBuf 时，数据存储在 OS 管理的堆外内存。Socket 读写时，数据直接在内核缓冲区和堆外内存间传输，无需经过 JVM 堆。如果用 HeapByteBuf，JVM 需要先把数据拷贝到临时的堆外内存再交给系统调用，多了一次拷贝。

**2. CompositeByteBuf（应用层零拷贝）**

将多个物理上不连续的 ByteBuf 逻辑上组合成一个视图，不拷贝数据：
```java
ByteBuf header = ...;
ByteBuf body = ...;
CompositeByteBuf buf = Unpooled.compositeBuffer();
buf.addComponents(true, header, body);  // true = 自动增加 writerIndex
channel.write(buf);
```

内部维护 Component 列表，读写时根据偏移量委托给对应的子 ByteBuf，不产生内存拷贝。

**3. FileRegion（OS 级零拷贝）**

```java
DefaultFileRegion region = new DefaultFileRegion(fileChannel, 0, file.length());
channel.writeAndFlush(region);
```

内部调用 `FileChannel.transferTo()`，底层走 Linux 的 `sendfile` 系统调用，文件数据直接从文件系统缓存发送到网络套接字，**数据完全不经过用户空间**。

**4. 包装与切片（Wrap & Slice）**

- `Unpooled.wrappedBuffer(byte[] array)`：包装现有字节数组，不拷贝
- `ByteBuf.slice()` / `duplicate()`：创建共享底层存储的视图，修改会相互影响，但不拷贝

---

###### 4. Netty 零拷贝体现在哪些方面？

按层次划分：

**网络 I/O 层**：Direct Buffer 避免了 JVM 堆内与堆外内存间的一次数据拷贝

**数据操作层**：CompositeByteBuf 实现逻辑聚合无物理拷贝；slice()/duplicate()/readSlice() 创建视图共享底层数据

**文件传输层**：FileRegion 利用 OS 级 sendfile，文件数据从磁盘到网卡全程不经过用户态

**编码层**：MessageToByteEncoder 可以直接将对象编码到目标 ByteBuf，避免先转为 Java 对象再编码的中间步骤

---

###### 5. 如何优化 Netty 的吞吐量和延迟？

**1. 线程模型配置**

- I/O 密集型：workerGroup 线程数可适当多于 CPU 核数
- 计算密集型：建议等于 CPU 核数
- 有阻塞操作：**绝对不要在 EventLoop 线程执行阻塞**，提交到独立业务线程池

```java
// 如果有阻塞操作，为 Handler 指定独立 Executor
pipeline.addLast(businessExecutor, "business", new BusinessHandler());
```

**2. 内存 & GC 优化**

- 确认 PooledByteBufAllocator 已启用（默认已启用，可通过 `ChannelOption.ALLOCATOR` 配置）
- 合理设置 SO_RCVBUF/SO_SNDBUF，或使用 AdaptiveRecvByteBufAllocator 动态调整
- 严格遵守 ByteBuf 引用计数规则，及时 release()，防止堆外内存泄漏

**3. 网络参数调优**

```java
serverBootstrap
    .childOption(ChannelOption.TCP_NODELAY, true)        // 禁用 Nagle，降低延迟
    .childOption(ChannelOption.SO_KEEPALIVE, true)        // TCP 保活
    .option(ChannelOption.SO_BACKLOG, 1024)               // 连接等待队列
    .option(ChannelOption.SO_REUSEADDR, true);            // 允许地址重用
```

**4. 业务逻辑优化**

- 批量处理：高频小消息在应用层聚合后批量 write，减少 flush 次数（channel.write 只写缓冲区，flush 才真正发出）
- 更高效的序列化：Protobuf/Kryo 替代 JSON，减少编解码时间和数据体积

**5. 监控诊断**

- 开启泄漏检测：`-Dio.netty.leakDetection.level=ADVANCED`
- 监控 EventLoop 任务队列积压
- 用 jstack 分析线程状态，jstat 观察 GC 情况

---

###### 6. Netty 如何减少 GC 压力？

**1. 内存池化（核心）**

PooledByteBufAllocator 从预分配的大块内存中切割，释放时归还到池，不产生 byte[] 垃圾对象。线程本地缓存（PoolThreadCache）使大部分分配/释放无需竞争全局锁，且完全不走 GC。

**2. 对象池化**

Netty 用 Recycler 池化频繁创建销毁的对象（如某些 ByteBuf、ChannelHandlerContext）。基于 ThreadLocal 的轻量级对象池，每个线程维护自己的对象栈，弹出复用，压回归还。

**3. 引用计数精准释放**

ByteBuf 基于引用计数，引用归零立即释放（归还给池），不等 GC。相比 GC 的不确定性，这种显式确定性管理大幅降低内存占用和 GC 压力。

**4. 减少临时对象创建**

- 网络读写直接操作 ByteBuf，避免创建大量中间 byte[] 数组
- 使用 ThreadLocal 重用 StringBuilder 等对象

**5. 堆外内存转移压力**

DirectByteBuf 使用的堆外内存不受 JVM GC 管理，将部分内存压力转移到堆外，减轻 JVM 堆压力（但仍需池化避免频繁申请堆外内存）。

---

###### 7. Netty 的 FastThreadLocal 是什么？

FastThreadLocal 是 Netty 为优化 ThreadLocal 访问性能设计的替代品，在高并发频繁访问的场景下（如 EventLoop 的各种状态存储）性能显著优于 JDK ThreadLocal。

**JDK ThreadLocal 的性能问题**：

每个 Thread 内部有一个 ThreadLocalMap（自定义哈希表），每次 get()/set() 需要计算哈希码、查表，遇到哈希冲突还需线性探测，有一定计算开销。

**FastThreadLocal 的实现原理**：

1. **唯一索引**：每个 FastThreadLocal 创建时，从全局原子递增序列获取一个唯一的整数索引
2. **数组存储**：每个 FastThreadLocalThread 内部维护一个 InternalThreadLocalMap，核心是一个 Object[] 数组
3. **O(1) 直接访问**：get() 时直接用索引访问数组 `indexedVariables[index]`，无哈希计算，无冲突

```java
public final V get() {
    InternalThreadLocalMap map = InternalThreadLocalMap.get();
    Object v = map.indexedVariable(index);  // 数组下标直接访问，O(1)
    if (v != UNSET) return (V) v;
    return initialize(map);
}
```

**性能优势**：省去哈希计算和冲突解决开销；数组存储比哈希表对 CPU 缓存更友好。

**使用注意**：
- 最大性能需要线程是 FastThreadLocalThread（Netty DefaultThreadFactory 默认创建）
- 普通 Thread 上会退化到慢速备用 ThreadLocal
- 用完记得 remove()，否则数组可能无限膨胀

---

###### 8. 【高频追问】Netty 的 HashedWheelTimer 是什么？适用什么场景？

HashedWheelTimer 是 Netty 实现的**高效定时任务调度器**，基于时间轮算法，专门解决大量短期定时任务的场景，比 JDK 的 ScheduledThreadPoolExecutor 效率高得多。

**时间轮原理**：

想象一个表盘，分成 N 个槽（ticksPerWheel），有一根指针每隔一个 tickDuration 跳一格。每个槽中存放该时刻到期的任务列表。

- 添加任务时：计算任务在哪个槽（`(currentTick + delay / tickDuration) % ticksPerWheel`），有的任务可能要转多圈（rounds 字段记录圈数）
- 每次 tick：遍历当前槽的任务，rounds > 0 的任务减 1，rounds == 0 的任务执行

**时间复杂度**：O(1) 添加任务；O(1) tick（每个槽的任务数均摊）

**Netty 中的应用**：
- 心跳检测：IdleStateHandler 内部就用了 HashedWheelTimer 实现读写空闲检测
- 连接超时：Channel 连接超时检测
- 断线重连：重连等待的定时任务

**适用场景**：大量定时任务、延迟执行任务，且对精度要求不是极高（tickDuration 级别的精度）

**不适合**：任务数量很少，或需要极高精度（毫秒级精度需要很小的 tickDuration，会有空转开销）
