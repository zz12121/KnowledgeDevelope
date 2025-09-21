```
技术自由圈
```
# 牛逼的职业发展之路

40 岁老架构尼恩用一张图揭秘: Java 工程师的高端职业发展路径，走向食物链顶端的之路

链接：https://www.processon.com/view/link/618a2b62e0b34d73f7eb3cd


```
技术自由圈^
```
# 史上最全：价值 10 W 的架构师知识图谱

此图梳理于尼恩的多个 3 高生产项目：多个亿级人民币的大型 SAAS 平台和智慧城市项目

链接：https://www.processon.com/view/link/60fb9421637689719d


```
技术自由圈
```
# 牛逼的架构师哲学

40 岁老架构师尼恩对自己的 20 年的开发、架构经验总结

链接：https://www.processon.com/view/link/616f801963768961e9d9aec


```
技术自由圈
```
# 牛逼的 3 高架构知识宇宙

尼恩 3 高架构知识宇宙，帮助大家穿透 3 高架构，走向技术自由，远离中年危机

链接：https://www.processon.com/view/link/635097d2e0b34d40be778ab


```
技术自由圈
```
# 尼恩 Java 面试宝典

40 个专题（卷王专供+ 史上最全 + 2023 面试必备）
详情：https://www.cnblogs.com/crazymakercircle/p/13917138.html


```
技术自由圈^
```
# 未来职业，如何突围：三栖架构师


## 专题 25 ：Netty 面试题（史上最全、定期更

## 新）

#### 本文版本说明：V

```
此文的格式，由markdown 通过程序转成而来，由于很多表格，没有来的及调整，出现一个格式
问题，尼恩在此给大家道歉啦。
由于社群很多小伙伴，在面试，不断的交流最新的面试难题，所以，《尼恩Java面试宝典》， 后
面会不断升级，迭代。
```
```
本专题，作为 《尼恩Java面试宝典》专题之一， 《尼恩Java面试宝典》一共 41 个面试专题，后续
还会增加
```
#### 版本升级记录

###### 升级说明：

**V 88 升级说明（2023-07-23）：**

美团二面：epoll 性能那么高，为什么？

**V 3 升级说明（2022-05-22）： 一图搞懂 netty**

**Netty 很难** ，一直以来，没有一张图能比较深入的介绍清楚 netty

于是， **尼恩绘制了一张** : **Netty 架构图**

通过此图，应该对 Netty 的核心组件，有一个清晰的了解

这个图上都有： **io 事件怎么查询，怎么分发，数据怎么读取，数据怎么传播，数据怎么写入**


关于此图的链接： 可以参考疯狂创客圈微信群的历史记录

###### 《尼恩面试宝典》升级的规划：

后续基本上， **每一个月，都会发布一次** ，最新版本，可以扫描扫架构师尼恩微信，发送 “领取电子书”
获取。

尼恩的微信二维码在哪里呢 ？ 请参见文末

###### 面试问题交流说明：

如果遇到面试难题，或者职业发展问题，或者中年危机问题，都可以来疯狂创客圈社群交流，

加入交流群，加尼恩微信即可，

**入交流群** ，加尼恩微信即可，发送 **“入群”**


#### 史上最全 Java 面试题之：Netty 篇

#### 40 岁老架构师尼恩暗语：

如果在简历上写了 Netty，那么： **下面的面试题，最好都会**

如果要面试 **高端的开发、大厂的开发、或者架构师** ，那么： **简历上一定要写 netty**

**so，下面的面试题，越烂熟于心，越好**

#### Netty 是什么？


Netty 是一个异步事件驱动的网络应用程序框架，用于快速开发可维护的高性能协议服务器和客户端。

Netty 是基于 nio 的，它封装了 jdk 的 nio，让我们使用起来更加方法灵活。

#### 聊聊：JDK 原生 NIO 程序的问题？

JDK 原生也有一套网络应用程序 API，但是存在一系列问题，主要如下：

```
NIO的类库和API繁杂，使用麻烦，你需要熟练掌握Selector、ServerSocketChannel、
SocketChannel、ByteBuffer等
需要具备其它的额外技能做铺垫，例如熟悉Java多线程编程，因为NIO编程涉及到Reactor模式，
你必须对多线程和网路编程非常熟悉，才能编写出高质量的NIO程序
可靠性能力补齐，开发工作量和难度都非常大。
例如客户端面临断连重连、网络闪断、半包读写、失败缓存、网络拥塞和异常码流的处理等等，
NIO编程的特点是功能开发相对容易，但是可靠性能力补齐工作量和难度都非常大
JDK NIO的BUG，例如臭名昭著的select bug，它会导致Selector空轮询，最终导致CPU 100%。
官方声称在JDK1.6版本的update18修复了该问题，但是直到JDK1.7版本该问题仍旧存在，只不过
该bug发生概率降低了一些而已，它并没有被根本解决
```
#### Netty 的优势有哪些？

```
使用简单：封装了 NIO 的很多细节，使用更简单。
功能强大：预置了多种编解码功能，支持多种主流协议。
定制能力强：可以通过 ChannelHandler 对通信框架进行灵活地扩展。
性能高：通过与其他业界主流的 NIO 框架对比，Netty 的综合性能最优。
稳定：Netty 修复了已经发现的所有 NIO 的 bug，让开发人员可以专注于业务本身。
社区活跃：Netty 是活跃的开源项目，版本迭代周期短，bug 修复速度快。
```
#### Netty 的应用场景有哪些？

典型的应用有：

```
阿里分布式服务框架 Dubbo，默认使用 Netty 作为基础通信组件，
还有 RocketMQ 也是使用 Netty 作为通讯的基础。
```
Netty 常见的使用场景如下：

```
互联网行业 在分布式系统中，各个节点之间需要远程服务调用，高性能的RPC框架必不可少，
Netty作为异步高新能的通信框架,往往作为基础通信组件被这些RPC框架使用。 典型的应用有：阿
里分布式服务框架Dubbo的RPC框架使用Dubbo协议进行节点间通信，Dubbo协议默认使用Netty
作为基础通信组件，用于实现各进程节点之间的内部通信。
游戏行业 无论是手游服务端还是大型的网络游戏，Java语言得到了越来越广泛的应用。Netty作为
高性能的基础通信组件，它本身提供了TCP/UDP和HTTP协议栈。 非常方便定制和开发私有协议
栈，账号登录服务器，地图服务器之间可以方便的通过Netty进行高性能的通信
大数据领域 经典的Hadoop的高性能通信和序列化组件Avro的RPC框架，默认采用Netty进行跨界
点通信，它的Netty Service基于Netty框架二次封装实现
```
#### Netty 的特点


Netty 的对 JDK 自带的 NIO 的 API 进行封装，解决上述问题，主要特点有：

```
设计优雅 适用于各种传输类型的统一API - 阻塞和非阻塞Socket 基于灵活且可扩展的事件模型，可
以清晰地分离关注点 高度可定制的线程模型 - 单线程，一个或多个线程池 真正的无连接数据报套
接字支持（自3.1起）
高性能 、高吞吐、低延迟、低消耗
最小化不必要的内存复制
安全 完整的SSL / TLS和StartTLS支持
高并发：Netty 是一款基于 NIO（Nonblocking IO，非阻塞IO）开发的网络通信框架，对比于
BIO（Blocking I/O，阻塞IO），他的并发性能得到了很大提高。
传输快：Netty 的传输依赖于零拷贝特性，尽量减少不必要的内存拷贝，实现了更高效率的传输。
封装好：Netty 封装了 NIO 操作的很多细节，提供了易于使用调用接口。
社区活跃，不断更新 社区活跃，版本迭代周期短，发现的BUG可以被及时修复，同时，更多的新
功能会被加入
使用方便 详细记录的Javadoc，用户指南和示例 没有其他依赖项，JDK 5（Netty 3.x）或 6 （Netty
4.x）就足够了
```
#### Netty 高性能表现在哪些方面？

```
IO 线程模型：通过多线程Reactor反应器模式，在应用层实现异步非阻塞（异步事件驱动）架构，
用最少的资源做更多的事。
内存零拷贝：尽量减少不必要的内存拷贝，实现了更高效率的传输。
内存池设计：申请的内存可以重用，主要指直接内存。内部实现是用一颗二叉查找树管理内存分配
情况。 （ 具体请参考尼恩稍后的手写内存池 ）
对象池设计：Java对象可以重用，主要指Minior GC非常频繁的对象，如ByteBuffer。并且， 对象
池使用无锁架构 ，性能非常高。 （ 具体请参考尼恩稍后的手写对象池 ）
mpsc无锁编程：串形化处理读写, 避免使用锁带来的性能开销。
高性能序列化协议：支持 protobuf 等高性能序列化协议。
```
#### 聊聊：NIO 的组成？

Buffer：与 Channel 进行交互，数据是从 Channel 读入缓冲区，从缓冲区写入 Channel 中的

flip 方法 ： 反转此缓冲区，将 position 给 limit，然后将 position 置为 0 ，其实就是切换读写模式

clear 方法 ：清除此缓冲区，将 position 置为 0 ，把 capacity 的值给 limit。

rewind 方法 ： 重绕此缓冲区，将 position 置为 0

DirectByteBuffer 可减少一次系统空间到用户空间的拷贝。但 Buffer 创建和销毁的成本更高，不可控，
通常会用内存池来提高性能。直接缓冲区主要分配给那些易受基础系统的本机 I/O 操作影响的大型、持
久的缓冲区。如果数据量比较小的中小应用情况下，可以考虑使用 heapBuffer，由 JVM 进行管理。

Channel：表示 IO 源与目标打开的连接，是双向的，但不能直接访问数据，只能与 Buffer 进行交互。
通过源码可知，FileChannel 的 read 方法和 write 方法都导致数据复制了两次！

Selector 可使一个单独的线程管理多个 Channel，open 方法可创建 Selector，register 方法向多路复用
器器注册通道，可以监听的事件类型：读、写、连接、accept。注册事件后会产生一个 SelectionKey：
它表示 SelectableChannel 和 Selector 之间的注册关系，wakeup 方法：使尚未返回的第一个选择操作
立即返回，唤醒的


原因是：注册了新的 channel 或者事件；channel 关闭，取消注册；优先级更高的事件触发（如定时器
事件），希望及时处理。

Selector 在 Linux 的实现类是 EPollSelectorImpl，委托给 EPollArrayWrapper 实现，其中三个 native 方法
是对 epoll 的封装，而 EPollSelectorImpl. implRegister 方法，通过调用 epoll_ctl 向 epoll 实例中注册事
件，还将注册的文件描述符 (fd) 与 SelectionKey 的对应关系添加到 fdToKey 中，这个 map 维护了文件描述
符与 SelectionKey 的映射。

fdToKey 有时会变得非常大，因为注册到 Selector 上的 Channel 非常多（百万连接）；过期或失效的
Channel 没有及时关闭。fdToKey 总是串行读取的，而读取是在 select 方法中进行的，该方法是非线程安
全的。

Pipe：两个线程之间的单向数据连接，数据会被写到 sink 通道，从 source 通道读取

NIO 的服务端建立过程：Selector.open ()：打开一个 Selector；ServerSocketChannel.open ()：创建服
务端的 Channel；bind ()：绑定到某个端口上。并配置非阻塞模式；register ()：注册 Channel 和关注的
事件到 Selector 上；select () 轮询拿到已经就绪的事件

#### 聊聊：BIO、NIO 和 AIO 的区别？

BIO：一个连接一个线程，客户端有连接请求时服务器端就需要启动一个线程进行处理。线程开销大。
伪异步 IO：将请求连接放入线程池，一对多，但线程还是很宝贵的资源。

NIO：一个请求一个线程，但客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接
有 I/O 请求时才启动一个线程进行处理。

AIO：一个有效请求一个线程，客户端的 I/O 请求都是由 OS 先完成了再通知服务器应用去启动线程进行
处理，

BIO 是面向流的，NIO 是面向缓冲区的；

BIO 的各种流是阻塞的。而 NIO 是非阻塞的；

BIO 的 Stream 是单向的，而 NIO 的 channel 是双向的。

NIO 的特点：事件驱动模型、单线程处理多任务、非阻塞 I/O，I/O 读写不再阻塞，而是返回 0 、基于
block 的传输比基于流的传输更高效、更高级的 IO 函数 zero-copy、IO 多路复用大大提高了 Java 网络应用
的可伸缩性和实用性。基于 Reactor 线程模型。

在 Reactor 模式中，事件分发器等待某个事件或者可应用或个操作的状态发生，事件分发器就把这个事
件传给事先注册的事件处理函数或者回调函数，由后者来做实际的读写操作。如在 Reactor 中实现读：
注册读就绪事件和相应的事件处理器、事件分发器等待事件、事件到来，激活分发器，分发器调用事件
对应的处理器、事件处理器完成实际的读操作，处理读到的数据，注册新的事件，然后返还控制权。

#### Netty 和 Tomcat 的区别？

```
作用不同：Tomcat 是 Servlet 容器，可以视为 Web 服务器，而 Netty 是异步事件驱动的网络应
用程序框架和工具用于简化网络编程，例如TCP和UDP套接字服务器。
协议不同：Tomcat 是基于 http 协议的 Web 服务器，而 Netty 能通过编程自定义各种协议，因为
Netty 本身自己能编码/解码字节流，所有 Netty 可以实现，HTTP 服务器、FTP 服务器、UDP 服
务器、RPC 服务器、WebSocket 服务器、Redis 的 Proxy 服务器、MySQL 的 Proxy 服务器等
等。
```

#### 聊聊：Netty 是怎么实现高性能设计的？

Netty 作为高性能 IO 组件的扛鼎制作，

高性能设计的核心： 巧妙的结合高性能 IO 模型和线程模型，相得益彰，达到了高性能、高吞吐、低
延迟、低消耗的目标

其 I/O 模型高性能 epoll/select 模型

其线程模型为多线程 reactor 反应器模型

I/O 模型决定如何收发数据，

线程模型决定如何处理数据

相得益彰，在应用层达到了异步 IO 的效果。

