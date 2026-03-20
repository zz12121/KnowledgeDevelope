# Future和Promise组件

## 概述

Netty的Future和Promise是其异步编程模型的核心，提供了非阻塞I/O操作的结果处理能力。Netty对JDK原生Future进行了增强，支持回调机制，避免了轮询或阻塞。

## Future接口

继承自`java.util.concurrent.Future`，代表异步操作的结果：

```java
public interface Future<V> extends java.util.concurrent.Future<V> {
    // 判断是否成功
    boolean isSuccess();
    
    // 获取异常原因
    Throwable cause();
    
    // 添加监听器
    Future<V> addListener(GenericFutureListener<? extends Future<? super V>> listener);
    
    // 同步等待完成，失败则抛异常
    Future<V> sync() throws InterruptedException;
    
    // 仅阻塞等待，不抛异常
    Future<V> await() throws InterruptedException;
}
```

## Promise接口

Promise是Netty特有的Future子接口，核心是可以手动设置操作结果：

```java
public interface Promise<V> extends Future<V> {
    // 设置成功结果
    Promise<V> setSuccess(V result);
    
    // 设置失败结果
    Promise<V> setFailure(Throwable cause);
    
    // 尝试设置结果
    boolean trySuccess(V result);
    boolean tryFailure(Throwable cause);
}
```

## 与JDK Future对比

| 特性 | JDK Future | Netty Future |
|------|------------|--------------|
| 状态查询 | 无 | isSuccess() |
| 获取异常 | 无 | cause() |
| 回调机制 | 无 | addListener() |
| 阻塞方式 | get() | sync()/await() |

## 工作流程

### 生产者-消费者模型

1. **创建Promise**：上层调用创建`DefaultChannelPromise`
2. **返回Future**：Promise作为ChannelFuture返回给调用者
3. **传递执行**：Promise传递给EventLoop线程执行实际IO操作
4. **设置结果**：IO完成后，调用`promise.setSuccess()`或`promise.setFailure()`
5. **通知结果**：
   - 阻塞线程被唤醒
   - 监听器回调立即触发

### 示例代码

```java
// 同步等待
ChannelFuture future = channel.closeFuture();
future.sync();

// 异步回调
future.addListener(future -> {
    if (future.isSuccess()) {
        System.out.println("连接关闭成功");
    } else {
        System.out.println("关闭失败：" + future.cause());
    }
});
```

## 核心实现

### DefaultPromise

- **状态存储**：使用volatile修饰的result字段
- **线程安全**：使用`AtomicReferenceFieldUpdater`
- **阻塞唤醒**：使用wait()和notifyAll()机制
- **监听器管理**：使用listeners对象维护回调列表

### ChannelFuture

继承Netty Future，专门用于处理IO操作结果：

```java
public interface ChannelFuture extends Future<Void> {
    // 获取关联的Channel
    Channel channel();
}
```

## 设计优势

1. **非阻塞**：通过回调机制，无需阻塞线程
2. **链式操作**：支持流式API添加多个监听器
3. **线程安全**：保证多线程环境下的正确性
4. **状态可查询**：随时获取操作状态和结果

## 总结

Future和Promise是Netty异步编程的基石，通过"生产者-消费者"模型，实现了高效的非阻塞IO操作。Promise负责设置结果，Future负责获取结果，二者配合实现了Netty的高性能异步通信。
