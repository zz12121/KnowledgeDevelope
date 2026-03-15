# 08、Netty 内存管理

> 相关知识库：[[35_NettyKnowledge/03_高性能设计]]

---

## 1. Netty 的内存管理机制是什么？

Netty 的内存管理建立在两大支柱上：**内存池**和**引用计数**，目的是减少系统调用、降低 GC 压力、实现精准的内存生命周期控制。

**内存池化**：Netty 从操作系统或 JVM 预先申请大块内存，切分成不同规格的内存块统一管理。申请内存时从池里找最合适的块，释放时归还池中而不是交给 GC，大幅降低了分配/释放的开销和内存碎片。

**引用计数**：`ByteBuf` 实现了 `ReferenceCounted` 接口，内部维护一个计数器。`retain()` 加计数，`release()` 减计数，计数归零时触发实际释放（归还给内存池）。这种确定性的方式比 GC 更可预测。

**分层管理结构（记忆口诀："Arena→Chunk→Page→Subpage"）**：

- **Arena**：内存池被划分为多个 `PoolArena`（默认 CPU 核心数 × 2），每个线程绑定一个 Arena，减少锁竞争
- **Chunk**：Arena 管理多个 `PoolChunk`，每个 Chunk 通常 16MB，是向操作系统申请/释放内存的基本单位
- **Page**：Chunk 被划分为多个 Page（默认 8KB），中等大小的内存以 Page 为单位分配
- **Subpage**：小于 8KB 的内存，Page 会进一步切成等大的 Subpage（16B、32B……4KB），用位图管理分配状态

**线程本地缓存**：每个线程在分配和释放时，优先在 `PoolThreadCache` 中操作，避免多线程竞争 Arena，是高性能的关键。

---

## 2. Netty 是如何管理内存的？

Netty 通过 `ByteBufAllocator` 接口统一管理内存，核心有两个实现：

**`PooledByteBufAllocator`（默认）**：

- **小内存（< 8KB）**：先查当前线程的 `PoolThreadCache` Subpage 缓存；缓存未命中则到 `PoolArena` 的 Subpage 链表申请；链表为空则从 `PoolChunk` 分出一个 Page 切成 Subpage
- **中等内存（8KB ~ 16MB）**：从 `PoolArena` 的 `PoolChunk` 中分配连续 Page，Chunk 内部通过完全平衡二叉树（伙伴分配算法）快速查找
- **大内存（> 16MB）**：直接由 JVM 分配，不走内存池

释放时调用 `release()` 引用计数减 1，归零后如果缓存未满就进线程缓存，否则归还给 Arena。

**`UnpooledByteBufAllocator`**：每次直接调 `ByteBuffer.allocate()` 或 `allocateDirect()`，释放依赖 GC，简单但 GC 压力大，适合一次性场景或测试。

源码入口：`PooledByteBufAllocator.newDirectBuffer()` → `PoolArena.allocate()` → 先尝试 `PoolThreadCache.allocateTiny()` → 再走 Arena 分配流程。

---

## 3. 如何设计一个内存池？

设计高效内存池需要考虑几个核心维度：

**内存划分策略**：
- 固定大小块：管理简单，但有内部碎片
- 可变大小块：灵活但易碎片化，需首次适应/最佳适应等算法
- **分级分配（推荐）**：像 Netty 一样，小内存用固定大小块（Subpage），中等用 Page 组合，大内存特殊处理

**分配算法**：
- **位图法**：一个比特位表示一块内存的分配状态，适合固定大小块，O(1) 检查
- **空闲链表法**：将空闲块用链表串联，分配时遍历找到合适的，需要分割和合并逻辑
- **伙伴系统（Buddy System）**：内存按 2 的幂次划分，分配时取最接近的 2 的幂大小；释放时检查相邻"伙伴"是否空闲，是则合并。能有效减少外部碎片，Netty 的 Page 分配就采用了这个思想

**并发性能**：
- 多 Arena 分区：不同线程用不同区域，减少竞争
- **线程本地缓存**：大部分分配释放在线程内完成，无需锁全局资源

**监控与调试**：记录分配/释放统计，检测内存泄漏（分配了未释放的块）。

简易设计：预分配 1GB 内存，切成 4KB 的槽，用 `BitSet` 记录分配状态，分配时扫描 BitSet，用 `ReentrantLock` 保护并发访问。

---

## 4. Netty 的内存池是如何实现的？

Netty 的 `PooledByteBufAllocator` 是一个工业级的分级、分区、带线程缓存的高性能内存池，核心组件如下：