#### 聊聊：I/O 模型

用什么样的通道将数据发送给对方，BIO、NIO 或者 AIO，I/O 模型在很大程度上决定了框架的性能

答案请参考《Java 高并发核心编程卷 1 》

书中，对 io 模型介绍得非常系统，并且不断完善

###### 阻塞 I/O

传统阻塞型 I/O (BIO) 可以用下图表示：


**特点**

```
每个请求都需要独立的线程完成数据read，业务处理，数据write的完整操作
```
**问题**

```
当并发数较大时，需要创建大量线程来处理连接，系统资源占用较大
连接建立后，如果当前线程暂时没有数据可读，则线程就阻塞在read操作上，造成线程资源浪费
```
###### I/O 复用模型

在 I/O 复用模型中，会用到 select，这个函数也会使进程阻塞，但是和阻塞 I/O 所不同的的，这两个函数
可以同时阻塞多个 I/O 操作，而且可以同时对多个读操作，多个写操作的 I/O 函数进行检测，直到有数据
可读或可写时，才真正调用 I/O 操作函数

Netty 的非阻塞 I/O 的实现关键是基于 I/O 复用模型，这里用 Selector 对象表示：


Netty 的 IO 线程 NioEventLoop 由于聚合了多路复用器 Selector，可以同时并发处理成百上千个客户端连
接。当线程从某客户端 Socket 通道进行读写数据时，若没有数据可用时，该线程可以进行其他任务。线
程通常将非阻塞 IO 的空闲时间用于在其他通道上执行 IO 操作，所以单独的线程可以管理多个输入和输
出通道。

由于读写操作都是非阻塞的，这就可以充分提升 IO 线程的运行效率，避免由于频繁 I/O 阻塞导致的线程
挂起，一个 I/O 线程可以并发处理 N 个客户端连接和读写操作，这从根本上解决了传统同步阻塞 I/O 一连
接一线程模型，架构的性能、弹性伸缩能力和可靠性都得到了极大的提升。

**基于 buffer**

传统的 I/O 是面向字节流或字符流的，以流式的方式顺序地从一个 Stream 中读取一个或多个字节, 因此
也就不能随意改变读取指针的位置。

在 NIO 中, 抛弃了传统的 I/O 流, 而是引入了 Channel 和 Buffer 的概念. 在 NIO 中, 只能从 Channel 中读取数
据到 Buffer 中或将数据 Buffer 中写入到 Channel。

基于 buffer 操作不像传统 IO 的顺序操作, NIO 中可以随意地读取任意位置的数据

#### 聊聊：AIO 是什么？


答案请参考《Java 高并发核心编程卷 1 》

书中，对 io 模型介绍得非常系统，并且不断完善

#### 聊聊：NIO 和 BIO 到底有什么区别？有什么关系？

```
1. NIO是以块的方式处理数据，BIO是以字节流或者字符流的形式去处理数据。
2. NIO是通过缓存区和通道的方式处理数据，BIO是通过InputStream和OutputStream流的方式处理
数据。
3. NIO的通道是双向的，BIO流的方向只能是单向的。
4. NIO采用的多路复用的同步非阻塞IO模型，BIO采用的是普通的同步阻塞IO模型。
5. NIO的效率比BIO要高，NIO适用于网络IO，BIO适用于文件IO。
```
#### 聊聊：NIO 是如何实现同步非阻塞的？

一个线程 Thread 使用一个选择器 Selector 监听多个通道 Channel 上的 IO 事件，从而让一个线程就可以
处理多个 IO 事件。通过配置监听的通道 Channel 为非阻塞，那么当 Channel 上的 IO 事件还未到达时，线
程会在 select 方法被挂起，让出 CPU 资源。直到监听到 Channel 有 IO 事件发生时，才会进行相应的响应
和处理。

Selector 能够检测多个注册的通道上是否有 IO 事件发生 (注意: 多个 Channel 以事件的方式可以注册到同
一个 Selector)，如果有事件发生，便获取事件然后针对每个事件进行相应的处理。这样就可以只用一个
单线程去管理多个通道，也就是管理多个连接和请求。

Selector 只有在通道上有真正的 IO 事件发生时，才会进行相应的处理，这就不必为每个连接都创建一个
线程，避免线程资源的浪费和多线程之间的上下文切换导致的开销。

#### 聊聊：BIO 和 NIO 应用场景

1 、 **BIO** 方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应
用中，JDK 1.4 以前的唯一选择。

2 、 **NIO** 方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，弹幕系统，服务器
间通讯等。JDK 1.4 开始支持。

#### 聊聊：阻塞/非阻塞区别

答案请参考《Java 高并发核心编程卷 1 》

书中，对 io 模型介绍得非常系统，并且不断完善

#### 聊聊：同步/异步区别


答案请参考《Java 高并发核心编程卷 1 》

书中，对 io 模型介绍得非常系统，并且不断完善

#### 聊聊：select、poll 和 epoll 的区别

**消息传递方式：**

select：内核需要将消息传递到用户空间，需要内核的拷贝动作；

poll：同上；

epoll：通过内核和用户空间共享一块内存来实现，性能较高；

**文件句柄剧增后带来的 IO 效率问题：**

select：因为每次调用都会对连接进行线性遍历，所以随着 FD 剧增后会造成遍历速度的“线性下降”的性
能问题；

poll：同上；

epoll：由于 epoll 是根据每个 FD 上的 callable 函数来实现的，只有活跃的 socket 才会主动调用 callback，
所以在活跃 socket 较少的情况下，使用 epoll 不会对性能产生线性下降的问题，如果所有 socket 都很活跃
的情况下，可能会有性能问题；

**支持一个进程所能打开的最大连接数：**

select：单个进程所能打开的最大连接数，是由 FD_SETSIZE 宏定义的，其大小是 32 个整数大小（在 32 位
的机器上，大小是 32 _32,64_ 位机器上 _FD_SETSIZE=32_ 64 ），我们可以对其进行修改，然后重新编译内核，
但是性能无法保证，需要做进一步测试；

poll：本质上与 select 没什么区别，但是他没有最大连接数限制，他是基于链表来存储的；

epoll：虽然连接数有上线，但是很大，1 G 内存的机器上可以打开 10 W 左右的连接；

#### 聊聊: 主要的 IO 线程模型有哪些?

数据报如何读取？读取之后的编解码在哪个线程进行，编解码后的消息如何派发，线程模型的不同，对
性能的影响也非常大。

###### Connection Per Thread（一个线程处理一个连接）线程模式

在 Java 的 OIO 编程中，最原始的网络服务器程序，一般是用一个 while 循环，不断地监听端口是否有新的
连接。

如果有，那么就调用一个处理函数来完成传输处理，示例代码如下：


这种方法的最大问题是：如果前一个网络连接的 handle（socket）没有处理完，那么后面的新连接没法
被服务端接收，于是后面的请求就会被阻塞住，这样就导致服务器的吞吐量太低。

这对于服务器来说，这是一个严重的问题。

为了解决这个严重的连接阻塞问题，出现了一个极为经典模式：Connection Per Thread（一个线程处
理一个连接）模式。示例代码如下：

```
while(true){
```
```
socket = accept(); //阻塞，接收连接
```
```
handle(socket) ; //读取数据、业务处理、写入结果
```
```
}
```
```
package com.crazymakercircle.iodemo.OIO;
//...省略import导入的Java类
class ConnectionPerThread implements Runnable {
public void run() {
try {
//服务器监听socket
ServerSocketserverSocket =
new ServerSocket(NioDemoConfig.SOCKET_SERVER_PORT);
while (!Thread.interrupted()) {
Socket socket = serverSocket.accept();
//接收一个连接后，为socket连接，新建一个专属的处理器对象
Handler handler = new Handler(socket);
//创建新线程，专门负责一个连接的处理
new Thread(handler).start();
}
```
```
} catch (IOException ex) { /* 处理异常 */ }
}
//处理器，这里将内容回显到客户端
static class Handler implements Runnable {
final Socket socket;
Handler(Socket s) {
socket = s;
}
public void run() {
while (true) {
try {
byte[] input = new byte[1024];
/* 读取数据 */
socket.getInputStream().read(input);
/* 处理业务逻辑，获取处理结果*/
byte[] output =null;
/* 写入结果 */
socket.getOutputStream().write(output);
} catch (IOException ex) { /*处理异常*/ }
}
```

以上示例代码中，对于每一个新的网络连接都分配给一个线程。每个线程都独自处理自己负责的 socket
连接的输入和输出。当然，服务器的监听线程也是独立的，任何的 socket 连接的输入和输出处理，不会
阻塞到后面新 socket 连接的监听和建立，这样，服务器的吞吐量就得到了提升。早期版本的 Tomcat 服
务器，就是这样实现的。

Connection Per Thread 模式（一个线程处理一个连接）的优点是：解决了前面的新连接被严重阻塞的
问题，在一定程度上，较大的提高了服务器的吞吐量。

Connection Per Thread 模式的缺点是：对应于大量的连接，需要耗费大量的线程资源，对线程资源要
求太高。在系统中，线程是比较昂贵的系统资源。如果线程的数量太多，系统无法承受。而且，线程的
反复创建、销毁、线程的切换也需要代价。因此，在高并发的应用场景下，多线程 OIO 的缺陷是致命
的。

新的问题来了：如何减少线程数量，比如说让一个线程同时负责处理多个 socket 连接的输入和输出，行
不行呢？

看上去，没有什么不可以。但是，实际上作用不大。为什么呢？传统 OIO 编程中每一次 socket 传输的 IO
读写处理，都是阻塞的。在同一时刻，一个线程里只能处理一个 socket 的读写操作，前一个 socket 操作
被阻塞了，其他连接的 IO 操作同样无法被并行处理。所以在 OIO 中，即使是一个线程同时负责处理多个
socket 连接的输入和输出，同一时刻，该线程也只能处理一个连接的 IO 操作。

如何解决 Connection Per Thread 模式的巨大缺陷呢？一个有效途径是：使用 Reactor 反应器模式。用反
应器模式对线程的数量进行控制，做到一个线程处理大量的连接。

###### Reactor 线程模型

Reactor 是反应堆/反应器的意思，Reactor 模型，是指通过一个或多个输入同时传递给服务处理器的服
务请求的 **事件驱动处理模式** 。服务端程序处理传入多路请求，并将它们同步分派给请求对应的处理线
程，Reactor 模式也叫 Dispatcher 模式，即 I/O 多了复用统一监听事件，收到事件后分发 (Dispatch 给某
进程)，是编写高性能网络服务器的必备技术之一。

Reactor 模型中有 2 个关键组成：

```
Reactor Reactor在一个单独的线程中运行，负责监听和分发事件，分发给适当的处理程序来对IO
事件做出反应。 它就像公司的电话接线员，它接听来自客户的电话并将线路转移到适当的联系人
Handlers 处理程序执行I/O事件要完成的实际事件，类似于客户想要与之交谈的公司中的实际官
员。Reactor通过调度适当的处理程序来响应I/O事件，处理程序执行非阻塞操作
```
```
}
}
}
```

取决于 Reactor 的数量和 Hanndler 线程数量的不同，Reactor 模型有 3 个变种

```
单Reactor单线程
单Reactor多线程
主从Reactor多线程
```
可以这样理解，Reactor 就是一个执行 while (true) { selector.select (); ...}循环的线程，会源源不断的产
生新的事件，称作反应堆很贴切。

篇幅关系，这里不再具体展开 Reactor 特性、优缺点比较，有兴趣的读者可以参考

《Java 高并发核心编程卷 1 》

书中，对 Reactor 模型介绍得非常系统，并且不断完善

###### Netty 线程模型

Netty 主要 **基于主从 Reactors 多线程模型** （如下图）做了一定的修改，其中主从 Reactor 多线程模型有
多个 Reactor：MainReactor 和 SubReactor：

```
MainReactor负责客户端的连接请求，并将请求转交给SubReactor
SubReactor负责相应通道的IO读写请求
非IO请求（具体逻辑处理）的任务则会直接写入队列，等待worker threads进行处理
```
这里引用 Doug Lee 大神的 Reactor 介绍：Scalable IO in Java 里面关于主从 Reactor 多线程模型的图


特别说明的是：

虽然 Netty 的线程模型基于主从 Reactor 多线程，借用了 MainReactor 和 SubReactor 的结构，

但是实际实现上，SubReactor 和 Worker 线程在同一个线程池中：

上面代码中的 bossGroup 和 workerGroup 是 Bootstrap 构造方法中传入的两个对象，这两个 group 均是
线程池

```
bossGroup线程池则只是在bind某个端口后，获得其中一个线程作为MainReactor，专门处理端
口的accept事件， 每个端口对应一个boss线程
workerGroup线程池会被各个SubReactor和worker线程充分利用
```
###### 异步处理

异步的概念和同步相对。当一个异步过程调用发出后，调用者不能立刻得到结果。实际处理这个调用的
部件在完成后，通过状态、通知和回调来通知调用者。

Netty 中的 I/O 操作是异步的，包括 bind、write、connect 等操作会简单的返回一个 ChannelFuture，调
用者并不能立刻获得结果，通过 Future-Listener 机制，用户可以方便的主动获取或者通过通知机制获得
IO 操作结果。

当 future 对象刚刚创建时，处于非完成状态，调用者可以通过返回的 ChannelFuture 来获取操作执行的
状态，注册监听函数来执行完成后的操，常见有如下操作：

```
通过isDone方法来判断当前操作是否完成
通过isSuccess方法来判断已完成的当前操作是否成功
```
```
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();
ServerBootstrap server = new ServerBootstrap();
server.group(bossGroup, workerGroup)
.channel(NioServerSocketChannel.class)
```

```
通过getCause方法来获取已完成的当前操作失败的原因
通过isCancelled方法来判断已完成的当前操作是否被取消
通过addListener方法来注册监听器，当操作已完成(isDone方法返回完成)，将会通知指定的监听
器；如果future对象已完成，则理解通知指定的监听器
```
例如下面的的代码中绑定端口是异步操作，当绑定操作处理完，将会调用相应的监听器处理逻辑

