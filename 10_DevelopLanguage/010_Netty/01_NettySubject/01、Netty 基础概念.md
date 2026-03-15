### 一、Netty 基础概念

###### 1. 什么是 Netty？为什么要使用 Netty？

Netty 是一个基于 Java NIO 构建的**异步事件驱动网络框架**，它把 TCP/UDP 服务器/客户端开发复杂度大幅降低，让你专注业务逻辑而不是跟 Selector、ByteBuffer 死磕。

**为什么用 Netty 而不直接用 NIO？**

原生 NIO 的 API 非常难用，需要手动管理 Selector 的注册、轮询和事件集处理，ByteBuffer 的 flip/clear 更是一个坑一个坑。Netty 在此基础上做了全面封装，给你 Channel、EventLoop、Pipeline、Handler 四大抽象，写起来清晰很多。

性能方面，Netty 有几个杀手锏：**零拷贝**（FileRegion 走 sendfile，CompositeByteBuf 聚合不拷贝）、**内存池**（PooledByteBufAllocator 大幅减少 GC）、**无锁串行化**（一个 Channel 绑定一个 EventLoop 线程，天然线程安全）。

可靠性方面，Netty 修复了 JDK NIO 臭名昭著的 epoll 空轮询 bug（CPU 100% 问题），还内置了空闲连接检测、流量整形、SSL/TLS 支持。

生态方面，Dubbo、RocketMQ、Elasticsearch、Spark、gRPC-Java 都以 Netty 为底层传输，经过了大规模生产验证。

---

###### 2. Netty 有什么特点？

- **异步非阻塞**：基于事件驱动的 Reactor 线程模型，少量线程支撑大量并发连接，不会因某个连接阻塞而影响其他连接。
- **责任链处理**：通过 ChannelPipeline 将多个 ChannelHandler 串联成流水线，I/O 事件在链中传播，可扩展性极强。
- **灵活的线程模型**：支持单线程、主从多线程 Reactor 等多种模式，通过配置 EventLoopGroup 轻松切换。
- **统一 API**：Channel 接口屏蔽了 NIO、BIO、本地传输的差异，内置了 TCP、UDP、HTTP、WebSocket、HTTP/2、Protobuf 等协议支持。
- **高性能设计**：零拷贝 + 内存池 + 对象重用，从底层榨干性能。

---

###### 3. Netty 的优势有哪些？

**API 友好性**：相比直接用 Selector/ByteBuffer/ServerSocketChannel，Netty 的 API 更符合面向对象思想，学习曲线平缓，代码可维护性高。

**性能优势**：
- 高效的 Reactor 线程模型，避免线程频繁创建和上下文切换
- ByteBuf 支持动态扩容、读写索引分离、链式调用，内存池（PooledByteBufAllocator 维护 PoolArena）大幅提升分配效率
- 一个 Channel 绑定一个 EventLoop，无锁串行处理，消除了大量锁竞争

**功能完备**：从编解码、心跳检测、连接管理到流量整形，开箱即用；ChannelOption 和 AttributeKey 提供精细控制。

**生态与社区**：活跃的社区、丰富的第三方编解码器、大量开源项目采用，遇到问题容易找到答案。

---

###### 4. 为什么使用 Netty 而不用 NIO 或者其他 NIO 框架？

**vs 原生 NIO**：

原生 NIO 需要自行处理 Selector 注册/轮询、ByteBuffer 的 flip/compact 状态，极易出错。更大的问题是 JDK NIO 存在 epoll 空轮询 bug，直接用 NIO 可能在某些情况下 Selector.select() 不阻塞直接返回，CPU 跑满 100%。Netty 修复了这个问题，并把所有复杂性封装起来。

**vs Apache MINA**：

Netty 在内存池、零拷贝方面的优化更激进，社区更活跃，更新更快，API 设计更直观一致。Netty 4.x 对架构进行了重大重构，引入了更先进的内存池，性能显著领先。在 Java 高性能网络编程领域，Netty 已成为**事实标准**。

---

###### 5. Netty 和 Tomcat 的区别？

两者不是竞争关系，定位和抽象层次完全不同：

**Tomcat** 是 Servlet 容器，核心功能是处理 HTTP 协议、将请求映射到 Servlet/JSP，服务于 Web 应用的部署和运行。抽象层是 Servlet 和 Filter。

**Netty** 是底层网络应用框架，不局限于 HTTP，可以构建任何基于 TCP/UDP 的自定义协议服务器，抽象层是 Channel 和字节流。

**结合点**：Spring WebFlux 的默认服务器就是 Netty，用它来替代 Tomcat 提供响应式 HTTP 支持。高并发、需要长连接或自定义协议的场景，通常直接选择 Netty。

---

###### 6. Netty 的应用场景有哪些？

- **RPC 框架底层通信**：Dubbo、gRPC-Java、Apache Thrift 都以 Netty 作为默认传输层
- **游戏服务器**：大量长连接、高实时性、自定义二进制协议，Netty 是标配
- **即时通讯/推送系统**：微信、钉钉后台，维护海量用户长连接实时推送消息
- **物联网（IoT）**：终端设备与服务器建立长连接上报数据，通常基于 TCP 自定义协议
- **高性能 API 网关**：Spring Cloud Gateway、Zuul 2，处理大量并发 HTTP 请求
- **大数据节点通信**：Avro、Spark 数据节点间高效数据交换

---

###### 7. Netty 在微服务架构中的应用场景有哪些？

在微服务里，Netty 主要扮演"高速公路基础设施"的角色：

- **服务间 RPC 传输层**：Dubbo、gRPC 使用 Netty 构建高效异步的服务调用链路，这是最核心的应用
- **API 网关**：Spring Cloud Gateway 基于 Netty 的异步非阻塞特性，以更少资源处理更大流量
- **服务发现客户端**：与注册中心（Nacos、Consul）保持长连接，监听服务列表变更
- **消息推送与事件总线**：向客户端推送通知，WebSocket 长连接天然适合 Netty
- **Sidecar 代理/协议转换**：将内部 gRPC 协议转换为外部 HTTP/JSON，类似 Envoy 的部分功能

---

###### 8. 【高频追问】Netty 是如何解决 JDK NIO epoll 空轮询 bug 的？（迁移到十二章，此处为简短版）

JDK NIO 的 bug 表现是：Selector.select() 在没有 I/O 事件的情况下不阻塞直接返回 0，导致 EventLoop 空转，CPU 飙升至 100%。

Netty 的解决思路是**计数 + 重建**：在 NioEventLoop 的 run() 方法中，记录 select() 返回 0 的次数。如果在一个时间窗口内空轮询次数超过阈值（默认 512 次），Netty 判定触发了这个 bug，随即**新建一个 Selector**，把所有已注册的 Channel 从旧 Selector 迁移到新 Selector 上，然后关闭旧 Selector。这样彻底规避了问题。
