# 十八、Netty 配置与调优

###### 1. Netty 有哪些重要的配置参数？

Netty 的配置参数通过 `ServerBootstrap.option()`（服务端 Channel 参数）和 `childOption()`（子 Channel 参数）来设置。

**线程模型参数：**

**EventLoopGroup 线程数**：
- `bossGroup`：通常设 1 个线程，只负责 accept 新连接。只有绑定多个端口时才考虑增加
- `workerGroup`：默认 CPU核心数 × 2。直接影响 I/O 并发处理能力

**TCP 底层参数（ChannelOption）：**

`SO_BACKLOG`：全连接队列（Accept Queue）的最大长度。高并发场景必须调大，如 1024。默认 Linux 是 128。

`TCP_NODELAY`：设为 `true` 禁用 Nagle 算法，降低延迟。低延迟服务（游戏、IM、金融）必须开启。

`SO_KEEPALIVE`：设为 `true` 开启 TCP 层心跳探测。探测间隔极长（默认 2 小时），只作保底兜底，应用层心跳才是主要手段。

`SO_RCVBUF` / `SO_SNDBUF`：内核 TCP 接收/发送缓冲区大小。**最佳实践是设为 -1，使用系统默认自动调优**；只有在已知网络特性（高带宽高延迟）时才手动设置。

`SO_REUSEADDR`：允许端口复用，即使处于 TIME_WAIT 状态也能绑定，服务重启时避免 "Address already in use"。

**内存分配参数：**

`ALLOCATOR`（`ChannelOption.ALLOCATOR`）：设置 ByteBuf 分配器。
- `PooledByteBufAllocator.DEFAULT`：**生产环境必须用**，池化内存分配，减少碎片和 GC 压力
- `UnpooledByteBufAllocator.DEFAULT`：每次分配新对象，测试用

`RCVBUF_ALLOCATOR`（`ChannelOption.RCVBUF_ALLOCATOR`）：控制每次读循环分配的 ByteBuf 大小。默认 `AdaptiveRecvByteBufAllocator`，根据历史读取量动态调整，避免空间浪费。

**高低水位线参数：**

`WRITE_BUFFER_WATER_MARK`：控制写缓冲区的高低水位，实现背压。
- 待发送字节数 > high → `channel.isWritable()` 返回 false，触发 `channelWritabilityChanged` 事件 → 暂停写入
- 字节数降到 < low → `isWritable()` 恢复 true → 继续写入

**超时参数：**

`CONNECT_TIMEOUT_MILLIS`：客户端连接超时时间，影响 `Bootstrap.connect()` 操作。

---

###### 2. 如何调优 Netty 的性能？

**1. 线程模型调优（最核心）**

**I/O 与业务严格隔离**：绝对禁止在 `channelRead` 等 I/O 事件方法中执行耗时操作（DB 查询、远程调用）。必须将任务提交到独立业务线程池：

```java
ch.pipeline().addLast(businessGroup, new BusinessLogicHandler());
// 或
ch.pipeline().addLast(businessGroup, new PacketDecoder());
```

**合理设置线程数**：
- `bossGroup` 设 1 个线程
- `workerGroup` 默认 CPU×2，纯 I/O 密集型可适当增加；计算密集型应等于 CPU 核心数
- **必须压测**，理论值只是参考，监控 EventLoop 任务队列积压情况

**2. 内存调优**

使用 `PooledByteBufAllocator`（生产环境默认必须），通过 PoolArena/PoolChunk/PoolSubpage 三级结构高效管理直接内存。

**防止内存泄漏**（遵循"谁最后用，谁负责释放"原则）：
- 继承 `SimpleChannelInboundHandler` → 父类自动释放入站消息
- 出站消息 → Netty 在写完后自动释放
- 手动创建的 ByteBuf → 用 `ReferenceCountUtil.release(msg)` 手动释放

**开启内存泄漏检测**（测试环境）：

```bash
-Dio.netty.leakDetection.level=PARANOID
```

**3. I/O 与网络参数调优**