相比传统阻塞 I/O，执行 I/O 操作后线程会被阻塞住, 直到操作完成；异步处理的好处是不会造成线程阻
塞，线程在 I/O 操作期间可以执行别的程序，在高并发情形下会更稳定和更高的吞吐量。

#### Netty 架构设计

前面介绍完 Netty 相关一些理论介绍，下面从功能特性、模块组件、运作过程来介绍 Netty 的架构设计

###### 功能特性

```
传输服务 支持BIO和NIO
容器集成 支持OSGI、JBossMC、Spring、Guice容器
协议支持 HTTP、Protobuf、二进制、文本、WebSocket等一系列常见协议都支持。 还支持通过
实行编码解码逻辑来实现自定义协议
Core核心 可扩展事件模型、通用通信API、支持零拷贝的ByteBuf缓冲对象
```
###### 模块组件

**Bootstrap、ServerBootstrap**

```
serverBootstrap.bind(port).addListener(future -> {
if (future.isSuccess()) {
System.out.println(new Date() + ": 端口[" + port + "]绑定成功!");
} else {
System.err.println("端口[" + port + "]绑定失败!");
}
});
```

Bootstrap 意思是引导，一个 Netty 应用通常由一个 Bootstrap 开始，主要作用是配置整个 Netty 程序，串
联各个组件，Netty 中 Bootstrap 类是客户端程序的启动引导类，ServerBootstrap 是服务端启动引导
类。

**Future、ChannelFuture**

正如前面介绍，在 Netty 中所有的 IO 操作都是异步的，不能立刻得知消息是否被正确处理，但是可以过
一会等它执行完成或者直接注册一个监听，具体的实现就是通过 Future 和 ChannelFutures，他们可以
注册一个监听，当操作执行成功或失败时监听会自动触发注册的监听事件。

**Channel**

Netty 网络通信的组件，能够用于执行网络 I/O 操作。 Channel 为用户提供：

```
当前网络连接的通道的状态（例如是否打开？是否已连接？）
网络连接的配置参数 （例如接收缓冲区大小）
提供异步的网络I/O操作(如建立连接，读写，绑定端口)，异步调用意味着任何I / O调用都将立即返
回，并且不保证在调用结束时所请求的I / O操作已完成。调用立即返回一个ChannelFuture实例，
通过注册监听器到ChannelFuture上，可以I / O操作成功、失败或取消时回调通知调用方。
支持关联I/O操作与对应的处理程序
```
不同协议、不同的阻塞类型的连接都有不同的 Channel 类型与之对应，下面是一些常用的 Channel 类
型

```
NioSocketChannel，异步的客户端 TCP Socket 连接
NioServerSocketChannel，异步的服务器端 TCP Socket 连接
NioDatagramChannel，异步的 UDP 连接
NioSctpChannel，异步的客户端 Sctp 连接
NioSctpServerChannel，异步的 Sctp 服务器端连接 这些通道涵盖了 UDP 和 TCP网络 IO以及文
件 IO.
```
**Selector**

Netty 基于 Selector 对象实现 I/O 多路复用，通过 Selector, 一个线程可以监听多个连接的 Channel 事件,
当向一个 Selector 中注册 Channel 后，Selector 内部的机制就可以自动不断地查询 (select) 这些注册的
Channel 是否有已就绪的 I/O 事件 (例如可读, 可写, 网络连接完成等)，这样程序就可以很简单地使用一个
线程高效地管理多个 Channel 。

**NioEventLoop**

NioEventLoop 中维护了一个线程和任务队列，支持异步提交执行任务，线程启动时会调用
NioEventLoop 的 run 方法，执行 I/O 任务和非 I/O 任务：

```
I/O任务 即selectionKey中ready的事件，如accept、connect、read、write等，由
processSelectedKeys方法触发。
非IO任务 添加到taskQueue中的任务，如register0、bind0等任务，由runAllTasks方法触发。
```
两种任务的执行时间比由变量 ioRatio 控制，默认为 50 ，则表示允许非 IO 任务执行的时间与 IO 任务的执
行时间相等。

**NioEventLoopGroup**

NioEventLoopGroup，主要管理 eventLoop 的生命周期，可以理解为一个线程池，内部维护了一组线
程，每个线程 (NioEventLoop) 负责处理多个 Channel 上的事件，而一个 Channel 只对应于一个线程。

**ChannelHandler**


ChannelHandler 是一个接口，处理 I / O 事件或拦截 I / O 操作，并将其转发到其 ChannelPipeline (业务
处理链) 中的下一个处理程序。

ChannelHandler 本身并没有提供很多方法，因为这个接口有许多的方法需要实现，方便使用期间，可
以继承它的子类：

```
ChannelInboundHandler用于处理入站I / O事件
ChannelOutboundHandler用于处理出站I / O操作
```
或者使用以下适配器类：

```
ChannelInboundHandlerAdapter用于处理入站I / O事件
ChannelOutboundHandlerAdapter用于处理出站I / O操作
ChannelDuplexHandler用于处理入站和出站事件
```
**ChannelHandlerContext**

保存 Channel 相关的所有上下文信息，同时关联一个 ChannelHandler 对象

**ChannelPipline**

保存 ChannelHandler 的 List，用于处理或拦截 Channel 的入站事件和出站操作。 ChannelPipeline 实现
了一种高级形式的拦截过滤器模式，使用户可以完全控制事件的处理方式，以及 Channel 中各个的
ChannelHandler 如何相互交互。

下图引用 Netty 的 Javadoc 4.1 中 ChannelPipline 的说明，描述了 ChannelPipeline 中 ChannelHandler 通
常如何处理 I/O 事件。 I/O 事件由 ChannelInboundHandler 或 ChannelOutboundHandler 处理，并通过
调用 ChannelHandlerContext 中定义的事件传播方法（例如
ChannelHandlerContext. fireChannelRead（Object）和
ChannelOutboundInvoker. write（Object））转发到其最近的处理程序。

```
I/O Request
via Channel or
ChannelHandlerContext
|
+---------------------------------------------------+---------------+
| ChannelPipeline | |
| \|/ |
| +---------------------+ +-----------+----------+ |
| | Inbound Handler N | | Outbound Handler 1 | |
| +----------+----------+ +-----------+----------+ |
| /|\ | |
| | \|/ |
| +----------+----------+ +-----------+----------+ |
| | Inbound Handler N-1 | | Outbound Handler 2 | |
| +----------+----------+ +-----------+----------+ |
| /|\. |
|.. |
| ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
| [ method call] [method call] |
|.. |
|. \|/ |
| +----------+----------+ +-----------+----------+ |
| | Inbound Handler 2 | | Outbound Handler M-1 | |
| +----------+----------+ +-----------+----------+ |
| /|\ | |
| | \|/ |
| +----------+----------+ +-----------+----------+ |
```

入站事件由自下而上方向的入站处理程序处理，如图左侧所示。入站 Handler 处理程序通常处理由图底
部的 I / O 线程生成的入站数据。通常通过实际输入操作（例如 SocketChannel. read（ByteBuffer））
从远程读取入站数据。

出站事件由上下方向处理，如图右侧所示。出站 Handler 处理程序通常会生成或转换出站传输，例如
write 请求。 I/O 线程通常执行实际的输出操作，例如 SocketChannel. write（ByteBuffer）。

在 Netty 中每个 Channel 都有且仅有一个 ChannelPipeline 与之对应, 它们的组成关系如下:

一个 Channel 包含了一个 ChannelPipeline, 而 ChannelPipeline 中又维护了一个由
ChannelHandlerContext 组成的双向链表, 并且每个 ChannelHandlerContext 中又关联着一个
ChannelHandler。入站事件和出站事件在一个双向链表中，入站事件会从链表 head 往后传递到最后一
个入站的 handler，出站事件会从链表 tail 往前传递到最前一个出站的 handler，两种类型的 handler 互不
干扰。

#### 聊聊：Netty 服务端过程初始化并启动过程

初始化并启动 Netty 服务端过程如下：

```
| | Inbound Handler 1 | | Outbound Handler M | |
| +----------+----------+ +-----------+----------+ |
| /|\ | |
+---------------+-----------------------------------+---------------+
| \|/
+---------------+-----------------------------------+---------------+
| | | |
| [ Socket.read() ] [ Socket.write() ] |
| |
| Netty Internal I/O Threads (Transport Implementation) |
+-------------------------------------------------------------------+
```
```
123456789101112131415161718192021222324252627282930313233343536373839
```
```
public static void main(String[] args) {
// 创建mainReactor
NioEventLoopGroup boosGroup = new NioEventLoopGroup();
// 创建工作线程组
NioEventLoopGroup workerGroup = new NioEventLoopGroup();
```
```
final ServerBootstrap serverBootstrap = new ServerBootstrap();
serverBootstrap
// 组装NioEventLoopGroup
.group(boosGroup, workerGroup)
// 设置channel类型为NIO类型
```

基本过程如下：

1 初始化创建 2 个 NioEventLoopGroup，其中 boosGroup 用于 Accetpt 连接建立事件并分发请求，
workerGroup 用于处理 I/O 读写事件和业务逻辑

2 基于 ServerBootstrap (服务端启动引导类)，配置 EventLoopGroup、Channel 类型，连接参数、配置
入站、出站事件 handler

3 绑定端口，开始工作

###### 服务端 Netty 的工作架构图

结合上面的介绍的 Netty Reactor 模型，介绍服务端 Netty 的工作架构图：

```
.channel(NioServerSocketChannel.class)
// 设置连接配置参数
.option(ChannelOption.SO_BACKLOG, 1024 )
.childOption(ChannelOption.SO_KEEPALIVE, true)
.childOption(ChannelOption.TCP_NODELAY, true)
// 配置入站、出站事件handler
.childHandler(new ChannelInitializer<NioSocketChannel>() {
@Override
protected void initChannel(NioSocketChannel ch) {
// 配置入站、出站事件channel
ch.pipeline().addLast(...);
ch.pipeline().addLast(...);
}
});
```
```
// 绑定端口
int port = 8080 ;
serverBootstrap.bind(port).addListener(future -> {
if (future.isSuccess()) {
System.out.println(new Date() + ": 端口[" + port + "]绑定成功!");
} else {
System.err.println("端口[" + port + "]绑定失败!");
}
});
}
```

server 端包含 1 个 Boss NioEventLoopGroup 和 1 个 Worker NioEventLoopGroup，

NioEventLoopGroup 相当于 1 个事件循环组，这个组里包含多个事件循环 NioEventLoop，每个
NioEventLoop 包含 1 个 selector 和 1 个事件循环线程。

每个 Boss NioEventLoop 循环执行的任务包含 3 步：

1 轮询 accept 事件

2 处理 accept I/O 事件，与 Client 建立连接，生成 NioSocketChannel，并将 NioSocketChannel 注册到某
个 Worker NioEventLoop 的 Selector 上 *3 处理任务队列中的任务，runAllTasks。

任务队列中的任务包括用户调用 eventloop. execute 或 schedule 执行的任务，或者其它线程提交到该
eventloop 的任务。

每个 Worker NioEventLoop 循环执行的任务包含 3 步：

1 轮询 read、write 事件；

2 处 I/O 事件，即 read、write 事件，在 NioSocketChannel 可读、可写事件发生时进行处理

3 处理任务队列中的任务，runAllTasks。

###### 任务队列的使用场景


其中任务队列中的 task 有 3 种典型使用场景

1 用户程序自定义的普通任务

2 非当前 reactor 线程调用 channel 的各种方法

例如在推送系统的业务线程里面，根据用户的标识，找到对应的 channel 引用，然后调用 write 类方法向
该用户推送消息，就会进入到这种场景。最终的 write 会提交到任务队列中后被异步消费。

3 用户自定义定时任务

#### 聊一下：Netty ByteBuf 的特点？

这里想要比较两种 Buffer, 对比 ByteBuffer 得出 ByteBuf 的优点点,

我们首先要做的就是总结 ByteBuf 的特点以及相比 ByteBuffer, 这个特点如何成为优点：
**一：ByteBuf 读写指针**
在 ByteBuffer 中, 读写指针都是 position,

而在 ByteBuf 中, 读写指针分别为 readerIndex 和 writerIndex,

直观看上去 ByteBuffer 仅用了一个指针就实现了两个指针的功能, 节省了变量, 但是当对于 ByteBuffer 的
读写状态切换的时候必须要调用 flip 方法, 而当下一次写之前, 必须要将 Buffe 中的内容读完, 再调用 clear 方
法。

每次读之前调用 flip, 写之前调用 clear, 这样无疑给开发带来了繁琐的步骤, 而且内容没有读完是不能写的,
这样非常不灵活。

相比之下我们看看 ByteBuf, 读的时候仅仅依赖 readerIndex 指针, 写的时候仅仅依赖 writerIndex 指针, 不
需每次读写之前调用对应的方法, 而且没有必须一次读完的限制。
**二：ByteBuf 引用计数**
ByteBuf 扩展了 ReferenceCountered 接口, 这个接口定义的功能主要是引用计数：
ReferenceCountered 接口定义
也就是所有对 ByteBuf 的实现, 都要实现引用计数, Netty 对 Buffer 资源进行了显式的管理, 这部分要结合

```
ctx.channel().eventLoop().execute(new Runnable() {
@Override
public void run() {
//...
}
});
```
```
ctx.channel().eventLoop().schedule(new Runnable() {
@Override
public void run() {
```
```
}
}, 60 , TimeUnit.SECONDS);
```

