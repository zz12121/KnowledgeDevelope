---
tags:
  - Java/基础
  - Java/IO
aliases:
  - BIO NIO AIO
  - Selector多路复用
  - 零拷贝
  - epoll原理
date: 2026-03-18
---

# IO 与 NIO 深度解析

> **核心关键词**：BIO、NIO、AIO、Channel、Buffer、Selector、零拷贝、多路复用、Netty

---

## 一、IO 模型全景

```
5 种 IO 模型（UNIX 网络编程）：
  1. 阻塞 IO（BIO）       ← Java 传统 IO
  2. 非阻塞 IO（NIO）      ← Java NIO（轮询模式）
  3. IO 多路复用           ← Java NIO Selector（select/poll/epoll）
  4. 信号驱动 IO           ← 较少使用
  5. 异步 IO（AIO）        ← Java AIO（NIO 2.0）
```

---

## 二、传统 BIO（Blocking IO）

### 2.1 核心特点

- **同步阻塞**：线程发起 IO 操作后一直阻塞，直到数据就绪并拷贝完成
- **一连接一线程**：每个客户端连接对应一个线程，高并发下线程数爆炸

```java
// BIO 服务端示例
ServerSocket serverSocket = new ServerSocket(8080);
while (true) {
    Socket socket = serverSocket.accept();  // 阻塞等待连接
    new Thread(() -> {                       // 每个连接一个线程
        try (InputStream in = socket.getInputStream()) {
            byte[] buf = new byte[1024];
            int len;
            while ((len = in.read(buf)) != -1) {  // 阻塞读
                System.out.println(new String(buf, 0, len));
            }
        } catch (IOException e) { e.printStackTrace(); }
    }).start();
}
```

**缺点**：C10K 问题（1万并发连接 → 1万线程 → 内存/上下文切换开销巨大）

---

## 三、传统 IO 体系（java.io）

### 3.1 四大抽象基类

```
字节流                    字符流
InputStream              Reader
OutputStream             Writer
```

### 3.2 常用类与装饰器模式

```java
// 文件读写（字节流）
FileInputStream  / FileOutputStream

// 字节流 → 字符流转换
InputStreamReader  / OutputStreamWriter

// 带缓冲（推荐使用，减少系统调用次数）
BufferedReader     / BufferedWriter
BufferedInputStream / BufferedOutputStream

// 数据类型读写
DataInputStream    / DataOutputStream

// 对象序列化
ObjectInputStream  / ObjectOutputStream

// 内存流
ByteArrayInputStream / ByteArrayOutputStream
StringReader / StringWriter

// 典型用法（装饰器链）
try (BufferedReader br = new BufferedReader(
        new InputStreamReader(
            new FileInputStream("data.txt"), "UTF-8"))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}
```

### 3.3 File 类 vs Path（NIO）

```java
// 旧方式：File（功能有限，错误处理差）
File file = new File("/data/test.txt");
file.exists();
file.mkdir();
file.delete();

// 新方式：Path + Files（NIO 2.0，JDK 7+，推荐）
Path path = Paths.get("/data/test.txt");
Files.exists(path);
Files.createDirectories(path.getParent());
Files.delete(path);
Files.readAllLines(path, StandardCharsets.UTF_8);
Files.write(path, content.getBytes(), StandardOpenOption.APPEND);
Files.copy(src, dest, StandardCopyOption.REPLACE_EXISTING);
```

---

## 四、NIO（Non-blocking IO）三大核心组件

### 4.1 Buffer（缓冲区）

NIO 中所有 IO 操作都通过 Buffer 进行，数据读写的核心。

```java
// Buffer 的核心属性
capacity  // 总容量（固定）
limit     // 可读/写的上限
position  // 当前读/写位置
mark      // 标记位（可 reset 回到此位置）
// 关系：0 ≤ mark ≤ position ≤ limit ≤ capacity

// 写模式 → 读模式：flip()
ByteBuffer buffer = ByteBuffer.allocate(1024);
buffer.put("Hello".getBytes());  // 写入（position 移动）
buffer.flip();                   // 切换读模式（limit=position, position=0）
byte[] bytes = new byte[buffer.limit()];
buffer.get(bytes);               // 读取
buffer.clear();                  // 清空，准备下次写（position=0, limit=capacity）
buffer.compact();                // 压缩：将未读数据移到开头，继续写入

// 直接内存（堆外内存，零拷贝关键）
ByteBuffer directBuf = ByteBuffer.allocateDirect(1024);
// 数据直接在 OS 内存操作，避免 JVM 堆和 OS 内存间的拷贝
```

### 4.2 Channel（通道）

