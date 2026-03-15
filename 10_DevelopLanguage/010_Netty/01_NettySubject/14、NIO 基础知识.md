# 14、NIO 基础知识

---

## 1. 说说 NIO 的组成

Java NIO（New I/O）是 Java 1.4 引入的非阻塞 I/O API，四大核心组件（记忆口诀：**通缓选键**）：

**① Channel（通道）**：数据交互的管道，类似 Stream 但可以同时读写，支持异步非阻塞操作。
- `FileChannel`：文件读写
- `SocketChannel`：TCP 客户端
- `ServerSocketChannel`：TCP 服务端监听
- `DatagramChannel`：UDP

**② Buffer（缓冲区）**：数据读写的唯一目的地和来源，所有数据必须通过 Buffer 中转。核心是 `ByteBuffer`，还有 `CharBuffer`、`IntBuffer` 等。

Buffer 内部维护四个属性：
- `capacity`：总容量，固定不变
- `limit`：当前可读/写的上限
- `position`：当前读/写位置
- `mark`：通过 `mark()` 设置的标记点，`reset()` 可回退到此

常用操作：`flip()`（写模式切读模式：limit=position, position=0）、`clear()`（清空准备写）、`compact()`（保留未读数据，继续写）。

**③ Selector（选择器）**：多路复用器。一个 Selector 可以监听多个 Channel 的 I/O 事件（读/写/连接/接受）。在 Linux 上底层用 `epoll`，Windows 用 `select`。

**④ SelectionKey（选择键）**：代表 Channel 在 Selector 中的注册关系，包含：
- `interestOps`：感兴趣的事件集合（`OP_READ`/`OP_WRITE`/`OP_CONNECT`/`OP_ACCEPT`）
- `readyOps`：已就绪的事件集合
- `attachment`：可附加任意对象（如连接状态信息）

**NIO 工作流程**：
1. Channel 设非阻塞 → 注册到 Selector 并指定感兴趣事件
2. 调 `Selector.select()` 阻塞等待就绪事件
3. 有事件就绪 → `selectedKeys()` 拿到就绪键集合 → 遍历处理
4. 读操作：`channel.read(buffer)` → `buffer.flip()` → 从 buffer 消费数据
5. 清理已处理的键（`iterator.remove()`），继续下一轮轮询

---

## 2. NIO 是如何实现同步非阻塞的？

先搞清楚两个概念：
- **同步**：I/O 操作的**数据拷贝阶段**需要应用程序自己调用（`channel.read(buffer)`），而非由操作系统主动推送
- **非阻塞**：发起 I/O 调用时，如果数据未就绪，**立即返回**而非挂起线程

**NIO 的非阻塞实现**：

**Channel 非阻塞模式**：
```java
channel.configureBlocking(false); // 切换为非阻塞
int n = channel.read(buffer); // 没有数据时立即返回 0，不阻塞
```
底层：非阻塞模式下调用 `recv` 系统调用时带 `MSG_DONTWAIT` 标志，立即返回。

**Selector 多路复用（高效等待的关键）**：
```java
Selector selector = Selector.open();
channel.register(selector, SelectionKey.OP_READ);

while (true) {
    int ready = selector.select(); // 阻塞，直到至少有一个 Channel 就绪
    if (ready > 0) {
        Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
        while (iter.hasNext()) {
            SelectionKey key = iter.next();
            if (key.isReadable()) {
                SocketChannel sc = (SocketChannel) key.channel();
                sc.read(buffer); // 此时数据已就绪，read 不会阻塞
            }
            iter.remove(); // 必须手动移除，否则下次 select 还会返回这个键
        }
    }
}
```

**总结**：NIO 是"用一次阻塞（`select()`）换取对多个连接的批量等待"。没有就绪事件时，线程阻塞在 `select()` 上；有事件时，线程被唤醒，只对就绪的 Channel 做不会阻塞的 I/O 调用。用少量线程管理大量连接，避免了 BIO 中"一连接一线程"的巨大开销。

---

## 3. NIO 和 BIO 到底有什么区别？

