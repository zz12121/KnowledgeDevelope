###### 1. 常用的 JVM 参数有哪些？

[[../../../20_JavaKnowledge/04_JVM/09、JVM调优实战#一、调优目标与原则|📖]]
JVM 参数分几类，调优时最常用的：

**堆内存设置**：
- `-Xms2g`：堆初始大小2GB，**建议和 -Xmx 设成相同值**，避免堆动态扩展带来的性能波动（GC 会暂停用户线程来扩容）
- `-Xmx4g`：堆最大大小4GB，不要设超过机器可用物理内存，否则会触发系统 Swap，性能急剧下降
- `-Xmn1g`：新生代大小，建议为总堆的1/4到1/3
- `-Xss1m`：每个线程的栈大小，设小了容易 StackOverflowError，设大了限制可创建的线程数量

**元空间**：
- `-XX:MetaspaceSize=256m`：元空间初始大小，也是触发 Full GC 的阈值
- `-XX:MaxMetaspaceSize=512m`：元空间上限，默认无限制，**强烈建议设置**，防止类加载失控把系统内存耗光

**其他**：
- `-XX:MaxDirectMemorySize=1g`：直接内存（堆外内存）上限，默认和 -Xmx 相同

###### 3. 如何设置新生代和老年代的比例？

[[../../../20_JavaKnowledge/04_JVM/09、JVM调优实战#二、JVM 常用参数|📖]]
- `-XX:NewRatio=3`：老年代:新生代 = 3:1，即新生代占整个堆的1/4。值越大老年代越大。
- `-XX:SurvivorRatio=8`：Eden:S0:S1 = 8:1:1。Eden 占新生代的8/10，两个 Survivor 各占1/10。

经验规则：**短生命周期对象多的应用**（典型 Web 服务）适当增大新生代；**长期存活对象多的应用**（大量缓存）适当减小新生代增大老年代。

###### 4. 什么是 -XX:+UseCompressedOops？什么是 -XX:+UseCompressedClassPointers？

[[../../../20_JavaKnowledge/04_JVM/09、JVM调优实战#三、堆内存调优|📖]]
这是64位 JVM 上减少内存占用的指针压缩技术：

**-XX:+UseCompressedOops（普通对象指针压缩）**：在64位系统中，对象引用原本占8字节，开启后压缩为4字节，对象头里的引用也压缩，显著节省堆内存（通常节省20-30%）。JDK 6u23 之后默认开启，堆不超过32GB时有效。

**-XX:+UseCompressedClassPointers（类指针压缩）**：压缩对象头中指向类元数据的指针。依赖 UseCompressedOops，必须先开启 UseCompressedOops 才能生效，两者通常一起开启。

###### 6. 如何开启 GC 日志？

[[../../../20_JavaKnowledge/04_JVM/09、JVM调优实战#五、实战案例|📖]]
GC 日志是调优的根本依据，线上应用必须开启：

**JDK 8 及之前**：
- `-Xloggc:/logs/gc.log`：GC 日志输出到文件
- `-XX:+PrintGCDetails`：详细 GC 信息（各内存区域变化、耗时）
- `-XX:+PrintGCDateStamps`：输出日期时间戳
- `-XX:+HeapDumpOnOutOfMemoryError`：OOM 时自动生成堆转储文件
- `-XX:HeapDumpPath=/logs/dump.hprof`：堆转储文件路径

**JDK 9+ 统一了日志系统**（推荐）：
```
-Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=100m
```

**常用收集器的启用参数**：
- `-XX:+UseG1GC`（G1，JDK 9+ 默认服务端）
- `-XX:+UseZGC`（ZGC，低延迟）
- `-XX:+UseParallelGC`（Parallel，高吞吐，JDK 8 默认）
- `-XX:+UseConcMarkSweepGC`（CMS，已过时）

**G1 关键调优参数**：
- `-XX:MaxGCPauseMillis=200`：目标最大停顿时间（默认200ms）
- `-XX:InitiatingHeapOccupancyPercent=45`：触发并发 GC 的堆占用率阈值

###### 7. 如何分析 GC 日志？

[[../../../20_JavaKnowledge/04_JVM/09、JVM调优实战#三、堆内存调优|📖]]
可以用专业工具可视化分析，推荐 **GCViewer** 或在线工具 **gceasy.io**，把 GC 日志文件上传上去就能看报告。

关注的核心指标：

- **GC 频率**：Minor GC 多久一次，Full GC 多久一次，Full GC 太频繁（每分钟多次）是警报
- **停顿时间**：每次 GC 的 STW 时间，P99 停顿是否满足业务要求
- **内存回收效果**：GC 后老年代还有多少，是否在稳定区间而不是持续增长
- **对象提升速率**：对象从新生代晋升到老年代的速度，速率过高说明新生代太小或者有大量中长生命周期对象
- **分配速率**：内存分配速率过高容易触发频繁 Minor GC

###### 8. 什么是安全点（Safepoint）？

[[../../../20_JavaKnowledge/04_JVM/09、JVM调优实战#一、调优目标与原则|📖]]
安全点是程序执行过程中**线程状态确定、所有对象引用关系已知**的特定位置。JVM 在需要让所有线程暂停时（STW），不会在任意位置强行暂停，而是让线程运行到最近的安全点才停下来。

什么地方会设置安全点：方法调用、循环末尾、异常抛出点等控制流转移的地方。

JVM 发起 STW 时，运行中的线程会在最近的安全点停下，休眠/阻塞的线程不需要走到安全点（它们处于安全区域），等所有线程都到位了，GC 才真正开始工作。

###### 9. 什么是安全区域（Safe Region）？

[[../../../20_JavaKnowledge/04_JVM/09、JVM调优实战#一、调优目标与原则|📖]]
安全区域是一段**引用关系不会发生变化**的代码区间，比如线程阻塞在 I/O 操作、sleep、wait 时。

为什么需要安全区域：睡眠或阻塞的线程无法主动跑到安全点响应 STW 请求。但这些线程本身也不在修改引用关系，所以 GC 可以直接忽略它们，把这些线程所在的代码区间视为安全区域，不需要等它们到安全点。

线程在进入安全区域前会标记自己进入了安全区域，GC 就不会等它了。线程要离开安全区域时，需要检查 JVM 是否正在 GC，如果是就等 GC 完成再继续。

###### 10. 如何优化 JVM 的内存分配？

[[../../../20_JavaKnowledge/04_JVM/09、JVM调优实战#一、调优目标与原则|📖]]
1. **Xms = Xmx**：避免堆动态扩展，减少 GC 暂停
2. **合理设置新生代大小**：短期对象多的应用增大新生代（`-Xmn`），减少 Minor GC 后对象晋升
3. **调整 Survivor 比例**：通过 GC 日志查看对象年龄分布，如果发现对象年龄还小就大量晋升老年代，需要调整 `-XX:MaxTenuringThreshold` 或 `-XX:SurvivorRatio`
4. **避免大对象频繁创建**：配置 `-XX:PretenureSizeThreshold`，大对象直接进老年代，避免在新生代来回复制

###### 11. 如何优化 JVM 的垃圾回收性能？

[[../../../20_JavaKnowledge/04_JVM/09、JVM调优实战#一、调优目标与原则|📖]]
1. **选对 GC**：根据应用特性选（见第5题的选型指南）
2. **设合理的停顿目标**：G1 的 `-XX:MaxGCPauseMillis` 不要设太小，否则 GC 频率会提高，吞吐量反而下降
3. **GC 线程数合理**：`-XX:ParallelGCThreads` 不要超过 CPU 核心数，否则 GC 线程本身会抢占业务线程
4. **减少 Full GC**：优化代码避免内存泄漏、合理设置堆大小、老年代 GC 触发阈值不要太低

###### 12. 如何监控 JVM 的运行状态？

[[../../../20_JavaKnowledge/04_JVM/09、JVM调优实战#一、调优目标与原则|📖]]
**命令行工具**（线上快速排查）：
- `jps`：查看所有 Java 进程及其 PID
- `jstat -gcutil <pid> 1000`：每秒打印一次 GC 统计信息（各区使用率、GC 次数和耗时）
- `jstack <pid>`：打印所有线程栈信息，排查线程死锁和死循环用
- `jmap -heap <pid>`：查看堆内存使用情况
- `jmap -dump:format=b,file=heap.hprof <pid>`：生成堆转储文件

**图形化工具**：
- **JConsole**：JDK 自带，连接 JVM 进程，实时查看堆内存、线程、类加载等
- **VisualVM**：JDK 自带（JDK 9 后独立发布），功能更强，支持 Profiling 和 Heap Dump 分析

**生产监控**：Prometheus + Grafana + JVM Exporter 做长期趋势监控和告警；**Arthas** 做在线诊断（不重启 JVM 动态 trace 方法、查看类信息等）。

###### 13. 如何解决 JVM 内存泄漏问题？如何处理 JVM 的 OOM 问题？

[[../../../20_JavaKnowledge/04_JVM/09、JVM调优实战#一、调优目标与原则|📖]]
**第一步：提前配置 OOM 自动 dump**（线上一定要配）：
```
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/heap-dump.hprof
```

**第二步：复现并获取 Heap Dump**：如果已经配了上面的参数，OOM 时会自动生成；也可以用 `jmap` 手动 dump 一份当前堆的快照。

**第三步：用 MAT（Memory Analyzer Tool）分析**：
- 看 "Leak Suspects" 报告，MAT 会自动识别最可能的泄漏点
- 找占用内存最大的对象及其 GC Roots 引用链，顺着引用链找到谁"抓着不放"

**常见内存泄漏场景**：
- 集合类（Map、List）持有对象引用，但不及时清理（比如全局缓存没有过期机制）
- 监听器、回调注册了但没有注销
- 数据库连接、文件流等资源没有 close
- ThreadLocal 在线程池中使用后没有 remove

###### 15. 如何处理 JVM 的 Full GC 问题？

[[../../../20_JavaKnowledge/04_JVM/09、JVM调优实战#一、调优目标与原则|📖]]
频繁 Full GC 说明有问题，排查思路：

**1. 看 GC 日志**：确认 Full GC 的触发原因（日志里会有提示）和频率。

**2. 老年代增长过快**：可能是对象提升速率过高（新生代太小），或者有内存泄漏。用 `jstat -gcutil` 监控老年代使用率是否持续增长，增长不停就是泄漏。

**3. 大对象直接进老年代**：检查是否有超大对象（大数组、大 Map 等），考虑拆分或延迟加载。

**4. 元空间不足**：调大 `-XX:MaxMetaspaceSize`，或者检查是否有类加载失控（大量动态代理、框架反射）。

**5. 代码里显式调用 System.gc()**：找到并去掉，或者用 `-XX:+DisableExplicitGC` 禁用。

**6. CMS 的并发失败（Concurrent Mode Failure）**：CMS 并发清理期间老年代满了，退化为 Serial Old 做 Full GC，停顿时间暴增。解决：适当降低 CMSInitiatingOccupancyFraction（提前触发 CMS），或者换 G1。
