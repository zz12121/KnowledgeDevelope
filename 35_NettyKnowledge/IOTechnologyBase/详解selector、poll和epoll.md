# 详解selector、poll和epoll

## 概述

IO多路复用允许单个线程监控多个文件描述符，当任意描述符就绪时，线程收到通知处理相应IO操作，避免为每个连接创建单独线程。

## 三种多路复用机制

### Select

**原理**：调用select函数时，将用户空间的fd_set（位图）复制到内核，内核线性扫描所有文件描述符。

**缺点**：
- 文件描述符数量上限通常为1024
- 每次调用都需要两次内存拷贝
- 时间复杂度O(n)

### Poll

**原理**：与Select类似，但使用pollfd结构体链表，突破了1024限制。

**缺点**：
- 仍然需要全量扫描所有描述符
- 每次调用仍需完整内存拷贝

### Epoll

**原理**：基于事件驱动的回调机制，仅关注就绪的文件描述符。

**数据结构**：
- **红黑树**：存储所有注册的文件描述符，O(log n)操作
- **就绪链表**：仅保存有事件发生的描述符

**核心函数**：
- `epoll_create`：创建epoll实例
- `epoll_ctl`：注册/修改/删除监听的文件描述符
- `epoll_wait`：等待事件发生

## 触发模式

### LT（水平触发）

只要缓冲区有数据，每次epoll_wait都会通知。编程简单，不易丢失事件。

### ET（边缘触发）

仅在状态变化时通知一次，必须一次性处理完所有数据。适合高性能场景。

## 性能对比

| 特性 | Select | Poll | Epoll |
|------|--------|------|-------|
| 连接数上限 | 1024 | 无限制 | 无限制 |
| 时间复杂度 | O(n) | O(n) | O(1) |
| 内存拷贝 | 2次/调用 | 2次/调用 | 共享内存 |
| 触发方式 | 水平触发 | 水平触发 | LT/ET |
| 适用场景 | 连接<100 | 中等连接 | 高并发≥10000 |

## Java NIO中的实现

Java NIO通过Selector实现多路复用：

```java
// 创建Selector
Selector selector = Selector.open();

// 注册Channel
ServerSocketChannel channel = ServerSocketChannel.open();
channel.register(selector, SelectionKey.OP_ACCEPT);

// 轮询就绪事件
while (selector.select() > 0) {
    Set<SelectionKey> keys = selector.selectedKeys();
    for (SelectionKey key : keys) {
        if (key.isAcceptable()) {
            // 处理Accept事件
        } else if (key.isReadable()) {
            // 处理Read事件
        }
    }
}
```

## Netty中的优化

Netty针对不同平台提供了不同的EventLoopGroup：
- **Linux**：EpollEventLoopGroup（最高性能）
- **macOS/Windows**：NioEventLoopGroup

```java
// Linux环境使用Epoll
EventLoopGroup group = new EpollEventLoopGroup();

// NIO通用
EventLoopGroup group = new NioEventLoopGroup();
```

## 适用场景建议

1. **高并发长连接**：如WebSocket、消息推送，优先选用Epoll
2. **短连接场景**：Select/Poll/Epoll差别不大
3. **跨平台**：使用NIO Selector

## 总结

Epoll是Linux下高性能网络编程的首选，通过事件驱动机制避免了O(n)扫描，Netty框架对Epoll进行了良好封装，开发者可以直接享受其高性能优势。