Netty 的内存池技术理解, 当 Buffer 引用+1 的时候, 需要调用 retain 来让 refCnt+1, 当 Buffer 引用数-1 的时候
需要调用 release 来让 refCnt-1, 当 refCnt 变为 0 的时候 Netty 为 pooled 和 unpooled 的不同 buffer 提供了不
同的实现, 通常对于非内存池的用法, Netty 把 Buffer 的内存回收交给了垃圾回收器, 对于内存池的用
法, Netty 对内存的回收实际上是回收到内存池内, 以提供下一次的申请所使用, 关于内存池这部分可以参考
我之前的一篇文章。
**三：池化 Buffer 资源**
由于 Netty 是一个 NIO 网络框架, 因此对于 Buffer 的使用如果基于直接内存 DirectBuffer：实现的话, 将会
大大提高 I/O 操作的效率, 然而 DirectBuffer 和 HeapBuffer 相比之下除了 I/O 操作效率高之外还有一个天生
的缺点, 即对于 DirectBuffer 的申请相比 HeapBuffer 效率更低, 因此 Netty 结合引用计数实现了
PolledBuffer, 即池化的用法, 当引用计数等于 0 的时候, Netty 将 Buffer 回收致池中, 在下一次申请 Buffer 的
没某个时刻会被复用。Netty 这样做的基本想法是我们花了很大的力气申请了一块内存, 不能轻易让他被
回收呀, 能重复利用当然重复利用咯。
**四：ByteBuffer 才能和 Channel 打交道**
归根结底, 站在 NIO 的立场上所有的缓冲区要想和 Channel 打交道, 换句话说也就是从网络 Channel 读取
数据的时候, 都是从 Channel 到 ByteBuffer, 从缓冲区写的网上上的时候, 都是从 ByteBuffer 到 Channel。
因此, 当 Netty 监听到 I/O 读事件的时候, 会将自己流从 Channel 读到 ByteBuffer 而不是 ByteBuf, see below:
return in.read ((ByteBuffer) internalNioBuffer (). clear (). position (index). limit (index + length));
上面是 ByteBuf 的其中一个具体的读实现, 可以看出 ByteBuf 维护着一个内部的 ByteBuffer, 叫做
internalNioBuffer。当需要将字节流写入网络的时候,

需要将 ByteBuf 转换为 ByteBuffer:

#### 简单聊聊：Netty 的线程模型的三种使用方式？

**单线程模型** ：

一个线程需要执行处理所有的 accept、read、decode、process、encode、send 事件。对于高负
载、高并发，并且对性能要求比较高的场景不适用。

对应到 Netty 代码是下面这样的

使用 NioEventLoopGroup 类的无参构造函数设置线程数量的默认值就是 **CPU 核心数 _2_ 。

```
ByteBuffer tmpBuf;
if (internal) {
tmpBuf = internalNioBuffer();
} else {
tmpBuf = ByteBuffer.wrap(array);
}
return out.write((ByteBuffer) tmpBuf.clear().position(index).limit(index +
length));
}
```
```
//1.eventGroup既用于处理客户端连接，又负责具体的处理。
EventLoopGroup eventGroup = new NioEventLoopGroup(1);
//2.创建服务端启动引导/辅助类：ServerBootstrap
ServerBootstrap b = new ServerBootstrap();
boobtstrap.group(eventGroup, eventGroup)
//......
```

**多线程模型**

一个 Acceptor 线程只负责监听客户端的连接，一个 NIO 线程池负责具体处理：accept、read、
decode、process、encode、send 事件。满足绝大部分应用场景，并发连接量不大的时候没啥问题，
但是遇到并发连接大的时候就可能会出现问题，成为性能瓶颈。

对应到 Netty 代码是下面这样的：

**主从多线程模型**

从一个主线程 NIO 线程池中选择一个线程作为 Acceptor 线程，绑定监听端口，接收客户端连接的连
接，其他线程负责后续的接入认证等工作。连接建立完成后，Sub NIO 线程池负责具体处理 I/O 读写。
如果多线程模型无法满足你的需求的时候，可以考虑使用主从多线程模型。

```
// 1.bossGroup 用于接收连接，workerGroup 用于具体的处理
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
try {
//2.创建服务端启动引导/辅助类：ServerBootstrap
ServerBootstrap b = new ServerBootstrap();
//3.给引导类配置两大线程组,确定了线程模型
b.group(bossGroup, workerGroup)
//......
```
```
// 1.bossGroup 用于接收连接，workerGroup 用于具体的处理
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();
try {
//2.创建服务端启动引导/辅助类：ServerBootstrap
ServerBootstrap b = new ServerBootstrap();
//3.给引导类配置两大线程组,确定了线程模型
b.group(bossGroup, workerGroup)
//......
```

#### 默认情况 Netty 起多少线程？何时启动？

Netty 默认是 CPU 处理器数的两倍，bind 完之后启动。

#### TCP 粘包/拆包的原因及解决方法？

###### TCP 粘包/分包的原因：

TCP 是以流的方式来处理数据，一个完整的包可能会被 TCP 拆分成多个包进行发送，也可能把小的封装
成一个大的数据包发送。

深层次的原因，是 TCP/IP 协议，是四层协议，会逐层进行数据的封装和分用。

而每一层都有自己的特定标识和头部，每一层的数据

数据通过互联网传输的时候不可能是光秃秃的不加标识，如果这样数据就会乱。所以数据在发送的时
候，需要加上特定标识，加上特定标识的过程叫做数据的封装，在数据使用的时候再去掉特定标识，去
掉特定标识的过程就叫做分用。


TCP/IP 协议的数据封装和分用过程，大致如下图所示：

图：TCP/IP 协议的数据封装和分用过程 TCP/IP 协议是层层

参考 TCP/IP 协议 （图解+秒懂+史上最全）

在 TCP/IP 分层中，OSI 的应用层、表示层、会话层全都统一为应用层，而底层的物理层和链路层又统一
为接口层。每个 TCP/IP 分层都对应一些协议，这些协议组成了 TCP/IP 协议栈。

最顶层的是应用层协议，比如 HTTP 协议、DNS 协议、FTP 协议、POP 3 协议等。

传输层包含两个协议，TCP 协议和 UDP 协议，在这一层将指定通信端口。

网络层包括 ARP 协议、IP 协议、ICMP 协议、IGMP 协议等等，网络层的协议有些并不完全属于网络层，
有可能向下跨到链路层，但总的来说，将它们都归类到网络层协议。

以太网： **每个以太网帧最小 64 字节，最大帧长为 1500 字节，** 以太网帧的格式如下图所示。

注意，上图中只有链路层部分才是以太网帧的数据，所以 7 字节的前同步码（Preamble）、 1 字节的帧
起始定界符、 12 字节的帧间距不属于以太网帧，它们属于物理层。其中，帧间距的存在，说明每个帧之
间是间隔传输的，并不是完全连续传输。

主要看以太网帧格式。首先是 6 字节的目标 MAC 地址和 6 字节的源 MAC 地址，然后是可选的 tag 标记，一
般当它不存在，之后是表示类型的 2 个字节，接着便是数据包部分，尾部是一个 4 字节的 CRC 的帧校验
码。

目标 MAC 字段记录了目标的 MAC 地址，当某网卡或适配器接收到了一个以太网帧，如果它的 MAC 地址
正好是帧中记录的目标地址，则将该帧解封后的数据包传递给网络层。此外，如果收到的是广播帧（目
标 MAC 地址为 FF-FF-FF-FF-FF-FF），则也将其传递给网络层，但如果收到目标 MAC 地址不是本接口地址
的任何数据帧，都将直接丢弃。

源 MAC 地址字段记录的是该帧的出口 MAC 地址。

由于信道是所有主机共享的，如果数据帧太长就会出现有的主机长时间不能发送数据，而且有的发送数
据可能超出接收端的缓冲区大小，造成缓冲溢出。为避免单一主机占用信道时间过长，规定了以太网帧
的最大帧长为 1500 。


由于信道是所有主机共享的，如果数据帧太长就会出现有的主机长时间不能发送数据，而且有的发送数
据可能超出接收端的缓冲区大小，造成缓冲溢出。为避免单一主机占用信道时间过长，规定了以太网帧
的最大帧长为 1500 。

**MTU**

在网络中，最大传输单元（MTU）指通过联网设备可以接收的最大数据包的值。

MTU 可以想象为高速公路地下通道或隧道的高度限制：超过高度限制的汽车和卡车无法通过，就像超
过网络 MTU 的数据包无法通过该网络一样。

不过，与汽车和卡车不同的是，超过 MTU 的数据包可被分解成较小的碎片，从而能通过网络。

这个过程称为分片。分片的数据包在到达目的地后便会重新组装。

MTU 以字节数为单位，一个“字节”等于 8 位信息，即 8 个一和零。1,500 字节是最大 MTU 大小。

**MSS**

MSS: Maximum Segment Size 最大报文段长度

MSS 是最大报文段长度的缩写，是 TCP 协议里面的一个概念。

MSS 是 TCP[数据包]每次能够传输的最大数据分段。

为了达到最佳的传输效能，TCP 协议在建立连接的时候通常要协商双方的 MSS 值，

这个值 TCP 协议在实现的时候往往用 MTU 值代替（需要减去 IP 数据包包头的大小 20 Bytes 和 TCP 数据段的
包头 20 Bytes）所以一般 MSS 值 1460

通讯双方会根据双方提供的 MSS 值得最小值确定为这次连接的最大 MSS 值。

```
注：最大报文段长度MSS这个名词很容易引起误解。
```
```
MSS是TCP报文段中的数据字段的最大长度。数据字段加上TCP首部才等于整个的TCP报文段。所
以MSS并不是TCP报文段的最大长度，而是：
```
MSS=TCP 报文段长度-TCP 首部长度-IP 首部。

MSS（Maximum Segment Size，最大报文长度），是 TCP 协议定义的一个选项，MSS 选项用于在 TCP
连接建立时，收发双方协商通信时每一个报文段所能承载的最大数据长度。

###### 应用层的半包问题（拆包和粘包）

拆包：一个完整的应用包可能会被 TCP 拆分成多个包进行发送。

粘包：也可能把小应用包的封装成一个大的 TCP 数据包发送。

应用程序写入的字节大小大于套接字发送缓冲区的大小，会发生拆包现象，

而应用程序写入数据小于套接字缓冲区大小，网卡将应用多次写入的数据发送到网络上，这将会发生粘
包现象。


当 TCP 帧的 payload（净荷）TCP 报文长度-TCP 头部长度>MSS (1460) 的时候，将发生拆包，进行 MSS
大小的 TCP 分段，

以太网帧的 payload（净荷）大于 MTU（ 1500 字节）进行太网帧分片。

###### 拆包和粘包解决方法

**解决方法**

Netty 中提供了多个 Decoder 解析类用于解决上述问题：

```
FixedLengthFrameDecoder 、LengthFieldBasedFrameDecoder ，固定长度是消息头指定消息
长度的一种形式，进行粘包拆包处理的。
LineBasedFrameDecoder 、DelimiterBasedFrameDecoder ，换行是于指定消息边界方式的一
种形式，进行消息粘
```
#### Netty 中有哪种重要组件？

```
Channel：Netty 网络操作抽象类，它除了包括基本的 I/O 操作，如 bind、connect、read、
write 等。
EventLoop：主要是配合 Channel 处理 I/O 操作，用来处理连接的生命周期中所发生的事情。
ChannelFuture：Netty 框架中所有的 I/O 操作都为异步的，因此我们需要 ChannelFuture 的
addListener()注册一个 ChannelFutureListener 监听事件，当操作执行成功或者失败时，监听就
会自动触发返回结果。
ChannelHandler：充当了所有处理入站和出站数据的逻辑容器。ChannelHandler 主要用来处理
各种事件，这里的事件很广泛，比如可以是连接、数据接收、异常、数据转换等。
ChannelPipeline：为 ChannelHandler 链提供了容器，当 channel 创建时，就会被自动分配到它
专属的 ChannelPipeline，这个关联是永久性的。
```
#### Netty 发送消息有几种方式？

Netty 有两种发送消息的方式：

```
直接写入 Channel 中，消息从 ChannelPipeline 当中尾部开始移动；
写入和 ChannelHandler 绑定的 ChannelHandlerContext 中，消息从 ChannelPipeline 中的下一
个 ChannelHandler 中移动。
```
#### 什么是 Netty 的零拷贝？

Netty 的零拷贝主要包含三个方面：

```
Netty 的接收和发送 ByteBuffer 采用 DIRECT BUFFERS，使用堆外直接内存进行 Socket 读写，不
需要进行字节缓冲区的二次拷贝。如果使用传统的堆内存（HEAP BUFFERS）进行 Socket 读写，
```
```
实际上，拆包、粘包的本质，就是受限于各层协议净菏的大小不一，
```
```
都要对上层的数据进行重新组装，
```
```
上层的协议收到下层的数据包，都有可能出现拆包、粘包问题。
```

```
JVM 会将堆内存 Buffer 拷贝一份到直接内存中，然后才写入 Socket 中。相比于堆外直接内存，
消息在发送过程中多了一次缓冲区的内存拷贝。
Netty 提供了组合 Buffer 对象，可以聚合多个 ByteBuffer 对象，用户可以像操作一个 Buffer 那
样方便的对组合 Buffer 进行操作，避免了传统通过内存拷贝的方式将几个小 Buffer 合并成一个大
的 Buffer。
Netty 的文件传输采用了 transferTo 方法，它可以直接将文件缓冲区的数据发送到目标
Channel，避免了传统通过循环 write 方式导致的内存拷贝问题。
```
这个答案不完整，具体请参考视频

#### 了解哪几种序列化协议？

序列化（编码）是将对象序列化为二进制形式（字节数组），主要用于网络传输、数据持久化等；而反
序列化（解码）则是将从网络、磁盘等读取的字节数组还原成原始对象，主要用于网络传输对象的解
码，以便完成远程调用。

影响序列化性能的关键因素：序列化后的码流大小（网络带宽的占用）、序列化的性能（CPU 资源占
用）；是否支持跨语言（异构系统的对接和开发语言切换）。

Java 默认提供的序列化：无法跨语言、序列化后的码流太大、序列化的性能差

XML，优点：人机可读性好，可指定元素或特性的名称。缺点：序列化数据只包含数据本身以及类的结
构，不包括类型标识和程序集信息；只能序列化公共属性和字段；不能序列化方法；文件庞大，文件格
式复杂，传输占带宽。适用场景：当做配置文件存储数据，实时数据转换。