**PoolArena**：
- 内部维护 `tinySubpagePools` 和 `smallSubpagePools` 两个数组，管理不同规格的小内存
- 还有 6 个 `PoolChunkList`，分别管理不同使用率（0-25%、25-50%……100%）的 Chunk，优先从低使用率的 ChunkList 分配

**PoolChunk**：
- 管理 16MB 的连续内存，内部是一棵**完全平衡二叉树**（数组实现，树高 12，叶子 2048 个，每个叶子代表一个 8KB Page）
- 每个节点记录子树中可用 Page 数量，分配时从根节点向下找最浅的满足条件的节点，时间复杂度 O(logN)

**PoolSubpage**：
- 每个 `PoolSubpage` 管理一个 Page（8KB），切分为多个等长单元
- 内部用 `long[]` 作位图，每个比特位表示一个单元是否被分配

**PoolThreadCache**：
- 线程本地缓存，分 tiny、small、normal 三种规格的缓存队列
- 分配时优先从这里取；释放时如果缓存未满则放入，否则归还 Arena

**以分配 256B 的 Direct Buffer 为例**：
1. 调用 `PooledByteBufAllocator.newDirectBuffer(256)`
2. 通过 ThreadLocal 获取或绑定一个 Arena 和 PoolThreadCache
3. 256B < 8KB，进入小内存路径，先查 PoolThreadCache 的 tiny 缓存队列
4. 缓存未命中 → 调 `PoolArena.allocate()` → 找对应 tinySubpagePools 链表
5. 链表为空 → 从 PoolChunk 分一个 Page 转成管理 256B 单元的 PoolSubpage
6. 从 PoolSubpage 的位图里分配一个空闲单元，初始化 `PooledByteBuf` 返回

---

## 5. PooledByteBuf 和 UnpooledByteBuf 有什么区别？

**PooledByteBuf**：从内存池中分配，`release()` 引用计数归零时将内存归还给池而不是交给 GC。分配释放成本低，是 Netty 高性能的默认选择。

**UnpooledByteBuf**：每次分配都创建新的 `byte[]` 或 `DirectByteBuffer`，释放依赖 JVM GC 或 Cleaner 机制。简单直接，适合一次性场景或测试。

Netty 默认使用 `PooledByteBufAllocator.DEFAULT`，`Unpooled` 工具类提供了创建非池化 ByteBuf 的静态方法。在 ChannelHandler 中通过 `ctx.alloc()` 获取的分配器通常是池化的。

---

## 6. 如何保证 Netty 中不会发生内存泄漏？

内存泄漏在 Netty 里通常是 `ByteBuf` 没有被正确 `release()` 导致的，核心原则是"**谁最后使用，谁负责释放**"。

**实战建议**：

1. **优先用 `SimpleChannelInboundHandler`**：它会自动释放入站消息，你只需在 `channelRead0()` 里写业务逻辑。但注意：如果你在里面把 msg 传出去（调了 `retain()`），仍需手动 `release()`

2. **传递 ByteBuf 时先 `retain()`**：如果把 ByteBuf 保存到成员变量或提交给异步线程，必须先 `retain()`，并在最终使用完后 `release()`

3. **出站消息放心交给 Netty**：调用 `ctx.write(msg)` 后，Netty 负责释放这个 msg；失败情况下 Netty 也会释放，不用手动处理

4. **用 finally 保底**：在可能抛异常的路径，用 finally 块兜底释放

5. **用工具类**：`ReferenceCountUtil.safeRelease(msg)` 会在释放前检查是否可释放，更安全

---

## 7. Netty 的引用计数机制是什么？

`ReferenceCounted` 接口定义了引用计数的核心 API，由 `ByteBuf`、`ByteBufHolder` 等实现：

- `refCnt()`：返回当前引用计数
- `retain()`：增加计数 1
- `retain(int increment)`：增加指定计数
- `release()`：减少计数 1，归零则触发 `deallocate()`，返回 true
- `release(int decrement)`：减少指定计数

**实现原理**：计数存储在 `volatile int` 字段中，用 `AtomicIntegerFieldUpdater` 做 CAS 操作，避免了创建 `AtomicInteger` 对象的开销。