```java
b.option(ChannelOption.SO_BACKLOG, 1024)         // 同时调整系统 somaxconn
 .childOption(ChannelOption.TCP_NODELAY, true)    // 禁用 Nagle，降延迟
 .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
 .childOption(ChannelOption.WRITE_BUFFER_WATER_MARK, 
              new WriteBufferWaterMark(32 * 1024, 64 * 1024));
```

**大文件传输用零拷贝**：`DefaultFileRegion` 通过 `sendfile` 系统调用，完全绕过用户态内存拷贝。

**4. 协议与序列化优化**

- 设计精简的私有协议，避免冗余字段，长度字段用 int（别用 varint，编解码复杂）
- 使用 Protobuf 或 Kryo，远离 Java 原生序列化

**5. GC 优化**

- 大量使用直接内存，设置合理的 `-XX:MaxDirectMemorySize`
- JVM 参数避免 `System.gc()` 触发 Full GC：`-XX:+DisableExplicitGC`
- 优先使用 G1 或 ZGC 收集器

---

###### 3. SO_BACKLOG 参数的作用是什么？

`SO_BACKLOG` 定义了**全连接队列（Accept Queue）的最大长度**。

**TCP 连接建立流程中的角色：**

1. 客户端发 SYN → 服务端将连接放入**半连接队列**（SYN Queue），状态 SYN_RECV
2. 三次握手完成 → 连接从半连接队列移到**全连接队列**（Accept Queue），状态 ESTABLISHED
3. Netty 的 boss 线程调用 `ServerSocketChannel.accept()` 从全连接队列取出连接

`SO_BACKLOG` 限制的就是第 2 步中全连接队列的大小。

**队列满了怎么办？**

默认行为（`tcp_abort_on_overflow=0`）：服务端忽略客户端的第三次 ACK，客户端以为连接建立了，但服务端实际上会丢弃。服务端会重传 SYN-ACK，等待队列腾出空间后再建立；若始终没空间，客户端最终收到 RST。

设为 1：服务端直接回 RST，立即拒绝。

**Netty 中设置：**

```java
b.option(ChannelOption.SO_BACKLOG, 1024)
```

**重要细节：实际队列长度 = min(SO_BACKLOG, somaxconn)**

所以在 Netty 里设 1024 不够，还需要：

```bash
sysctl -w net.core.somaxconn=1024
sysctl -w net.ipv4.tcp_max_syn_backlog=1024
```

否则设再大也没用，内核会以 somaxconn 为准。

---

###### 4. SO_KEEPALIVE 参数的作用是什么？

`SO_KEEPALIVE` 是 TCP 层提供的连接保活探测机制，用于检测对端是否"已死"（进程崩溃、主机断电、网络不通）。

**工作原理：**

启用后，如果连接上在 `tcp_keepalive_time`（默认 7200 秒，即 2 小时）内没有数据交换，内核自动发送一个空 ACK 保活探测包：
- 收到正常 ACK → 连接活跃，计时器重置
- 收到 RST → 对端已重启，关闭连接
- 连续 `tcp_keepalive_probes`（默认 9）次无响应 → 关闭连接（每次间隔 `tcp_keepalive_intvl`，默认 75 秒）

**Netty 中设置：**

```java
.childOption(ChannelOption.SO_KEEPALIVE, true)
```

**严重局限性（三大硬伤）：**

1. **探测间隔极长**：默认 2 小时才触发，根本赶不上业务需求
2. **只检测 TCP 连接**：检测不了应用层假死（进程卡住但 TCP 还通着）
3. **受中间设备影响**：NAT 防火墙可能丢弃长时间无数据的连接，`SO_KEEPALIVE` 反而成了问题

**结论：`SO_KEEPALIVE` 只是最后的保底兜底，Netty 中应用层心跳（`IdleStateHandler`）才是主要的连接保活手段。两者最好结合使用。**

---

###### 5. TCP_NODELAY 参数的作用是什么？

`TCP_NODELAY=true` 表示**禁用 Nagle 算法**。

**Nagle 算法是什么：**

Nagle 算法旨在减少小数据包数量。核心规则：**一个 TCP 连接上最多只能有一个未被确认的小分组（小于 MSS）。在收到 ACK 之前，后续的小分组会被缓冲等待。**

