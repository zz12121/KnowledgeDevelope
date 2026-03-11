# Condition 接口的作用是什么？如何使用？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
`Condition`接口提供了类似 `Object.wait()`和 `Object.notify()`的线程协调机制，但与 `Lock`绑定，功能更强大。
- **一个 `Lock`可以创建多个 `Condition`对象**，允许对不同条件下的线程进行分组管理。
- 核心方法：`await()`（等待）、`signal()`（唤醒一个）、`signalAll()`（唤醒所有）。
**使用示例**（生产者-消费者模型）：
```java
Lock lock = new ReentrantLock();
Condition notFull = lock.newCondition(); // 条件：队列未满
Condition notEmpty = lock.newCondition(); // 条件：队列非空

// 生产者
lock.lock();
try {
    while (queue.isFull()) {
        notFull.await(); // 等待"队列未满"条件
    }
    // 生产数据...
    notEmpty.signal(); // 唤醒等待"队列非空"的消费者
} finally {
    lock.unlock();
}

// 消费者
lock.lock();
try {
    while (queue.isEmpty()) {
        notEmpty.await(); // 等待"队列非空"条件
    }
    // 消费数据...
    notFull.signal(); // 唤醒等待"队列未满"的生产者
} finally {
    lock.unlock();
}
```

##关联知识
- 

## 延伸阅读（后续补充）
- 