JSON，是一种轻量级的数据交换格式，优点：兼容性高、数据格式比较简单，易于读写、序列化后数
据较小，可扩展性好，兼容性好、与 XML 相比，其协议比较简单，解析速度比较快。缺点：数据的描述
性比 XML 差、不适合性能要求为 ms 级别的情况、额外空间开销比较大。适用场景（可替代ＸＭＬ）：
跨防火墙访问、可调式性要求高、基于 Web browser 的 Ajax 请求、传输数据量相对小，实时性要求相对
低（例如秒级别）的服务。

Fastjson，采用一种“假定有序快速匹配”的算法。优点：接口简单易用、目前 java 语言中最快的 json
库。缺点：过于注重快，而偏离了“标准”及功能性、代码质量不高，文档不全。适用场景：协议交互、
Web 输出、Android 客户端

Thrift，不仅是序列化协议，还是一个 RPC 框架。优点：序列化后的体积小, 速度快、支持多种语言和丰
富的数据类型、对于数据字段的增删具有较强的兼容性、支持二进制压缩编码。缺点：使用者较少、跨
防火墙访问时，不安全、不具有可读性，调试代码时相对困难、不能与其他传输层协议共同使用（例如
HTTP）、无法支持向持久层直接读写数据，即不适合做数据持久化序列化协议。适用场景：分布式系
统的 RPC 解决方案

Avro，Hadoop 的一个子项目，解决了 JSON 的冗长和没有 IDL 的问题。优点：支持丰富的数据类型、简
单的动态语言结合功能、具有自我描述属性、提高了数据解析速度、快速可压缩的二进制数据形式、可
以实现远程过程调用 RPC、支持跨编程语言实现。缺点：对于习惯于静态类型语言的用户不直观。适用
场景：在 Hadoop 中做 Hive、Pig 和 MapReduce 的持久化数据格式。

Protobuf，将数据结构以. proto 文件进行描述，通过代码生成工具可以生成对应数据结构的 POJO 对象
和 Protobuf 相关的方法和属性。优点：序列化后码流小，性能高、结构化数据存储格式（XML JSON
等）、通过标识字段的顺序，可以实现协议的前向兼容、结构化的文档更容易管理和维护。缺点：需要
依赖于工具生成代码、支持的语言相对较少，官方只支持 Java 、C++ 、python。适用场景：对性能要


求高的 RPC 调用、具有良好的跨防火墙的访问属性、适合应用层对象的持久化

其它

protostuff 基于 protobuf 协议，但不需要配置 proto 文件，直接导包即可
Jboss marshaling 可以直接序列化 java 类，无须实 java. io. Serializable 接口
Message pack 一个高效的二进制序列化格式
Hessian 采用二进制协议的轻量级 remoting onhttp 工具
kryo 基于 protobuf 协议，只支持 java 语言, 需要注册（Registration），然后序列化（Output），反序
列化（Input）

#### 如何选择序列化协议？

具体场景

对于公司间的系统调用，如果性能要求在 100 ms 以上的服务，基于 XML 的 SOAP 协议是一个值得考虑的
方案。
基于 Web browser 的 Ajax，以及 Mobile app 与服务端之间的通讯，JSON 协议是首选。对于性能要求不
太高，或者以动态类型语言为主，或者传输数据载荷很小的的运用场景，JSON 也是非常不错的选择。
对于调试环境比较恶劣的场景，采用 JSON 或 XML 能够极大的提高调试效率，降低系统开发成本。
当对性能和简洁性有极高要求的场景，Protobuf，Thrift，Avro 之间具有一定的竞争关系。
对于 T 级别的数据的持久化应用场景，Protobuf 和 Avro 是首要选择。如果持久化后的数据存储在
hadoop 子项目里，Avro 会是更好的选择。

对于持久层非 Hadoop 项目，以静态类型语言为主的应用场景，Protobuf 会更符合静态类型语言工程师
的开发习惯。由于 Avro 的设计理念偏向于动态类型语言，对于动态语言为主的应用场景，Avro 是更好的
选择。
如果需要提供一个完整的 RPC 解决方案，Thrift 是一个好的选择。
如果序列化之后需要支持不同的传输层协议，或者需要跨防火墙访问的高性能场景，Protobuf 可以优先
考虑。
protobuf 的数据类型有多种：bool、double、float、int 32、int 64、string、bytes、enum、
message。protobuf 的限定符：required: 必须赋值，不能为空、optional: 字段可以赋值，也可以不赋
值、repeated: 该字段可以重复任意次数（包括 0 次）、枚举；只能用指定的常量集中的一个值作为其
值；

protobuf 的基本规则：每个消息中必须至少留有一个 required 类型的字段、包含 0 个或多个 optional 类
型的字段；repeated 表示的字段可以包含 0 个或多个数据；[1,15]之内的标识号在编码的时候会占用一
个字节（常用），[16,2047]之内的标识号则占用 2 个字节，标识号一定不能重复、使用消息类型，也可
以将消息嵌套任意多层，可用嵌套消息类型来代替组。

protobuf 的消息升级原则：不要更改任何已有的字段的数值标识；不能移除已经存在的 required 字段，
optional 和 repeated 类型的字段可以被移除，但要保留标号不能被重用。新添加的字段必须是 optional
或 repeated。因为旧版本程序无法读取或写入新增的 required 限定符的字段。

编译器为每一个消息类型生成了一个. java 文件，以及一个特殊的 Builder 类（该类是用来创建消息类接
口的）。如：UserProto. User. Builder builder = UserProto.User.newBuilder (); builder.build ()；

Netty 中的使用：ProtobufVarint 32 FrameDecoder 是用于处理半包消息的解码类；
ProtobufDecoder (UserProto.User.getDefaultInstance ()) 这是创建的 UserProto. java 文件中的解码
类；ProtobufVarint 32 LengthFieldPrepender 对 protobuf 协议的消息头上加上一个长度为 32 的整形字
段，用于标志这个消息的长度的类；ProtobufEncoder 是编码类

将 StringBuilder 转换为 ByteBuf 类型：copiedBuffer () 方法

#### Netty 支持哪些心跳类型设置？


```
readerIdleTime：为读超时时间（即测试端一定时间内未接受到被测试端消息）。
writerIdleTime：为写超时时间（即测试端一定时间内向被测试端发送消息）。
allIdleTime：所有类型的超时时间。
```
#### 为什么没有使用 Netty 5 ？

现在稳定推荐使用的主流版本还是 Netty 4，

Netty 5 中使用了 ForkJoinPool，增加了代码的复杂度，但是对性能的改善却不明显，

所以 Netty 5 版本不推荐使用，官网也没有提供下载链接。

Netty 入门门槛相对较高，其实是因为这方面的资料较少，并不是因为他有多难，大家其实都可以像搞
透 Spring 一样搞透 Netty。

在学习之前，建议先理解透整个框架原理结构，运行过程，可以少走很多弯路。

###### 简单说说：NIOEventLoopGroup 源码？

NioEventLoopGroup (其实是 MultithreadEventExecutorGroup) 内部维护一个类型为 EventExecutor
children [],

默认大小是处理器核数 * 2, 这样就构成了一个线程池，初始化 EventExecutor 时 NioEventLoopGroup 重
载 newChild 方法，所以 children 元素的实际类型为 NioEventLoop。

线程启动时调用 SingleThreadEventExecutor 的构造方法，执行 NioEventLoop 类的 run 方法，首先会调
用 hasTasks () 方法判断当前 taskQueue 是否有元素。

如果 taskQueue 中有元素，执行 selectNow () 方法，最终执行 selector.selectNow ()，该方法会立即返
回。如果 taskQueue 没有元素，执行 select (oldWakenUp) 方法

select ( oldWakenUp) 方法解决了 Nio 中的 bug，具体的方案为：

selectCnt 用来记录 selector. select 方法的执行次数和标识是否执行过 selector.selectNow ()，若触发了
epoll 的空轮询 bug，则会反复执行 selector.select (timeoutMillis)，变量 selectCnt 会逐渐变大，当
selectCnt 达到阈值（默认 512 ），则执行 rebuildSelector 方法，进行 selector 重建，解决 cpu 占用 100%
的 bug。

rebuildSelector 方法先通过 openSelector 方法创建一个新的 selector。然后将 old selector 的
selectionKey 执行 cancel。最后将 old selector 的 channel 重新注册到新的 selector 中。rebuild 后，需要
重新执行方法 selectNow，检查是否有已 ready 的 selectionKey。

接下来调用 processSelectedKeys 方法（处理 I/O 任务），当 selectedKeys != null 时，调用
processSelectedKeysOptimized 方法，迭代 selectedKeys 获取就绪的 IO 事件的 selectkey 存放在数组
selectedKeys 中, 然后为每个事件都调用 processSelectedKey 来处理它，processSelectedKey 中分别
处理 OP_READ；OP_WRITE；OP_CONNECT 事件。

最后调用 runAllTasks 方法（非 IO 任务），该方法首先会调用 fetchFromScheduledTaskQueue 方法，把
scheduledTaskQueue 中已经超过延迟执行时间的任务移到 taskQueue 中等待被执行，然后依次从
taskQueue 中取任务执行，每执行 64 个任务，进行耗时检查，如果已执行时间超过预先设定的执行时
间，则停止执行非 IO 任务，避免非 IO 任务太多，影响 IO 任务的执行。


每个 NioEventLoop 对应一个线程和一个 Selector，NioServerSocketChannel 会主动注册到某一个
NioEventLoop 的 Selector 上，NioEventLoop 负责事件轮询。

Outbound 事件都是请求事件, 发起者是 Channel，处理者是 unsafe，通过 Outbound 事件进行通
知，传播方向是 tail 到 head。Inbound 事件发起者是 unsafe，事件的处理者是 Channel, 是通知事件，
传播方向是从头到尾。

内存管理机制，首先会预申请一大块内存 Arena，Arena 由许多 Chunk 组成，而每个 Chunk 默认由 2048
个 page 组成。Chunk 通过 AVL 树的形式组织 Page，每个叶子节点表示一个 Page，而中间节点表示内存
区域，节点自己记录它在整个 Arena 中的偏移地址。当区域被分配出去后，中间节点上的标记位会被标
记，这样就表示这个中间节点以下的所有节点都已被分配了。大于 8 k 的内存分配在 poolChunkList 中，
而 PoolSubpage 用于分配小于 8 k 的内存，它会把一个 page 分割成多段，进行内存分配。

ByteBuf 的特点：支持自动扩容（4 M），保证 put 方法不会抛出异常、通过内置的复合缓冲类型，实现
零拷贝（zero-copy）；不需要调用 flip () 来切换读/写模式，读取和写入索引分开；方法链；引用计数基
于 AtomicIntegerFieldUpdater 用于内存回收；PooledByteBuf 采用二叉树来实现一个内存池，集中管
理内存的分配和释放，不用每次使用都新建一个缓冲区对象。UnpooledHeapByteBuf 每次都会新建一
个缓冲区对象。

#### 问题：JDK NIO 中 Epoll 空轮询 Bug 产生的原因以及 Netty^

#### 中是如何解决的

**产生原因**


正常情况下，selector.select () 操作是阻塞的，只有被监听的 fd 有读写操作时，才被唤醒。

但是，在这个 bug 中，没有任何 fd 有读写请求，但是 select () 操作依旧被唤醒很显然，这种情况下，
selectedKeys () 返回的是个空数组，然后按照逻辑执行到 while (true) 处，循环执行，导致死循环。

**Netty 的解决方法**

```
1. 对Selector的select操作周期进行统计，每完成一次空的select操作进行一次计数。
2. 若在某个周期内连续发生N次空轮询，则触发了epoll死循环bug。
3. 重建Selector，判断是否是其他线程发起的重建请求，若不是则将原SocketChannel从旧的
Selector上去除注册，重新注册到新的Selector上，并将原来的Selector关闭。
```
**尼恩暗语：**

以上的问题和答案，来自于网络（一搜一大把），并且点击率非常高。

以上问题，出现了知识性错误，不是答案有误，而是问题有误。


因为： 是 select 空轮训导致 cpu 100% 的问题，二 epoll 并没有出现空轮训导致 cpu 100% 的问题

所以，使用 epoll 反应器就可以了

#### 简单说说：Netty 如何解决 Selector 空轮询 BUG 的策略

这是早期，小伙伴在卷王群里，提到的高频面试题，直接贴链接哈

Netty 解决 Selector 空轮询 BUG 的策略（图解+秒懂+史上最全）

#### Netty 如何实现断线重连?

在实现 TCP 长连接功能中，客户端断线重连是一个很常见的问题，当我们使用 netty 实现断线重连时，是
否考虑过如下几个问题：

```
如何监听到客户端和服务端连接断开?
如何实现断线后重新连接?
netty客户端线程给多大比较合理?
```
**实现思路**

客户端在监测到与服务器端的连接断开后，或者一开始就无法连接的情况下，使用指定的重连策略进行
重连操作，直到重新建立连接或重试次数耗尽。

对于如何监测连接是否断开，则是通过重写 ChannelInboundHandler #channelInactive来实现 ，但连
接不可用，该方法会被触发，所以只需要在该方法做好重连工作即可。

**代码实现**

```
注：以下代码都是在上面心跳机制的基础上修改/添加的。
```
因为断线重连是客户端的工作，所以只需对客户端代码进行修改。

**重试策略**

**RetryPolicy —— 重试策略接口**

```
public interface RetryPolicy {
```
```
/**
* Called when an operation has failed for some reason. This method should
return
* true to make another attempt.
*
* @param retryCount the number of times retried so far (0 the first time)
* @return true/false
*/
boolean allowRetry(int retryCount);
```
```
/**
* get sleep time in ms of current retry count.
```

**ExponentialBackOffRetry —— 重连策略的默认实现**