举个例子：客户端发 "hello"（5字节），在收到 ACK 之前，如果又写了 "world"，"world" 会被缓冲，等到 "hello" 的 ACK 到达后才发出。这引入了一个 RTT 的延迟。

**优点**：合并小包，减少网络拥塞，提升吞吐量。

**缺点**：引入延迟，在请求-响应模式的交互式应用中尤为明显。

**Netty 中设置：**

```java
.childOption(ChannelOption.TCP_NODELAY, true)
```

**选择策略：**

- **低延迟优先**（IM、游戏、金融交易、RPC）：设 `true`，禁用 Nagle，减少延迟
- **吞吐量优先、对延迟不敏感**（大文件传输、批量数据同步）：可保持默认 `false`

**Netty 的最佳实践是默认禁用 Nagle 算法**，因为大多数 Netty 应用场景都对延迟敏感，而且应用层自己会做合理的批量写入。

---

###### 6. SO_RCVBUF 和 SO_SNDBUF 的作用是什么？

两者设置的是**内核**中为 TCP 套接字分配的接收和发送缓冲区大小，与 Netty 的用户态 ByteBuf 不是同一个层次。

**`SO_RCVBUF`（接收缓冲区）：**

数据从网卡 → 内核接收缓冲区 → 用户态 ByteBuf。如果应用层（Netty）来不及消费，数据暂存在内核缓冲区；满了则通过 TCP 滑动窗口通知对端停止发送（流量控制）。

**`SO_SNDBUF`（发送缓冲区）：**

应用调用 `ctx.write()` 时，数据先进内核发送缓冲区，内核协议栈在适当时机（收到 ACK、Nagle 算法等）发出；缓冲区满时，非阻塞模式返回 EAGAIN。

**Netty 中设置：**

```java
.option(ChannelOption.SO_RCVBUF, 128 * 1024)
.option(ChannelOption.SO_SNDBUF, 128 * 1024)
```

**重要注意事项：**

1. Java 设置的是"建议值"，内核会调整为 2 的幂次方，且受 `net.core.rmem_max`/`wmem_max` 上限限制
2. 现代 Linux 支持 TCP 自动调优（`tcp_moderate_rcvbuf=1`），内核会动态调整，可能覆盖你的设置
3. **最佳实践：设为 -1（使用系统默认值）**，只有在跨数据中心高带宽高延迟场景，才根据 BDP（带宽延迟积）手动计算并调大

**BDP 计算示例**：带宽 1Gbps，RTT 100ms → BDP = 1Gbps × 100ms / 8 ≈ 12.5MB，则缓冲区应设 ≥ 12.5MB 才能跑满带宽。

---

###### 7. 如何设置 Netty 的线程池大小？

**BossGroup 线程数：**

职责只有一个：accept 新连接。通常设 1 个线程足够，哪怕万级并发新连接也应付得来。只有绑定多个端口时才考虑增加。

**WorkerGroup 线程数：**

默认 CPU核心数 × 2。设置思路：

- **纯 I/O 密集型**（消息转发、代理）：可设 CPU × 2，因为大部分时间在等 I/O
- **计算密集型**（复杂编解码）：应设接近 CPU 核心数，避免线程切换开销过大
- **通用公式**：`N = CPU核心数 × CPU利用率目标 × (1 + 等待时间/计算时间)`

**示例配置（CPU 8核）：**

```java
int cores = Runtime.getRuntime().availableProcessors(); // 8
EventExecutorGroup businessGroup = new DefaultEventExecutorGroup(cores * 2); // 16

ServerBootstrap b = new ServerBootstrap();
b.group(new NioEventLoopGroup(1),           // bossGroup
        new NioEventLoopGroup(cores))       // workerGroup: 8线程
 .childHandler(new ChannelInitializer<SocketChannel>() {
     protected void initChannel(SocketChannel ch) {
         ch.pipeline()
           .addLast(new PacketDecoder())    // I/O 线程执行
           .addLast(new PacketEncoder())    // I/O 线程执行
           .addLast(businessGroup, new BusinessLogicHandler()); // 业务线程池
     }
 });
```

