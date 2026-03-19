# Vertx 线程模型

## 1. Vertx 简介

Vert.x 是一个基于 JVM 的响应式编程框架，被 MyCAT2 作为底层网络通信框架。

### 1.1 性能特点

Vert.x 具有出色的性能表现，能够支撑高并发的网络请求。

### 1.2 核心概念

| 概念 | 说明 |
|------|------|
| Event Bus | 事件总线，用于 Verticle 之间通信 |
| Event Loop | 事件循环线程，处理 IO 事件 |
| Verticle | 部署和运行的代码块 |

---

## 2. 同步、异步、阻塞、非阻塞

### 2.1 网络包接收流程

```
网卡 → 操作系统 → TCP/IP协议栈 → 应用层缓冲区 → 读取数据
```

### 2.2 阻塞 vs 非阻塞

**阻塞**：
- 调用 IO 操作时，如果没有数据，线程会一直等待
- 线程被挂起，无法处理其他任务

**非阻塞**：
- 调用 IO 操作时，立即返回
- 线程可以继续处理其他任务

### 2.3 同步 vs 异步

**同步**：
- 调用者主动等待结果
- 结果返回后继续执行

**异步**：
- 调用者不主动等待结果
- 通过回调/事件通知获取结果

---

## 3. NIO

Vert.x 底层使用 Netty 的 NIO 模型：

```java
// EventLoopGroup 的创建
eventLoopGroup = new NioEventLoopGroup(
    options.getEventLoopPoolSize(), 
    eventLoopThreadFactory
);
eventLoopGroup.setIoRatio(NETTY_IO_RATIO);
```

### EventLoop 选择策略

```java
// 2 的幂次方时使用位运算
return executors[idx.getAndIncrement() & executors.length - 1];

// 非 2 的幂次方时使用取模
return executors[Math.abs(idx.getAndIncrement() % executors.length)];
```

---

## 4. Vertx 线程模型

### 4.1 核心原则

> **最重要的一点：永远不要阻塞 Event Loop 线程**
> 
> 因为一旦处理事件的线程被阻塞了，事件就会一直积压着不能被处理，整个应用也就不能正常工作了。

### 4.2 Verticle

Verticle 是 Vert.x 中执行单元，在同一个 Vertx 实例中可以同时执行多个 Verticle。

**特点**：
- 默认情况下一个 Vert.x 实例维护了 **N**（默认 N = CPU 核数 × 2）个 Event Loop 线程
- Verticle 实例可以使用任意 Vert.x 支持的编程语言编写
- 可以将 Verticle 想成 Actor Model 中的 Actor

**类型**：
| 类型 | 说明 |
|------|------|
| Standard Verticle | 最常用，运行在 Event Loop 线程上 |
| Worker Verticle | 运行在 Worker Pool 中的线程上 |
| Multi-Threaded Worker Verticle | 运行在 Worker Pool 中，可以由多个线程同时执行 |

### 4.3 VertxThread

Vert.x 中的 Event Loop 线程及 Worker 线程都用 `VertxThread` 类表示。

---

## 5. Vert.x Context

### 5.1 Context 类型

Vert.x 中存在三种 Context，与线程种类相对应：

| Context 类型 | 说明 |
|-------------|------|
| EventLoopContext | 绑定 Event Loop 线程，处理 IO 事件 |
| WorkerContext | 运行阻塞任务 |
| MultiThreadedWorkerContext | 多线程 Worker |

### 5.2 Context 获取规则

1. 如果当前线程是 Vert.x 线程（VertxThread），则复用此线程上绑定的 Context
2. 如果当前线程是普通线程，就创建新的 Context

---

## 6. 线程池

Vert.x 中包含多个线程池：

### 6.1 Event Loop 线程池

```java
eventLoopThreadFactory = new VertxThreadFactory(
    "vert.x-eventloop-thread-", 
    checker, 
    false, 
    options.getMaxEventLoopExecuteTime()
);
eventLoopGroup = new NioEventLoopGroup(
    options.getEventLoopPoolSize(), 
    eventLoopThreadFactory
);
```

### 6.2 Worker 线程池

```java
ExecutorService workerExec = Executors.newFixedThreadPool(
    options.getWorkerPoolSize(),
    new VertxThreadFactory(
        "vert.x-worker-thread-", 
        checker, 
        true, 
        options.getMaxWorkerExecuteTime()
    )
);
```

### 6.3 内部阻塞线程池

```java
ExecutorService internalBlockingExec = Executors.newFixedThreadPool(
    options.getInternalBlockingPoolSize(),
    new VertxThreadFactory(
        "vert.x-internal-blocking-", 
        checker, 
        true, 
        options.getMaxWorkerExecuteTime()
    )
);
```

### 6.4 Acceptor Event Loop 线程池

```java
acceptorEventLoopGroup = new NioEventLoopGroup(
    1, 
    acceptorEventLoopThreadFactory
);
```

负责处理客户端的连接请求。

---

## 7. Event Bus

Event Bus 是 Verticle 之间通信的桥梁：

```java
public class Sender extends AbstractVerticle {
    @Override
    public void start() {
        EventBus eb = vertx.eventBus();
        vertx.setPeriodic(1000, v -> {
            eb.send("ping-address", "ping!");
        });
    }
}
```

```java
public class Receiver extends AbstractVerticle {
    @Override
    public void start() {
        vertx.eventBus().consumer("ping-address", message -> {
            System.out.println("Received: " + message.body());
        });
    }
}
```

---

⬆️ [[索引]]