```
*
* @param retryCount current retry count
* @return the time to sleep
*/
long getSleepTimeMs(int retryCount);
}
```
```
/**
* <p>Retry policy that retries a set number of times with increasing sleep time
between retries</p>
*/
public class ExponentialBackOffRetry implements RetryPolicy {
```
```
private static final int MAX_RETRIES_LIMIT = 29 ;
private static final int DEFAULT_MAX_SLEEP_MS = Integer.MAX_VALUE;
```
```
private final Random random = new Random();
private final long baseSleepTimeMs;
private final int maxRetries;
private final int maxSleepMs;
```
```
public ExponentialBackOffRetry(int baseSleepTimeMs, int maxRetries) {
this(baseSleepTimeMs, maxRetries, DEFAULT_MAX_SLEEP_MS);
}
```
```
public ExponentialBackOffRetry(int baseSleepTimeMs, int maxRetries, int
maxSleepMs) {
this.maxRetries = maxRetries;
this.baseSleepTimeMs = baseSleepTimeMs;
this.maxSleepMs = maxSleepMs;
}
```
```
@Override
public boolean allowRetry(int retryCount) {
if (retryCount < maxRetries) {
return true;
}
return false;
}
```
```
@Override
public long getSleepTimeMs(int retryCount) {
if (retryCount < 0 ) {
throw new IllegalArgumentException("retries count must greater than
0.");
}
if (retryCount > MAX_RETRIES_LIMIT) {
System.out.println(String.format("maxRetries too large (%d). Pinning
to %d", maxRetries, MAX_RETRIES_LIMIT));
retryCount = MAX_RETRIES_LIMIT;
}
long sleepMs = baseSleepTimeMs * Math.max( 1 , random.nextInt( 1 <<
retryCount));
if (sleepMs > maxSleepMs) {
```

**ReconnectHandler—— 重连处理器**

```
System.out.println(String.format("Sleep extension too large (%d).
Pinning to %d", sleepMs, maxSleepMs));
sleepMs = maxSleepMs;
}
return sleepMs;
}
}
```
```
@ChannelHandler.Sharable
public class ReconnectHandler extends ChannelInboundHandlerAdapter {
```
```
private int retries = 0 ;
private RetryPolicy retryPolicy;
```
```
private TcpClient tcpClient;
```
```
public ReconnectHandler(TcpClient tcpClient) {
this.tcpClient = tcpClient;
}
```
```
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
System.out.println("Successfully established a connection to the
server.");
retries = 0 ;
ctx.fireChannelActive();
}
```
```
@Override
public void channelInactive(ChannelHandlerContext ctx) throws Exception {
if (retries == 0 ) {
System.err.println("Lost the TCP connection with the server.");
ctx.close();
}
```
```
boolean allowRetry = getRetryPolicy().allowRetry(retries);
if (allowRetry) {
```
```
long sleepTimeMs = getRetryPolicy().getSleepTimeMs(retries);
```
```
System.out.println(String.format("Try to reconnect to the server
after %dms. Retry count: %d.", sleepTimeMs, ++retries));
```
```
final EventLoop eventLoop = ctx.channel().eventLoop();
eventLoop.schedule(() -> {
System.out.println("Reconnecting ...");
tcpClient.connect();
}, sleepTimeMs, TimeUnit.MILLISECONDS);
}
ctx.fireChannelInactive();
}
```
```
private RetryPolicy getRetryPolicy() {
if (this.retryPolicy == null) {
this.retryPolicy = tcpClient.getRetryPolicy();
```

**ClientHandlersInitializer**

在之前的基础上，添加了重连处理器 ReconnectHandler。

**TcpClient**

在之前的基础上添加重连、重连策略的支持。

```
}
return this.retryPolicy;
}
}
```
```
public class ClientHandlersInitializer extends ChannelInitializer<SocketChannel>
{
```
```
private ReconnectHandler reconnectHandler;
private EchoHandler echoHandler;
```
```
public ClientHandlersInitializer(TcpClient tcpClient) {
Assert.notNull(tcpClient, "TcpClient can not be null.");
this.reconnectHandler = new ReconnectHandler(tcpClient);
this.echoHandler = new EchoHandler();
}
```
```
@Override
protected void initChannel(SocketChannel ch) throws Exception {
ChannelPipeline pipeline = ch.pipeline();
pipeline.addLast(this.reconnectHandler);
pipeline.addLast(new LengthFieldBasedFrameDecoder(Integer.MAX_VALUE, 0,
4, 0, 4));
pipeline.addLast(new LengthFieldPrepender(4));
pipeline.addLast(new StringDecoder(CharsetUtil.UTF_8));
pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8));
pipeline.addLast(new Pinger());
}
}
```
```
public class TcpClient {
```
```
private String host;
private int port;
private Bootstrap bootstrap;
/** 重连策略 */
private RetryPolicy retryPolicy;
/** 将<code>Channel</code>保存起来, 可用于在其他非handler的地方发送数据 */
private Channel channel;
```
```
public TcpClient(String host, int port) {
this(host, port, new ExponentialBackOffRetry(1000, Integer.MAX_VALUE, 60
* 1000));
}
```
```
public TcpClient(String host, int port, RetryPolicy retryPolicy) {
this.host = host;
this.port = port;
this.retryPolicy = retryPolicy;
```

在测试之前，为了避开 Connection reset by peer 异常，可以稍微修改 Pinger 的 ping () 方法，添加 if
(second == 5) 的条件判断。如下：

```
init();
}
```
```
/**
* 向远程TCP服务器请求连接
*/
public void connect() {
synchronized (bootstrap) {
ChannelFuture future = bootstrap.connect(host, port);
future.addListener(getConnectionListener());
this.channel = future.channel();
}
}
```
```
public RetryPolicy getRetryPolicy() {
return retryPolicy;
}
```
```
private void init() {
EventLoopGroup group = new NioEventLoopGroup();
// bootstrap 可重用, 只需在TcpClient实例化的时候初始化即可.
bootstrap = new Bootstrap();
bootstrap.group(group)
.channel(NioSocketChannel.class)
.handler(new ClientHandlersInitializer(TcpClient.this));
}
```
```
private ChannelFutureListener getConnectionListener() {
return new ChannelFutureListener() {
@Override
public void operationComplete(ChannelFuture future) throws Exception
{
if (!future.isSuccess()) {
future.channel().pipeline().fireChannelInactive();
}
}
};
}
```
```
public static void main(String[] args) {
TcpClient tcpClient = new TcpClient("localhost", 2222);
tcpClient.connect();
}
```
```
}
```
```
private void ping(Channel channel) {
int second = Math.max(1, random.nextInt(baseRandom));
if (second == 5) {
second = 6;
}
System.out.println("next heart beat will send after " + second + "s.");
```

**启动客户端**

先只启动客户端，观察控制台输出，可以看到类似如下日志：

```
ScheduledFuture<?> future = channel.eventLoop().schedule(new Runnable()
{
@Override
public void run() {
if (channel.isActive()) {
System.out.println("sending heart beat to the server...");
channel.writeAndFlush(ClientIdleStateTrigger.HEART_BEAT);
} else {
System.err.println("The connection had broken, cancel the
task that will send a heart beat.");
channel.closeFuture();
throw new RuntimeException();
}
}
}, second, TimeUnit.SECONDS);
```
```
future.addListener(new GenericFutureListener() {
@Override
public void operationComplete(Future future) throws Exception {
if (future.isSuccess()) {
ping(channel);
}
}
});
}
```

断线重连测试——客户端控制台输出

可以看到，当客户端发现无法连接到服务器端，所以一直尝试重连。随着重试次数增加，重试时间间隔
越大，但又不想无限增大下去，所以需要定一个阈值，比如 60 s。如上图所示，当下一次重试时间超过
60 s 时，会打印 Sleep extension too large (*). Pinning to 60000，单位为 ms。出现这句话的意思是，计
算出来的时间超过阈值（60 s），所以把真正睡眠的时间重置为阈值（60 s）。

#### 聊聊：Netty 如何实现心跳机制?

**何为心跳**

所谓心跳, 即在 TCP 长连接中, 客户端和服务器之间定期发送的一种特殊的数据包, 通知对方自己还在线,
以确保 TCP 连接的有效性.

```
注：心跳包还有另一个作用，经常被忽略，即： 一个连接如果长时间不用，防火墙或者路由器就
会断开该连接。
```
**如何实现**

**核心 Handler —— IdleStateHandler**

在 Netty 中, 实现心跳机制的关键是 IdleStateHandler, 那么这个 Handler 如何使用呢? 先看下它的构造
器：


这里解释下三个参数的含义：

```
readerIdleTimeSeconds: 读超时. 即当在指定的时间间隔内没有从 Channel 读取到数据时, 会触发
一个 READER_IDLE 的 IdleStateEvent 事件.
writerIdleTimeSeconds: 写超时. 即当在指定的时间间隔内没有数据写入到 Channel 时, 会触发一
个 WRITER_IDLE 的 IdleStateEvent 事件.
allIdleTimeSeconds: 读/写超时. 即当在指定的时间间隔内没有读或写操作时, 会触发一个
ALL_IDLE 的 IdleStateEvent 事件.
```
```
注：这三个参数默认的时间单位是秒。若需要指定其他时间单位，可以使用另一个构造方法：
IdleStateHandler(boolean observeOutput, long readerIdleTime, long writerIdleTime, long
allIdleTime, TimeUnit unit)
```
在看下面的实现之前，建议先了解一下 IdleStateHandler 的实现原理。

下面直接上代码，需要注意的地方，会在代码中通过注释进行说明。

**使用 IdleStateHandler 实现心跳**

下面将使用 IdleStateHandler 来实现心跳，Client 端连接到 Server 端后，会循环执行一个任务： **随机等
待几秒** ， **然后** ping **一下** Server **端** ， **即发送一个心跳包** 。当等待的时间超过规定时间，将会发送失败，以
为 Server 端在此之前已经主动断开连接了。代码如下：

**Client 端**

**ClientIdleStateTrigger —— 心跳触发器**

类 ClientIdleStateTrigger 也是一个 Handler，只是重写了 userEventTriggered 方法，用于捕获
IdleState. WRITER_IDLE 事件（未在指定时间内向服务器发送数据），然后向 Server 端发送一个心跳
包。

```
public IdleStateHandler(int readerIdleTimeSeconds, int writerIdleTimeSeconds,
int allIdleTimeSeconds) {
this((long)readerIdleTimeSeconds, (long)writerIdleTimeSeconds,
(long)allIdleTimeSeconds, TimeUnit.SECONDS);
}
```
```
/**
* <p>
* 用于捕获{@link IdleState#WRITER_IDLE}事件（未在指定时间内向服务器发送数据），然后向
<code>Server</code>端发送一个心跳包。
* </p>
*/
public class ClientIdleStateTrigger extends ChannelInboundHandlerAdapter {
```
```
public static final String HEART_BEAT = "heart beat!";
```
```
@Override
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws
Exception {
if (evt instanceof IdleStateEvent) {
IdleState state = ((IdleStateEvent) evt).state();
if (state == IdleState.WRITER_IDLE) {
// write heartbeat to server
ctx.writeAndFlush(HEART_BEAT);
}
} else {
super.userEventTriggered(ctx, evt);
```

**Pinger —— 心跳发射器**

```
}
}
```
```
}
```
```
/**
* <p>客户端连接到服务器端后，会循环执行一个任务：随机等待几秒，然后ping一下Server端，即发送
一个心跳包。</p>
*/
public class Pinger extends ChannelInboundHandlerAdapter {
```
```
private Random random = new Random();
private int baseRandom = 8 ;
```
```
private Channel channel;
```
```
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
super.channelActive(ctx);
this.channel = ctx.channel();
```
```
ping(ctx.channel());
}
```
```
private void ping(Channel channel) {
int second = Math.max( 1 , random.nextInt(baseRandom));
System.out.println("next heart beat will send after " + second + "s.");
ScheduledFuture<?> future = channel.eventLoop().schedule(new Runnable()
{
@Override
public void run() {
if (channel.isActive()) {
System.out.println("sending heart beat to the server...");
channel.writeAndFlush(ClientIdleStateTrigger.HEART_BEAT);
} else {
System.err.println("The connection had broken, cancel the
task that will send a heart beat.");
channel.closeFuture();
throw new RuntimeException();
}
}
}, second, TimeUnit.SECONDS);
```
```
future.addListener(new GenericFutureListener() {
@Override
public void operationComplete(Future future) throws Exception {
if (future.isSuccess()) {
ping(channel);
}
}
});
}
```
```
@Override
```

**ClientHandlersInitializer —— 客户端处理器集合的初始化类**

```
注： 上面的Handler集合，除了Pinger，其他都是编解码器和解决粘包，可以忽略。
```
**TcpClient —— TCP 连接的客户端**

```
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
throws Exception {
// 当Channel已经断开的情况下, 仍然发送数据, 会抛异常, 该方法会被调用.
cause.printStackTrace();
ctx.close();
}
}
```
```
public class ClientHandlersInitializer extends ChannelInitializer<SocketChannel>
{
```
```
private ReconnectHandler reconnectHandler;
private EchoHandler echoHandler;
```
```
public ClientHandlersInitializer(TcpClient tcpClient) {
Assert.notNull(tcpClient, "TcpClient can not be null.");
this.reconnectHandler = new ReconnectHandler(tcpClient);
this.echoHandler = new EchoHandler();
}
```
```
@Override
protected void initChannel(SocketChannel ch) throws Exception {
ChannelPipeline pipeline = ch.pipeline();
pipeline.addLast(new LengthFieldBasedFrameDecoder(Integer.MAX_VALUE, 0 ,
4 , 0 , 4 ));
pipeline.addLast(new LengthFieldPrepender( 4 ));
pipeline.addLast(new StringDecoder(CharsetUtil.UTF_8));
pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8));
pipeline.addLast(new Pinger());
}
}
```
```
public class TcpClient {
```
```
private String host;
private int port;
private Bootstrap bootstrap;
/** 将<code>Channel</code>保存起来, 可用于在其他非handler的地方发送数据 */
private Channel channel;
```
```
public TcpClient(String host, int port) {
this(host, port, new ExponentialBackOffRetry( 1000 , Integer.MAX_VALUE, 60
* 1000 ));
}
```
```
public TcpClient(String host, int port, RetryPolicy retryPolicy) {
this.host = host;
this.port = port;
init();
}
```