**业务线程池大小（`DefaultEventExecutorGroup`）：**

独立于 WorkerGroup 设置，通常 CPU × 2 起步，再根据业务阻塞程度调整。监控其队列长度和活跃线程数，确保无明显积压。

**核心原则：理论值仅供参考，必须压测验证。目标是：达到目标 QPS 时，CPU 利用率在 70%-80%，EventLoop 任务队列无积压。**

---

###### 8. 如何监控 Netty 的性能指标？

**1. 内置指标采集**

```java
// 内存池使用情况（诊断内存泄漏/碎片的核心）
PooledByteBufAllocatorMetric metric = PooledByteBufAllocator.DEFAULT.metric();
System.out.println("Used heap: " + metric.usedHeapMemory());
System.out.println("Used direct: " + metric.usedDirectMemory());
```

**2. 自定义 MetricsHandler（集成 Micrometer）**

```java
public class MetricsHandler extends ChannelDuplexHandler {
    private final AtomicInteger activeConnections;
    private final Counter readBytes;
    
    public MetricsHandler(MeterRegistry registry) {
        activeConnections = registry.gauge("netty.connections.active", new AtomicInteger(0));
        readBytes = registry.counter("netty.bytes.read");
    }
    
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        activeConnections.incrementAndGet();
        ctx.fireChannelActive();
    }
    
    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        activeConnections.decrementAndGet();
        ctx.fireChannelInactive();
    }
    
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        if (msg instanceof ByteBuf) {
            readBytes.increment(((ByteBuf) msg).readableBytes());
        }
        ctx.fireChannelRead(msg);
    }
}
```

**关键监控指标清单：**

- **连接数**：当前活跃连接数（`channelActive`/`channelInactive` 计数）
- **流量**：每秒/累计读写字节数（`ChannelTrafficShapingHandler` 有内置统计）
- **延迟**：请求处理耗时（在 Handler 入口/出口打时间戳）
- **EventLoop 任务队列**：待处理任务数，积压预警
- **内存池**：直接内存使用量、分配次数、释放次数
- **错误**：连接异常断开次数、解码失败次数

**OS 和 JVM 监控：**

- `ss -s` / `netstat -antp`：查看各状态 TCP 连接数，特别关注 TIME_WAIT 和 CLOSE_WAIT
- `jstack`：检查 EventLoop 线程是否阻塞
- `BufferPoolMXBean`：直接内存使用量（JVM 级别）

---

###### 9. 高频追问：线上 Netty 服务 OOM，应该怎么排查？

**OOM 在 Netty 中有两种：堆内存 OOM 和直接内存 OOM（`OutOfMemoryError: Direct buffer memory`）。**

**排查步骤：**

**第一步：确认 OOM 类型**

看 OOM 错误信息：堆内存 OOM 是 `Java heap space`；直接内存 OOM 是 `Direct buffer memory`。

**第二步：直接内存 OOM 排查**

最常见原因是 ByteBuf 泄漏。立即开启泄漏检测：

```bash
-Dio.netty.leakDetection.level=PARANOID
```

日志会打印出泄漏的 ByteBuf 在哪里被创建、哪里被访问、哪里没有被释放，精确到代码行。

常见泄漏场景：
- `channelRead` 中保存了 ByteBuf 引用但忘记 retain + release
- 自定义 Handler 没有继承 `SimpleChannelInboundHandler`，手动处理也没 release
- 编码器中创建了 ByteBuf 但没加入 out 列表，Netty 不会自动释放

**第三步：堆内存 OOM 排查**

用 `jmap -dump:format=b,file=heap.hprof <pid>` 导出堆快照，用 MAT 分析：
- 看最大的对象是什么
- 如果是大量 `DefaultChannelPromise` 或 `ChannelFuture`，说明写操作太多没 flush，堆积在 `ChannelOutboundBuffer`
- 如果是业务对象，说明业务处理速度跟不上接收速度，需要背压控制

**第四步：背压控制**

调大 JVM 直接内存上限或添加背压：当 `channel.isWritable()` 为 false 时，暂停从上游接收消息；`channelWritabilityChanged` 变为 true 时再恢复。