```java
// Channel：双向的，可读可写（Stream 是单向的）
FileChannel       // 文件 IO
SocketChannel     // TCP 客户端
ServerSocketChannel // TCP 服务端
DatagramChannel   // UDP

// 文件 Channel 示例（文件拷贝）
try (FileChannel src  = FileChannel.open(Paths.get("src.txt"), StandardOpenOption.READ);
     FileChannel dest = FileChannel.open(Paths.get("dest.txt"),
                            StandardOpenOption.WRITE, StandardOpenOption.CREATE)) {
    // 零拷贝：直接在内核空间完成数据传输，无需经过用户空间
    src.transferTo(0, src.size(), dest);
}

// 内存映射文件（MappedByteBuffer，大文件高性能读写）
MappedByteBuffer mbb = fileChannel.map(
    FileChannel.MapMode.READ_WRITE, 0, fileChannel.size());
mbb.put(0, (byte) 'X');  // 直接修改内存，OS 负责同步到磁盘
```

### 4.3 Selector（选择器）— 多路复用核心

```java
// Selector 允许一个线程监控多个 Channel 的 IO 状态
// 底层依赖 OS 的 select/poll/epoll（Linux 优先用 epoll）

Selector selector = Selector.open();

// 注册 Channel 到 Selector（Channel 必须是非阻塞模式）
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.configureBlocking(false);  // 必须设置非阻塞
ssc.bind(new InetSocketAddress(8080));
ssc.register(selector, SelectionKey.OP_ACCEPT);  // 监听连接事件

// 事件循环
while (true) {
    int readyCount = selector.select(1000);  // 阻塞直到有事件或超时
    if (readyCount == 0) continue;

    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> it = keys.iterator();
    while (it.hasNext()) {
        SelectionKey key = it.next();
        it.remove();  // 必须手动移除，否则重复处理

        if (key.isAcceptable()) {
            // 有新连接
            SocketChannel sc = ssc.accept();
            sc.configureBlocking(false);
            sc.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {
            // 有数据可读
            SocketChannel sc = (SocketChannel) key.channel();
            ByteBuffer buf = ByteBuffer.allocate(1024);
            int len = sc.read(buf);
            if (len == -1) { sc.close(); continue; }
            buf.flip();
            // 处理数据...
        }
    }
}
```

---

## 五、零拷贝（Zero Copy）

### 5.1 传统 IO 的数据拷贝问题

```
传统 read() + write() 文件传输（4次拷贝，4次上下文切换）：
  磁盘 → 内核缓冲区（DMA拷贝）
  内核缓冲区 → 用户空间（CPU拷贝）← 第1次上下文切换
  用户空间 → Socket缓冲区（CPU拷贝）← 第2次上下文切换
  Socket缓冲区 → 网卡（DMA拷贝）
```

### 5.2 零拷贝实现方式

```
sendfile（Linux）：2次拷贝，2次切换
  磁盘 → 内核缓冲区（DMA）→ Socket缓冲区（CPU/DMA）→ 网卡

mmap + write：3次拷贝，4次切换
  磁盘 → 内核缓冲区（DMA，用户空间共享）→ Socket缓冲区 → 网卡
  优点：用户态可直接操作内核缓冲区
```

```java
// Java 中的零拷贝
// 1. FileChannel.transferTo() → 底层用 sendfile
fileChannel.transferTo(0, fileSize, socketChannel);

// 2. MappedByteBuffer → 底层用 mmap
MappedByteBuffer buffer = fileChannel.map(MapMode.READ_ONLY, 0, fileSize);

// Netty 的 FileRegion 也是基于 sendfile
ctx.writeAndFlush(new DefaultFileRegion(file, 0, file.length()));
```

---

## 六、AIO（Asynchronous IO）

```java
// AIO = 真正的异步 IO，操作完成后回调通知，无需阻塞等待
// Java AIO 在 Linux 上底层实际是线程池模拟，并非真正 io_uring
// 实际项目中 Netty 默认不用 AIO，因为其 NIO + Selector 性能已足够好

AsynchronousServerSocketChannel server =
    AsynchronousServerSocketChannel.open().bind(new InetSocketAddress(8080));

server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
    @Override
    public void completed(AsynchronousSocketChannel client, Void attachment) {
        server.accept(null, this);  // 继续接受下一个连接
        ByteBuffer buf = ByteBuffer.allocate(1024);
        client.read(buf, buf, new CompletionHandler<Integer, ByteBuffer>() {
            @Override
            public void completed(Integer len, ByteBuffer b) {
                b.flip();
                client.write(b, b, null);  // echo 回去
            }
            @Override
            public void failed(Throwable e, ByteBuffer b) { }
        });
    }
    @Override
    public void failed(Throwable e, Void attachment) { }
});
```

---

## 八、epoll 原理（Selector 的 Linux 实现）