**Server 端**

**ServerIdleStateTrigger —— 断连触发器**

**ServerBizHandler —— 服务器端的业务处理器**

```
/**
* 向远程TCP服务器请求连接
*/
public void connect() {
synchronized (bootstrap) {
ChannelFuture future = bootstrap.connect(host, port);
this.channel = future.channel();
}
}
```
```
private void init() {
EventLoopGroup group = new NioEventLoopGroup();
// bootstrap 可重用, 只需在TcpClient实例化的时候初始化即可.
bootstrap = new Bootstrap();
bootstrap.group(group)
.channel(NioSocketChannel.class)
.handler(new ClientHandlersInitializer(TcpClient.this));
}
```
```
public static void main(String[] args) {
TcpClient tcpClient = new TcpClient("localhost", 2222 );
tcpClient.connect();
}
```
```
}
```
```
/**
* <p>在规定时间内未收到客户端的任何数据包, 将主动断开该连接</p>
*/
public class ServerIdleStateTrigger extends ChannelInboundHandlerAdapter {
@Override
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws
Exception {
if (evt instanceof IdleStateEvent) {
IdleState state = ((IdleStateEvent) evt).state();
if (state == IdleState.READER_IDLE) {
// 在规定时间内没有收到客户端的上行数据, 主动断开连接
ctx.disconnect();
}
} else {
super.userEventTriggered(ctx, evt);
}
}
}
```
```
/**
* <p>收到来自客户端的数据包后, 直接在控制台打印出来.</p>
*/
@ChannelHandler.Sharable
public class ServerBizHandler extends SimpleChannelInboundHandler<String> {
```

**ServerHandlerInitializer —— 服务器端处理器集合的初始化类**

```
private final String REC_HEART_BEAT = "I had received the heart beat!";
```
```
@Override
protected void channelRead0(ChannelHandlerContext ctx, String data) throws
Exception {
try {
System.out.println("receive data: " + data);
// ctx.writeAndFlush(REC_HEART_BEAT);
} catch (Exception e) {
e.printStackTrace();
}
}
```
```
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
System.out.println("Established connection with the remote client.");
```
```
// do something
```
```
ctx.fireChannelActive();
}
```
```
@Override
public void channelInactive(ChannelHandlerContext ctx) throws Exception {
System.out.println("Disconnected with the remote client.");
```
```
// do something
```
```
ctx.fireChannelInactive();
}
```
```
@Override
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
throws Exception {
cause.printStackTrace();
ctx.close();
}
}
```
```
/**
* <p>用于初始化服务器端涉及到的所有<code>Handler</code></p>
*/
public class ServerHandlerInitializer extends ChannelInitializer<SocketChannel>
{
```
```
protected void initChannel(SocketChannel ch) throws Exception {
ch.pipeline().addLast("idleStateHandler", new IdleStateHandler( 5 , 0 ,
0 ));
ch.pipeline().addLast("idleStateTrigger", new ServerIdleStateTrigger());
ch.pipeline().addLast("frameDecoder", new
LengthFieldBasedFrameDecoder(Integer.MAX_VALUE, 0 , 4 , 0 , 4 ));
ch.pipeline().addLast("frameEncoder", new LengthFieldPrepender( 4 ));
ch.pipeline().addLast("decoder", new StringDecoder());
ch.pipeline().addLast("encoder", new StringEncoder());
ch.pipeline().addLast("bizHandler", new ServerBizHandler());
```

```
注：new IdleStateHandler(5, 0, 0)该handler代表如果在 5 秒内没有收到来自客户端的任何数据包
（包括但不限于心跳包），将会主动断开与该客户端的连接。
```
**TcpServer —— 服务器端**

至此，所有代码已经编写完毕。

**测试**

首先启动客户端，再启动服务器端。启动完成后，在客户端的控制台上，可以看到打印如下类似日志：

```
}
```
```
}
```
```
public class TcpServer {
private int port;
private ServerHandlerInitializer serverHandlerInitializer;
```
```
public TcpServer(int port) {
this.port = port;
this.serverHandlerInitializer = new ServerHandlerInitializer();
}
```
```
public void start() {
EventLoopGroup bossGroup = new NioEventLoopGroup( 1 );
EventLoopGroup workerGroup = new NioEventLoopGroup();
try {
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(bossGroup, workerGroup)
.channel(NioServerSocketChannel.class)
.childHandler(this.serverHandlerInitializer);
// 绑定端口，开始接收进来的连接
ChannelFuture future = bootstrap.bind(port).sync();
```
```
System.out.println("Server start listen at " + port);
future.channel().closeFuture().sync();
} catch (Exception e) {
bossGroup.shutdownGracefully();
workerGroup.shutdownGracefully();
e.printStackTrace();
}
}
```
```
public static void main(String[] args) throws Exception {
int port = 2222 ;
new TcpServer(port).start();
}
}
```

客户端控制台输出的日志

在服务器端可以看到控制台输出了类似如下的日志：

服务器端控制台输出的日志

#### 默认情况 Netty 起多少线程？何时启动？

如果你在简历上写了 Netty，那么面试官百分之九十的可能会问你 Netty 默认其多少线程？在什么时候
启动的问题。

面试官一方面是想考验你对 Netty 有没有最基本的知识点掌握，一方面是想试探你有没有深入了解过
Netty 的源码和启动流程。

你在编写 Netty 服务端的时候经常会编写下面的代码：

这里构建了两个事件循环组，循环组具体是干什么的我就暂时不细说了。我们拿其中的一个来进行
分析底层的源码即可。这里调用了无参的构造方法，也就使用了默认的线程数，那么默认线程数是多少
呢？慢慢进行分析：

调用了“this (0)”，也就是调用了单个参数的构造方法，然后调用的传递线程数是 0 。

```
EventLoopGroup boss = new NioEventLoopGroup();
EventLoopGroup worker = new NioEventLoopGroup();
```
```
public NioEventLoopGroup() {
this( 0 );
}
```

紧接着调用线程数和线程池接口的方法，需要注意的是，这里的“executor”是 null。

因为 Netty 底层封装了 NIO 所以调用了传递“provider”的构造参数。

紧接着比之前的多了一个默认选择策略工厂，传递了一个单例对象。

之后调用了父类的构造方法，这里又比之前多了一个参数是异常拒绝处理器，也就是出现异常的时
候执行的拒绝处理器。

紧接着又调用了父类的“MultithreadEventLoopGroup”构造方法，下面这个就是默认初始化的线程
了：

这里是一个三元表达式，判断传递过来的参数“nThreads”是否等于 0 ，如果是的话就赋值
“DEFAULT_EVENT_LOOP_THREADS”，如果不是的话就赋值为传递过来的值。

那么“DEFAULT_EVENT_LOOP_THREADS”是在什么时候初始化的呢？是在静态代码块中初始化。

```
public NioEventLoopGroup(int nThreads) {
this(nThreads, (Executor) null);
}
```
```
public NioEventLoopGroup(int nThreads, Executor executor) {
this(nThreads, executor, SelectorProvider.provider());
}
```
```
public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider
selectorProvider) {
this(nThreads, executor, selectorProvider,
DefaultSelectStrategyFactory.INSTANCE);
}
```
```
public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider
selectorProvider, final SelectStrategyFactory selectStrategyFactory) {
super(nThreads, executor, selectorProvider, selectStrategyFactory,
RejectedExecutionHandlers.reject());
}
```
```
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object...
args) {
super(nThreads == 0? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor,
args);
}
```
```
nThreads == 0? DEFAULT_EVENT_LOOP_THREADS : nThreads
```

它获取的值也就是“NettyRuntime.availableProcessors () * 2”。

然后调用了“availableProcessors”的方法：

需要注意的就是“Runtime.getRuntime (). availableProcessors ()”这个方法，这个方法的意思是获取
运行时可用线程数。接下来下来写个方法测试一下：

由此可见，Netty 的默认启动了电脑可用线程数的两倍，在调用了 bind 方法的时候执行。

那么反过来思考下！

我们真的需要每个 EventLoopGroup 都默认启用线程吗？

答案当然不是！

我们具体的线程数应该交由业务来决定而不是使用默认的参数。

有的时候我们只需要使用一个线程来监听注册事件。

```
private static final int DEFAULT_EVENT_LOOP_THREADS;
static {
DEFAULT_EVENT_LOOP_THREADS = Math.max( 1 , SystemPropertyUtil.getInt(
"io.netty.eventLoopThreads", NettyRuntime.availableProcessors() *
2 ));
```
```
if (logger.isDebugEnabled()) {
logger.debug("-Dio.netty.eventLoopThreads: {}",
DEFAULT_EVENT_LOOP_THREADS);
}
}
```
```
public static int availableProcessors()
{
return holder.availableProcessors();
}
```
```
synchronized int availableProcessors() {
if (this.availableProcessors == 0 ) {
final int availableProcessors =SystemPropertyUtil.getInt(
"io.netty.availableProcessors",
Runtime.getRuntime().availableProcessors());
setAvailableProcessors(availableProcessors);
}
return this.availableProcessors;
}
```
```
public class NettyServer {
public static void main(String[] args) {
System.out.println(Runtime.getRuntime().availableProcessors());
}
}
```
```
“Runtime.getRuntime().availableProcessors()”
```

比如，在客户端代码中，线程数的调整，非常有必要。

#### 聊聊：NioEventLoopGroup 默认的构造函数会起多少线

#### 程？

同上

#### 聊聊：如何设计一个内存池，或者内存分配器

```
这是出现的次数比较多的大厂面试题：怎样设计一个内存池和内存分配器？
```
```
尼恩暗语：会的人不多，如果能答上来，面试官会两眼发绿光
```
**重要性：对内存分配器透彻理解，并且具备实操能力，是编程高手的标志之一** 。

如果你不能理解 malloc 之类内存分配器实现原理的话，那你可能就写不出高性能程序，写不出高性能程
序就很难参与核心项目，参与不了核心项目那么很难升职加薪，很难升级加薪就无法走向人生巅峰，所
以，内存分配非常关键

说明：此题是一个实操性质的题目，后续尼恩带大家参考 netty 内存池，从 0 到 1 ，架构、设计、实现一
个高性能内存池组件

关于对象池的知识，请参见 Netty 内存池（史上最全 + 5 W 字长文）

#### 聊聊：如何设计一个 Java 对象池，减少 GC 和内存分配消耗

**重要性：对对象池透彻理解，并且具备实操能力，也是编程高手的标志之一** 。

对象池顾名思义就是存放对象的池，与我们常听到的线程池、数据库连接池、http 连接池等一样，都是
典型的池化设计思想。

对象池的优点就是可以集中管理池中对象，减少频繁创建和销毁长期使用的对象，从而提升复用性，以
节约资源的消耗，可以有效避免频繁为对象分配内存和释放堆中内存，进而减轻 jvm 垃圾收集器的负
担，避免内存抖动。

Apache Common Pool 2 是 Apache 提供的一个通用对象池技术实现，可以方便定制化自己需要的对象
池，大名鼎鼎的 Redis 客户端 Jedis 内部连接池就是基于它来实现的。

说明：此题是一个实操性质的题目，后续尼恩带大家参考 netty 对象池，从 0 到 1 ，架构、设计、实现一
个高性能对象池组件


关于对象池的知识，请参见 Netty 内存池（史上最全 + 5 W 字长文）

## 美团二面：epoll 性能那么高，为什么？

#### 说在前面

在 40 岁老架构师尼恩的 **读者交流群** (50+) 中，最近有小伙伴拿到了一线互联网企业如美团、拼多多、极
兔、有赞、希音的面试资格，遇到一几个很重要的面试题：

```
说说epoll的数据结构
说说epoll的实现原理
协议栈如何与epoll通信？
epoll线程安全如何加锁？
说说ET与LT的实现
......
```
这里尼恩给大家做一下系统化、体系化的梳理，使得大家可以充分展示一下大家雄厚的 “技术肌肉”， **让
面试官爱到 “不能自已、口水直流”** 。

也一并把这个题目以及参考答案，收入咱们的《尼恩 Java 面试宝典》V 88 版本，供后面的小伙伴参考，
提升大家的 3 高架构、设计、开发水平。

```
最新《尼恩 架构笔记》《尼恩高并发三部曲 》《尼恩Java面试宝典》 的PDF文件，请关注公众号
【技术自由圈】领取，暗号：领电子书
```
#### epoll 的数据结构

###### 多种数据结构进行决策

epoll 至少需要两个集合

```
所有fd的总集
就绪fd的集合
```
那么这个总集选用什么数据结构存储呢？

我们知道，一个 fd，其底层对应一个 TCB。那么也就是说 key=fd, val=TCB, 是一个典型的 kv 型数据结构，
对于 kv 型数据结构我们可以使用以下三种进行存储。

```
1. hash
2. 红黑树
3. b/b+tree
```

如果使用 hash 进行存储，其优点是查询速度很快，O (1)。

但是在我们调用 epoll_create () 的时候，hash 底层的数组创建多大合适呢？

如果我们有百万的 fd，那么这个数组越大越好，如果我们仅仅十几个 fd 需要管理，在创建数组的时候，
太大的空间就很浪费。而这个 fd 我们又不能预先知道有多少，所以 hash 是不合适的。

b/b+tree 是多叉树，一个结点可以存多个 key，主要是用于降低层高，用于磁盘索引的，所以在我们这
个内存场景下也是不适合的。

在内存索引的场景下我们一般使用红黑树来作为首选的数据结构，首先红黑树的查找速度很
快,O (log (N))。其次在调用 epoll_create () 的时候，只需要创建一个红黑树树根即可，无需浪费额外的空
间。

那么就绪集合用什么数据结构呢，首先就绪集合不是以查找为主的，就绪集合的作用是将里面的元素拷
贝给用户进行处理，所以集合里的元素没有优先级，那么就可以采用线性的数据结构，使用队列来存
储，先进先出，先就绪的先处理。