| 维度 | BIO（同步阻塞） | NIO（同步非阻塞） |
|------|----------|----------|
| **I/O 模型** | 流（Stream）导向，顺序读写 | 缓冲区（Buffer）导向，先读到 Buffer |
| **阻塞性** | 完全阻塞，`read()` 一直等到数据就绪 | 可选非阻塞，`read()` 没数据立即返回 0 |
| **线程模型** | **一连接一线程**，连接多时线程数线性增长 | **一线程管多连接**，通过 Selector 多路复用 |
| **API 复杂度** | 简单直观 | 复杂，需理解 Channel/Buffer/Selector 交互 |
| **适用场景** | 连接数少且固定，如后台任务 | 连接数多，如 IM/推送/游戏服务器 |

**举例说明**：
- BIO 处理 1000 个并发连接 → 需要 1000 个线程，每个线程至少 512KB 栈内存，光线程就要 500MB
- NIO 处理 1000 个并发连接 → 1 个线程 + 1 个 Selector，内存节省 99%

**两者关系**：NIO 是为了解决 BIO 在高并发下的性能瓶颈而引入的。两者可以共存：NIO 负责网络 I/O，文件 I/O 或数据库操作仍然可能用 BIO（或交给业务线程池处理）。

---

## 4. BIO、NIO 和 AIO 的区别？

| 维度 | BIO | NIO | AIO |
|------|-----|-----|-----|
| **同步/异步** | 同步 | 同步 | **异步** |
| **阻塞/非阻塞** | 阻塞 | 非阻塞 | 非阻塞 |
| **数据拷贝** | 应用程序自己调 read/write | 应用程序自己调 read/write | **操作系统负责拷贝，完成后通知应用** |
| **核心 API** | InputStream/ServerSocket | Channel/Buffer/Selector | AsynchronousSocketChannel/CompletionHandler |
| **线程模型** | 一连接一线程 | 一线程多连接（Selector） | OS 完成后回调（或线程池） |

**关键区别：同步 vs 异步**：

- **同步（BIO/NIO）**：数据就绪后，**应用程序主动调用** `read()` 把数据从内核空间拷到用户空间。这个拷贝过程需要应用线程参与。
- **异步（AIO）**：应用程序发起 I/O 请求后立即返回，**操作系统负责把数据拷好**，拷完后通过回调（`CompletionHandler`）通知应用。

**AIO 的现状**：
- Java 7 引入（NIO.2），在 Windows 上用 IOCP 实现，性能不错
- 在 Linux 上底层实现不完美（Linux 的异步 I/O 对 socket 支持有限，通常用线程池模拟），性能优势不明显
- Netty 在 Linux 上默认用基于 NIO 的 epoll 传输，而非 AIO
- 实际项目中 AIO 使用很少，了解概念即可

---

## 5. 说说 select、poll 和 epoll 的区别

三者都是 Linux 的 I/O 多路复用系统调用，允许一个进程监视多个文件描述符（fd）的读写就绪事件：

| 维度 | select | poll | epoll |
|------|--------|------|-------|
| **fd 数量限制** | 有，通常 1024（`FD_SETSIZE`） | 无 | 无 |
| **时间复杂度** | O(n)，遍历所有 fd | O(n)，遍历所有 fd | **O(1)，只处理就绪的 fd** |
| **内核实现** | 每次调用把整个 fd_set 从用户空间拷到内核，返回时再拷回来 | 同 select，用 pollfd 数组，无数量限制 | `epoll_ctl` 注册 fd 到内核事件表（红黑树），`epoll_wait` 从就绪链表取结果 |
| **触发方式** | 水平触发（LT） | 水平触发（LT） | 支持 LT 和**边缘触发（ET）** |

**epoll 为什么快**：
1. **内核维护事件表（红黑树）**：通过 `epoll_ctl` 注册/修改/删除 fd，无需每次调用都传全部 fd
2. **就绪链表**：fd 就绪时内核主动加入就绪链表，`epoll_wait` 直接取结果，无需遍历
3. **零拷贝**：注册时拷贝一次，之后不需要重复拷贝

**epoll 三个系统调用**：
```
epoll_create() → 创建 epoll 实例，返回 epfd
epoll_ctl(epfd, op, fd, event) → 注册/修改/删除要监视的 fd
epoll_wait(epfd, events, maxevents, timeout) → 等待事件，返回就绪的 fd 列表
```