```
IO 多路复用在 Linux 上的三代演进：

select（POSIX）：
  - 将所有 fd 集合从用户态拷贝到内核态
  - 内核遍历所有 fd 检查就绪状态
  - 就绪 fd 返回给用户态（用户态再遍历找就绪的）
  - 限制：fd 数量上限 1024；每次都需要完整拷贝和遍历 → O(n)

poll：
  - 用链表代替 fd_set，突破 1024 限制
  - 仍然是线性扫描 O(n)，大量 fd 时性能差

epoll（Linux 2.6+，Java NIO Selector 的底层）：
  - 注册时（epoll_ctl）：将 fd 加入内核的红黑树
  - 等待时（epoll_wait）：不遍历，由驱动层将就绪 fd 加入就绪链表
  - 用户态只拿到就绪的 fd 列表 → O(1) 获取就绪事件

epoll 的两种触发模式：
  LT（Level Triggered，电平触发，默认）：
    fd 就绪 → 通知；若没有处理完，下次 epoll_wait 继续通知
    
  ET（Edge Triggered，边沿触发）：
    fd 从未就绪变为就绪 → 只通知一次；必须一次性处理完
    性能更高（减少通知次数），但编程更复杂，Nginx/Netty ET 模式常用

Java NIO 的 Selector 与 epoll 的对应关系：
  Selector.open()    → epoll_create
  channel.register() → epoll_ctl(ADD)
  selector.select()  → epoll_wait
  selectedKeys()     → 就绪事件列表
```

---

## 九、Netty 核心架构简述

```
Netty 是基于 Java NIO 的高性能异步网络框架，解决了原生 NIO 的使用复杂性。

核心组件：
  EventLoopGroup：线程组
    BossGroup：接受新连接（通常1个线程）
    WorkerGroup：处理 IO 读写（通常 CPU*2 个线程）

  Channel：网络连接的抽象（NioServerSocketChannel/NioSocketChannel）
  ChannelPipeline：处理器链（责任链模式）
  ChannelHandler：具体处理逻辑（Inbound/Outbound Handler）
  ByteBuf：Netty 的 Buffer（相比 NIO ByteBuffer，无需 flip/clear，支持引用计数）

Reactor 线程模型：
  主从 Reactor 多线程模型（Netty 默认）
  BossGroup（acceptor） → WorkerGroup（I/O + decode/encode）

与零拷贝的关系：
  Netty 的 FileRegion → sendfile（文件到网络零拷贝）
  CompositeByteBuf → 逻辑合并多个 ByteBuf，避免内存复制
  Direct ByteBuf → 堆外内存，减少 JVM 堆 ↔ 内核的拷贝
```

---

## 十、BIO / NIO / AIO 对比

| 维度 | BIO | NIO | AIO |
|------|-----|-----|-----|
| 阻塞 | 阻塞 | 非阻塞 | 异步非阻塞 |
| 线程模型 | 一连接一线程 | 一线程多连接（Selector） | 回调通知 |
| 吞吐量 | 低 | 高 | 高 |
| 编程复杂度 | 低 | 较高 | 较高 |
| 适用场景 | 连接数少 | 高并发（Netty） | 文件/少量高延迟连接 |
| 典型框架 | Tomcat(传统) | Netty、Netty-based | - |

---

## 十一、面试要点速查

| 问题 | 要点 |
|------|------|
| BIO 的问题 | 一连接一线程，C10K 下线程资源耗尽 |
| NIO 三大组件 | Buffer（数据容器）/ Channel（双向通道）/ Selector（多路复用） |
| flip() 作用 | 将 Buffer 从写模式切换到读模式（limit=position, position=0） |
| 零拷贝原理 | sendfile/mmap 让数据在内核空间直接传输，减少 CPU 拷贝和上下文切换 |
| transferTo 底层 | 调用 OS 的 sendfile 系统调用，2次DMA拷贝，无CPU拷贝 |
| epoll 相比 select/poll 的优势 | O(1) 获取就绪事件（不需要全量遍历）；无 fd 数量上限限制 |
| epoll LT vs ET | LT 多次通知（编程容易）；ET 仅通知一次变化（性能更高，需一次处理完）|
| Netty 相比原生 NIO 的优势 | 封装复杂性、零拷贝 ByteBuf、线程模型（主从 Reactor）、丰富的 Handler 生态 |
| Selector 底层 | Linux 用 epoll（O(1) 事件通知），Mac 用 kqueue，Windows 用 IOCP |
| 直接内存优势 | 避免 JVM 堆 ↔ OS 内存间的数据拷贝，适合大量 IO 操作 |
| NIO vs AIO | NIO 是同步非阻塞（需轮询/select），AIO 是异步（操作完成后回调） |


---

**相关面试题** → [[../../10_Developlanguage/001_Java/01_JavaBaseSubject/11、IO 流|11、IO 流]]