```java
// AbstractReferenceCountedByteBuf 的核心逻辑
private static final AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> refCntUpdater =
        AtomicIntegerFieldUpdater.newUpdater(AbstractReferenceCountedByteBuf.class, "refCnt");
private volatile int refCnt = 1; // 初始值为1

@Override
public boolean release() {
    for (;;) {
        int ref = refCntUpdater.get(this);
        if (ref == 0) {
            throw new IllegalReferenceCountException(0, -1);
        }
        if (refCntUpdater.compareAndSet(this, ref, ref - 1)) { // CAS 原子减
            if (ref == 1) {
                deallocate(); // 计数归零，触发实际释放
                return true;
            }
            return false;
        }
    }
}
```

**注意**：`slice()`、`duplicate()`、`readSlice()` 创建的派生缓冲区共享底层数据但拥有**独立的引用计数（初始为 1）**，需要单独管理释放。

---

## 8. 如何检测 Netty 中的内存泄漏？

**Netty 内置的 `ResourceLeakDetector`**（首选）：

通过系统属性 `io.netty.leakDetection.level` 设置检测级别：
- `DISABLED`：禁用
- `SIMPLE`：**默认**，约 1% 采样率，报告泄漏并显示最近访问记录
- `ADVANCED`：同样 1% 采样，但提供更详细的访问路径
- `PARANOID`：检测**所有**分配，性能开销最大，仅用于测试环境

当 ByteBuf 被 GC 回收但从未调用 `release()` 时，日志会出现：
```
LEAK: ByteBuf.release() was not called before it's garbage-collected.
Recent access records (5 latest): ...
```

**其他手段**：
- 搜索日志中的 `LEAK:` 关键字，这是最直接的泄漏信号
- 用 `jcmd <pid> VM.native_memory` 监控堆外内存使用趋势
- VisualVM 或 JMC 监控 Direct Buffer 的实例数量变化
- 测试时用高并发模拟，结束后检查 ByteBuf 的分配/释放是否平衡

---

## 9. Direct Buffer 和 Heap Buffer 的区别？

两者最核心的区别在于**数据在哪里**：

**Heap Buffer（堆内缓冲区）**：
- 数据存在 JVM 堆内存中，是普通的 `byte[]`
- 分配释放由 GC 管理，开销较低
- 做 Socket I/O 时，JVM 需要先把数据拷到堆外临时缓冲区再系统调用，多一次拷贝
- 适合业务逻辑处理，对数据进行复杂操作的场景

**Direct Buffer（堆外缓冲区）**：
- 数据在 JVM 堆外，底层是操作系统内存（`ByteBuffer.allocateDirect()`）
- 分配/释放涉及系统调用，成本高于堆内；释放依赖 Cleaner 机制
- 可直接作为 Socket I/O 系统调用参数，**实现零拷贝**，I/O 性能更好
- 不占用堆空间，不受 Young GC 直接影响；但泄漏会造成堆外内存耗尽
- Java 代码访问相对慢（涉及 JNI 或 `sun.misc.Unsafe`）
- 适合网络 I/O 操作和需要避免额外拷贝的场景

**Netty 的选择**：出于性能考虑，Netty 进行网络 I/O 时默认分配 Direct Buffer。业务处理中如果需要对数据做复杂处理，有时会把 Direct Buffer 内容拷到 Heap Buffer 里操作，处理完再写回 Direct Buffer 发送。

---

## 高频追问：Netty 内存池分配 256B 的完整流程？

完整流程走的是"**PoolThreadCache → PoolArena → PoolChunk → PoolSubpage**"这条路：

1. **线程缓存查询**：调用方的线程绑定了一个 `PoolThreadCache`，先查 tiny 缓存队列里有没有 256B 规格的缓存块。有的话直接返回，全程无锁，速度最快

2. **Arena 分配**：缓存未命中，进入 `PoolArena.allocate()`。因为 256B < 8KB，走 tiny 分配路径，到 `tinySubpagePools` 数组里找对应 256B 规格的链表

3. **Chunk 申请 Page**：如果 Subpage 链表为空（第一次分配这个规格），就从 `PoolChunk` 的伙伴二叉树中分配一个 8KB 的 Page

4. **Page 切 Subpage**：把这个 Page 转成 `PoolSubpage`，按 256B 切成 32 个槽，用 `long[]` 位图记录每个槽的分配状态，并加入 tinySubpagePools 链表

5. **位图分配**：从 PoolSubpage 的位图中找第一个空闲比特，标记已用，初始化并返回 `PooledByteBuf`

**释放时的逆流程**：`release()` → 引用计数归零 → `deallocate()` → 如果 PoolThreadCache 未满则进缓存 → 缓存满了则调 `PoolSubpage.free()` 位图清位 → 如果整个 Page 空闲了则归还给 PoolChunk 的二叉树