**LT vs ET**：
- **水平触发（LT）**：只要 fd 可读/可写，每次 `epoll_wait` 都会通知。没读完下次继续通知，编程简单，不容易漏数据
- **边缘触发（ET）**：只在 fd 状态**发生变化**时通知一次（从不可读变为可读）。必须一次性读完所有数据（否则下次状态不变就不再通知了），性能更高但编程难度大，一般配合非阻塞 fd 使用

**在 Java NIO 中**：Linux 上 `Selector` 默认用 epoll，Netty 还额外提供了原生 epoll 传输（`EpollEventLoop`）以获得更好的性能。

---

## 6. 什么是 Selector？它的作用是什么？

**Selector** 是 Java NIO 中实现 I/O 多路复用的核心组件，一个 Selector 可以同时监视多个 Channel 的 I/O 事件。

**核心作用**：
1. **多路复用**：注册成千上万个 Channel，`select()` 等待，只有就绪的才被处理，避免为每个连接创建线程
2. **事件驱动**：返回就绪的 SelectionKey 集合，驱动应用程序响应
3. **非阻塞 I/O 的核心**：配合 Channel 的非阻塞模式，实现用少量线程高效管理大量连接

**核心方法**：
- `open()`：创建 Selector
- `select()`/`select(timeout)`：阻塞等待，至少一个就绪事件或超时
- `selectNow()`：非阻塞，立即返回
- `selectedKeys()`：返回已就绪的 SelectionKey 集合
- `wakeup()`：让阻塞在 `select()` 的线程立即返回
- `close()`：关闭 Selector

**内部三个键集（SelectorImpl）**：
- `keys`：所有注册的 SelectionKey
- `selectedKeys`：已就绪的键（每次 `select()` 后更新）
- `cancelledKeys`：已取消注册的键（待清理）

**注意**：处理完 selectedKeys 后必须调 `iterator.remove()`，否则下次 `select()` 还会返回这些键。

**在 Netty 中的角色**：`NioEventLoop` 内部封装一个 Selector，`run()` 方法不断循环 `select()` 和 `processSelectedKeys()`，是 Netty 事件循环的核心驱动。

---

## 7. NIO 的 Channel 和 Stream 的区别

| 维度 | Stream（流） | Channel（通道） |
|------|-------------|----------------|
| **方向** | 单向（InputStream 只读，OutputStream 只写） | **双向**（同一 Channel 可读可写） |
| **阻塞性** | 默认阻塞 | 可配置为非阻塞 |
| **传输单位** | 字节 | **缓冲区（Buffer）** |
| **多路复用** | 不支持 | 支持，配合 Selector |
| **分散/聚集** | 不支持 | 支持（`ScatteringByteChannel` / `GatheringByteChannel`） |
| **性能** | 较低（逐字节系统调用） | 较高（批量操作，支持零拷贝） |

---

## 高频追问：Netty 的 Channel 和 NIO 的 Channel 有什么关系？

**NIO Channel** 是 Java 标准库提供的底层 I/O 抽象，直接操作 Socket 或 File。

**Netty Channel** 是对 NIO Channel 的**更高层次封装**，提供了更强大的功能：

1. **统一抽象**：Netty Channel 不仅封装了 NIO Channel，还可以封装 OIO（BIO）、本地传输（`LocalChannel`）、嵌入式传输（`EmbeddedChannel`），统一了 API

2. **绑定了 EventLoop**：Netty Channel 创建时就绑定到一个 EventLoop，所有 I/O 操作都在那个线程上串行执行，保证线程安全

3. **绑定了 ChannelPipeline**：每个 Netty Channel 都有专属的 Pipeline，I/O 数据自动流过 Pipeline 里的所有 Handler

4. **更丰富的状态和操作**：`isActive()`/`isOpen()`/`isRegistered()`、`closeFuture()`、`attr()`（绑定用户属性）等

5. **ID 唯一标识**：每个 Netty Channel 有全局唯一的 `ChannelId`，便于日志追踪和连接管理

**举例**：`NioSocketChannel` 内部持有一个 `java.nio.channels.SocketChannel`，对 Netty 的 `write()`/`read()` 调用，最终会调到底层 NIO Channel 的对应方法，并配合 Selector 完成事件驱动。