所有 fd 的总集 -----> 红黑树

就绪 fd 的集合 -----> 队列

#### 红黑树和就绪队列的关系

红黑树的结点和就绪队列的结点的同一个节点，所谓的加入就绪队列，就是将结点的前后指针联系到一
起。所以就绪了不是将红黑树结点 delete 掉然后加入队列。他们是同一个结点，不需要 delete。

```
struct epitem{
RB_ENTRY(epitem) rbn;
LIST_ENTRY(epitem) rdlink;
int rdy; //exist in list
```
```
int sockfd;
struct epoll_event event;
};
struct eventpoll {
ep_rb_tree rbr;
int rbcnt;
LIST_HEAD( ,epitem) rdlist;
int rdnum;
```
```
int waiting;
pthread_mutex_t mtx; //rbtree update
pthread_spinlock_t lock; //rdlist update
pthread_cond_t cond; //block for event
pthread_mutex_t cdmtx; //mutex for cond
};
```

协议栈如何与 epoll 模块通信

#### epoll 的工作环境

应用程序只能通过三个 api 接口来操作 epoll。当一个 io 准备就绪的时候，epoll 是怎么知道 io 准备就绪了
呢？是由协议栈将数据解析出来触发回调通知 epoll 的。也就是说可以把 epoll 的工作环境看出三部分，
左边应用程序的 api，中间的 epoll，右边是协议栈的回调 (协议栈当然不能直接操作 epoll，中间的 vfs 在
此不是重点，就直接省略 vfs 这一层了)。

#### 协议栈触发回调通知 epoll 的时机


socket 有两类，一类是监听 listenfd，一类是客户端 clientfd。对于 sockfd 而言，我们一般比较关注
EPOLLIN 和 EPOLLOUT 这两个事件，所以如果是 listenfd，我们通常的做法就是 accept。对于 clientfd 来
说，如果可读我们就 recv，如果可写我们就 send。

协议栈将数据解析出来触发回调通知 epoll。epoll 是怎么知道哪个 io 就绪了呢？我们从 ip 头可以解析出源
ip，目的 ip 和协议，从 tcp 头可以解析出源端口和目的端口，此时五元组就凑齐了。socket fd --- < 源 IP
地址 , 源端口 , 目的 IP 地址 , 目的端口 , 协议 > 一个 fd 就是一个五元组，知道了 fd，我们就能从红黑树中
找到对应的结点。

那么这个回调函数做什么事情呢？我们传入 fd 和具体事件这两个参数，然后做下面两个操作

```
1. 通过fd找到对应的结点
2. 把结点加入到就绪队列
```
1 、协议栈中，在三次握手完成之后，会往全连接队列中添加一个 TCB 结点，然后触发一个回调函数，
通知到 epoll 里面有个 EPOLLIN 事件


2 、客户端发送一个数据包，协议栈接收后回复 ACK，之后触发一个回调函数，通知到 epoll 里面有个
EPOLLIN 事件

3 、每个连接的 TCB 里面都有一个 sendbuf，在对端接收到数据并返回 ACK 以后，sendbuf 就可以将这部
分确认接收的数据清空，此时 sendbuf 里面就有剩余空间，此时触发一个回调函数，通知到 epoll 里面有
个 EPOLLOUT 事件


4 、当对端发送 close，在接收到 fin 后回复 ACK，此时会调用回调函数，通知到 epoll 有个 EPOLLIN 事件


5 、当接收到 rst 标志位的时候，回复 ack 之后也会触发回调函数，通知 epoll 有一个 EPOLLERR 事件


###### 通知的时机总结

一个有 5 个通知的地方

```
1. 三次握手完成之后，
2. 接收数据回复ACK之后，
3. 发送数据收到ACK之后，
4. 接收FIN回复ACK之后，
5. 接收RST回复ACK之后
```

#### 从回调机制看 epoll 与 select/poll 的区别

由于 select 和 poll 没有本质的区别，所以下面统一称为 poll。

我们看到每次调用 poll，都需要把总集 fds 拷贝到内核态，检测完之后，再有内核态拷贝的用户态，这就
是 poll。而 epoll 不是这样，epoll 只要有新的 io 就调用 epoll_ctl () 加入到红黑树里面，一旦有触发就用
epoll_wait () 将有事件的结点带出来，可以看到他们的第一个区别：poll 总是拷贝总集，如果有 100 w 个
fd，只有两三个就绪呢？这会造成大量资源浪费；而 epoll 总是将需要拷贝的东西进行拷贝，没有浪费。

第二个区别：我们从上面知道了 epoll 的事件都是由协议栈进行回调然后加入到就绪队列的，而 poll 呢？
内核如何检测 poll 的 io 是否就绪？只能通过遍历的方法判断，所以 poll 检测 io 通过遍历的方法也是比较慢
的。

所以两者的区别：

select/poll 需要把总集 copy 到内核，而 epoll 不用

实现原理上面，select/poll 需要循环遍历总集是否有就绪，而 epoll 是那个结点就绪了就加入就绪队列
里面。

注意：poll 不一定就比 epoll 慢，在 io 量小的情况下，poll 是比 epoll 快的，而在大 io 量下，epoll 绝对是有
主导地位的。至于有多少个 io 才算多，其实也很难说，一般认为 500 或者 1024 为分界点（我乱说的） 。

#### epoll 线程安全如何加锁

###### 3 个 api 做什么事情

epoll_create () ===》创建红黑树的根节点

epoll_ctl () ===》add, del, mod 增加、删除、修改结点

epoll_wait () ===》把就绪队列的结点 copy 到用户态放到 events 里面，跟 recv 函数很像

```
// poll跟select类似, 其实poll就是把select三个文件描述符集合变成一个集合了。
```
```
int select(int nfds, fd_set * readfds, fd_set *writefds, fd_set *exceptfds,
struct timeval *timeout);
```
```
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

#### 分析加锁

如果有 3 个线程同时操作 epoll，有哪些地方需要加锁？我们用户层面一共就只有 3 个 api 可以使用：

如果同时调用 epoll_create () ，那就是创建三颗红黑树，没有涉及到资源竞争，没有关系。

如果同时调用 epoll_ctl () ，对同一颗红黑树进行，增删改，这就涉及到资源竞争需要加锁了，此时我们
对整棵树进行加锁。

如果同时调用 epoll_wait () ，其操作的是就绪队列，所以需要对就绪队列进行加锁。

我们要扣住 epoll 的工作环境，在应用程序调用 epoll_ctl () ，协议栈会不会有回调操作红黑树结点？调用
epoll_wait () copy 出来的时候，协议栈会不会操作操作红黑树结点加入就绪队列？综上所述：

那么红黑树加什么锁，就绪队列加什么锁呢？

对于红黑树这种节点比较多的时候，采用互斥锁来加锁。

就绪队列就跟生产者消费者一样，结点是从协议栈回调函数来生产的，消费是 epoll_wait () 来消费。那么
对于队列而言，用自旋锁（对于队列而言，插入删除比较简单，cpu 自旋等待比让出的成本更低，所以
用自旋锁）。

#### ET 与 LT 如何实现

ET 边沿触发，只触发一次

LT 水平触发，如果没有读完就一直触发

代码如何实现 ET 和 LT 的效果呢？水平触发和边沿触发不是故意设计出来的，这是自然而然，水到渠成的
功能。水平触发和边沿触发代码只需要改一点点就能实现。从协议栈检测到接收数据，就调用一次回
调，这就是 ET，接收到数据，调用一次回调。而 LT 水平触发，检测到 recvbuf 里面有数据就调用回调。
所以 ET 和 LT 就是在使用回调的次数上面的差异。

那么具体如何实现呢？协议栈流程里面触发回调，是天然的符合 ET 只触发一次的。那么如果是 LT，在
recv 之后，如果缓冲区还有数据那么加入到就绪队列。那么如果是 LT，在 send 之后，如果缓冲区还有空
间那么加入到就绪队列。那么这样就能实现 LT 了。

```
epoll_ctl() 对红黑树加锁
epoll_wait()对就绪队列加锁
回调函数() 对红黑树加锁,对就绪队列加锁
```

（真实实现代码参看：linux-2.6.24/fs/eventpoll. c 文件中的 **ep_send_events** 函数）

#### 说在最后

Linux 相关面试题，是非常常见的面试题。

以上的内容，如果大家能对答如流，如数家珍，基本上面试官会被你震惊到、吸引到。

最终， **让面试官爱到 “不能自已、口水直流”** 。offer，也就来了。

学习过程中，如果有啥问题，大家可以来找 40 岁老架构师尼恩交流。

## 参考文献

https://segmentfault.com/a/1190000022568931

https://blog.csdn.net/MonkeyBrothers/article/details/122860753

https://blog.csdn.net/kaihuishang666/article/details/116095480

[http://www.wjhsh.net/hujinshui-p-10450548.html](http://www.wjhsh.net/hujinshui-p-10450548.html)

## 推荐阅读

《百亿级访问量，如何做缓存架构设计》


《多级缓存架构设计》

《消息推送架构设计》

《阿里 2 面：你们部署多少节点？1000 W 并发，当如何部署？》

《美团 2 面： 5 个 9 高可用 99.999%，如何实现？》

《网易一面：单节点 2000 Wtps，Kafka 怎么做的？》

《字节一面：事务补偿和事务重试，关系是什么？》

《网易一面：25 Wqps 高吞吐写 Mysql，100 W 数据 4 秒写完，如何实现？》

《亿级短视频，如何架构？》

《炸裂，靠“吹牛”过京东一面，月薪 40 K》

《太猛了，靠“吹牛”过顺丰一面，月薪 30 K》

《炸裂了... 京东一面索命 40 问，过了就 50 W+》

《问麻了... 阿里一面索命 27 问，过了就 60 W+》

《百度狂问 3 小时，大厂 offer 到手，小伙真狠！》

《饿了么太狠：面个高级 Java，抖这多硬活、狠活》

《字节狂问一小时，小伙 offer 到手，太狠了！》

《收个滴滴 Offer：从小伙三面经历，看看需要学点啥？》


```
技术自由圈
```
## 未来职业，如何突围：三栖架构师


```
技术自由圈
```
### 成功案例： 2 年翻 3 倍， 35 岁卷王成功转型为架构师

详情：http://topcoder.cloud/forum.php?mod=forumdisplay&fid=43&page=1


技术自由圈


技术自由圈


技术自由圈


```
技术自由圈
```
### 硬核推荐：尼恩 Java 硬核架构班

详情：https://www.cnblogs.com/crazymakercircle/p/9904544.html


技术自由圈


```
技术自由圈
```
##### 架构班（社群 VIP）的起源：

最初的视频，主要是给读者加餐。很多的读者，需要一些高质量的实操、理论视频，所以，我就围绕书，和底层，做了几个
实操、理论视频，然后效果还不错，后面就做成迭代模式了。

##### 架构班（社群 VIP）的功能：^

提供高质量实操项目整刀真枪的架构指导、快速提升大家的:
⚫ 开发水平
⚫ 设计水平
⚫ 架构水平
弥补业务中 CRUD 开发短板，帮助大家尽早脱离具备 3 高能力，掌握：
⚫ 高性能
⚫ 高并发
⚫ 高可用
作为一个高质量的架构师成长、人脉社群，把所有的卷王聚焦起来，一起卷：
⚫ 卷高并发实操
⚫ 卷底层原理
⚫ 卷架构理论、架构哲学
⚫ 最终成为顶级架构师，实现人生理想，走向人生巅峰

##### 架构班（社群 VIP）的目的：^

⚫ 高质量的实操，大大提升简历的含金量，吸引力，增强面试的召唤率
⚫ 为大家提供九阳真经、葵花宝典，快速提升水平
⚫ 进大厂、拿高薪
⚫ 一路陪伴，提供助学视频和指导，辅导大家成为架构师
⚫ 自学为主，和其他卷王一起，卷高并发实操，卷底层原理、卷大厂面试题，争取狠卷 3 月成高手，狠卷 3 年成为顶级架
构师


```
技术自由圈
```
##### N 个超高并发实操项目：简历压轴、个顶个精彩


```
技术自由圈
```
【样章】第 17 章：横扫全网 Rocketmq 视频第 2 部曲: 工业级 rocketmq 高可用（HA）底层原
理和实操

工业级 rocketmq 高可用底层原理，包含：消息消费、同步消息、异步消息、单向消息等不同消息的底层原理和源码实现；
消息队列非常底层的主从复制、高可用、同步刷盘、异步刷盘等底层原理。
工业级 rocketmq 高可用底层原理和搭建实操，包含：高可用集群的搭建。
解决以下难题：
1 、技术难题：RocketMQ 如何最大限度的保证消息不丢失的呢？RocketMQ 消息如何做到高可靠投递？
2 、技术难题：基于消息的分布式事务，核心原理不理解
3 、选型难题： kafka or rocketmq ，该娶谁？
下图链接：https://www.processon.com/view/6178e8ae0e3e7416bde9da19


```
技术自由圈
```
### 简历优化后的成功涨薪案例（ VIP 含免费简历优化）


技术自由圈


技术自由圈


技术自由圈


技术自由圈


技术自由圈


技术自由圈


技术自由圈


```
技术自由圈
```
### 修改简历找尼恩（资深简历优化专家）

⚫ 如果面试表达不好，尼恩会提供简历优化指导

⚫ 如果项目没有亮点，尼恩会提供项目亮点指导

⚫ 如果面试表达不好，尼恩会提供面试表达指导

作为 40 岁老架构师，尼恩长期承担技术面试官的角色：

⚫ 从业以来，“阅历”无数，对简历有着点石成金、改头换面、脱胎换骨的指导能力。

⚫ 尼恩指导过刚刚就业的小白，也指导过 P 8 级的老专家，都指导他们上岸。

如何联系尼恩。尼恩微信，请参考下面的地址：

语雀：https://www.yuque.com/crazymakercircle/gkkw8s/khigna
码云：https://gitee.com/crazymaker/SimpleCrayIM/blob/master/疯狂创客圈总目录.md


