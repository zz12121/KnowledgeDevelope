### 一、Netty 基础概念
###### 1. 什么是 Netty？为什么要使用 Netty？
###### 2. Netty 有什么特点？
###### 3. Netty 的优势有哪些？
###### 4. 为什么使用 Netty 而不用 NIO 或者其他 NIO 框架？
###### 5. Netty 和 Tomcat 的区别？
###### 6. Netty 的应用场景有哪些？
###### 7. Netty 在微服务架构中的应用场景有哪些？
### 二、Netty 核心组件
###### 1. Netty 的核心组件有哪些？简述其作用
###### 2. Netty 中有哪些重要组件？
###### 3. 说说 Netty 中网络通信的核心组件有哪些？
###### 4. 什么是 Channel？它的作用是什么？
###### 5. 什么是 ChannelHandler？它有哪些类型？
###### 6. Netty 中的 ChannelHandler 和 ChannelPipeline 的关系是什么？
###### 7. 说说 ChannelPipeline 的作用和工作原理
###### 8. 说说 inbound 和 outbound 的区别
###### 9. 什么是 ChannelHandlerContext？
###### 10. EventLoopGroup 了解么？和 EventLoop 啥关系？
###### 11. 什么是 EventLoop？它的作用是什么？
###### 12. Bootstrap 和 ServerBootstrap 了解么？
###### 13. 什么是 ChannelFuture？它的作用是什么？
###### 14. 说说 ByteBuf 有什么特点
###### 15. ByteBuf 和 NIO 的 ByteBuffer 有什么区别？
###### 16. ByteBuf 的分类有哪些？
### 三、Netty 线程模型
###### 1. 说说 Netty 的线程模型
###### 2. Netty 的 Reactor 线程模型是什么？
###### 3. 默认情况 Netty 起多少线程？何时启动？
###### 4. Netty 的 EventLoop 如何与线程绑定？
###### 5. Netty 如何保证线程安全？
### 四、Netty 启动流程
###### 1. 简单说说 Netty 服务端初始化并启动过程
###### 2. 说说 Netty 的整体工作机制
###### 3. Netty 客户端启动流程是怎样的？
###### 4. Netty 如何处理新连接接入？
### 五、TCP 粘包/拆包问题
###### 1. TCP 粘包/拆包的原因及解决方法
###### 2. 什么是 TCP 粘包/拆包？Netty 是如何解决的？
###### 3. Netty 提供了哪些解码器来处理粘包/拆包？
###### 4. LineBasedFrameDecoder 的工作原理是什么？
###### 5. DelimiterBasedFrameDecoder 的作用是什么？
###### 6. FixedLengthFrameDecoder 是如何工作的？
###### 7. LengthFieldBasedFrameDecoder 的使用场景是什么？
### 六、序列化与编解码
###### 1. 说说序列化
###### 2. 说说影响序列化性能的关键因素
###### 3. 说说 Java 序列化的缺点是什么？
###### 4. 除了 Java 序列化，还有哪些序列化？
###### 5. Netty 支持哪些编解码器？
###### 6. 如何自定义 Netty 的编解码器？
###### 7. Protobuf 在 Netty 中如何使用？
### 七、Netty 高性能设计
###### 1. Netty 高性能表现在哪些方面？
###### 2. Netty 是怎么实现高性能设计的？
###### 3. 简单说说 Netty 的零拷贝实现
###### 4. Netty 零拷贝体现在哪些方面？
###### 5. 如何优化 Netty 的吞吐量和延迟？
###### 6. Netty 如何减少 GC 压力？
###### 7. Netty 的 FastThreadLocal 是什么？
### 八、Netty 内存管理
###### 1. Netty 的内存管理机制是什么？
###### 2. Netty 是如何管理内存的？
###### 3. 如何设计一个内存池，或者内存分配器
###### 4. Netty 的内存池是如何实现的？
###### 5. 什么是 PooledByteBuf 和 UnpooledByteBuf？
###### 6. 如何保证 Netty 中不会发生内存泄漏？
###### 7. Netty 的引用计数机制是什么？
###### 8. 如何检测 Netty 中的内存泄漏？
###### 9. Direct Buffer 和 Heap Buffer 的区别是什么？
### 九、心跳机制与长连接
###### 1. Netty 支持哪些心跳类型设置？
###### 2. Netty 如何实现心跳机制？
###### 3. IdleStateHandler 的作用是什么？
###### 4. Netty 如何实现长连接？
###### 5. Netty 的连接管理是如何实现的？
###### 6. 如何处理 Netty 中的连接断线重连？
###### 7. Netty 如何实现连接池？
### 十、Netty 消息发送
###### 1. Netty 发送消息有几种方式？
###### 2. write 和 writeAndFlush 的区别是什么？
###### 3. 如何处理 Netty 中的高并发写操作？
###### 4. Netty 的写缓冲区是如何工作的？
###### 5. 如何控制 Netty 的写速度？
###### 6. ChannelOutboundBuffer 的作用是什么？
### 十一、Netty 异步模型
###### 1. Netty 的异步模型是怎样的？
###### 2. Future 和 Promise 的区别是什么？
###### 3. 如何使用 ChannelFuture 进行异步编程？
###### 4. Netty 中如何实现异步回调？
### 十二、Netty 常见问题处理
###### 1. Netty 是如何解决 JDK 中的 Selector BUG 的？
###### 2. Netty 如何解决 epoll 空轮询 bug？
###### 3. 什么是 Netty 的空轮询 bug？
###### 4. 如何处理 Netty 中的异常？
###### 5. Netty 的优雅关闭是如何实现的？
###### 6. 如何处理 Netty 中的半包读问题？
### 十三、Netty 设计模式
###### 1. Netty 中使用了哪些设计模式？
###### 2. 责任链模式在 Netty 中的应用
###### 3. 单例模式在 Netty 中的应用
###### 4. 工厂模式在 Netty 中的应用
###### 5. 观察者模式在 Netty 中的应用
### 十四、NIO 基础知识
###### 1. 说说 NIO 的组成
###### 2. NIO 是如何实现同步非阻塞的？
###### 3. NIO 和 BIO 到底有什么区别？有什么关系？
###### 4. BIO、NIO 和 AIO 的区别？
###### 5. 说说 select、poll 和 epoll 的区别
###### 6. 什么是 Selector？它的作用是什么？
###### 7. NIO 的 Channel 和 Stream 的区别
### 十五、协议支持
###### 1. 选 HTTP 作为数据传输有什么好处吗？
###### 2. 说说 HTTP 协议和 RPC 协议的区别
###### 3. Netty 如何支持 HTTP 协议？
###### 4. Netty 如何支持 WebSocket 协议？
###### 5. Netty 如何实现自定义协议？
###### 6. Netty 如何支持 SSL/TLS？
### 十六、Netty 安全性
###### 1. 如何在 Netty 中实现 SSL/TLS 加密？
###### 2. SslHandler 的作用是什么？
###### 3. 如何防止 Netty 中的 DDoS 攻击？
###### 4. 如何实现 Netty 的访问控制？
### 十七、Netty 实战应用
###### 1. 如何使用 Netty 实现一个简单的 HTTP 服务器？
###### 2. 如何使用 Netty 实现一个 WebSocket 服务器？
###### 3. 如何使用 Netty 实现 RPC 框架？
###### 4. 如何使用 Netty 实现 IM 即时通讯系统？
###### 5. 如何使用 Netty 实现文件传输？
###### 6. Netty 在 Dubbo 中的应用
###### 7. Netty 在 RocketMQ 中的应用
###### 8. Netty 在 Elasticsearch 中的应用
### 十八、Netty 配置与调优
###### 1. Netty 有哪些重要的配置参数？
###### 2. 如何调优 Netty 的性能？
###### 3. SO_BACKLOG 参数的作用是什么？
###### 4. SO_KEEPALIVE 参数的作用是什么？
###### 5. TCP_NODELAY 参数的作用是什么？
###### 6. SO_RCVBUF 和 SO_SNDBUF 的作用是什么？
###### 7. 如何设置 Netty 的线程池大小？
###### 8. 如何监控 Netty 的性能指标？
### 十九、Netty 测试与调试
###### 1. 如何对 Netty 应用进行单元测试？
###### 2. EmbeddedChannel 的作用是什么？
###### 3. 如何调试 Netty 应用？
###### 4. 如何使用 Netty 的日志记录功能？
### 二十、Netty 版本与更新
###### 1. Netty 3.x 和 Netty 4.x 的主要区别是什么？
###### 2. Netty 4.x 和 Netty 5.x 的区别是什么？
###### 3. 如何从 Netty 3.x 迁移到 Netty 4.x？