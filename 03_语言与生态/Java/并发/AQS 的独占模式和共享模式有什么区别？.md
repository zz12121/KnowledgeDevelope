# AQS 的独占模式和共享模式有什么区别？

## 一句话说明（白话）

这是一个 Java关键概念/特性，用于解释语言规则或运行机制。

## 它解决什么问题 / 为什么重要

帮助理解规范与最佳实践，避免常见错误。

## 核心原理（一步步讲清楚）

说明语法/机制，再解释运行时表现与影响。

##典型使用场景

面试常问点、日常开发高频使用。

## 简单例子 /伪代码

给出最小示例说明用法。

## 常见坑与误区

列出1-2个易错点。

##题库要点（原始材料）
|特性|独占模式（Exclusive）|共享模式（Shared）|
|---|---|---|
|**核心概念**​|同一时刻**只有一个线程**能获取资源，如写锁。|同一时刻**多个线程**可以同时获取资源，如读锁。|
|**代表组件**​|`ReentrantLock`|`Semaphore`, `CountDownLatch`, `ReentrantReadWriteLock`的读锁。|
|**关键方法**​|`acquire(int arg)`, `release(int arg)`|`acquireShared(int arg)`, `releaseShared(int arg)`|
|**尝试获取**​|`tryAcquire(int arg)`：成功返回true，失败返回false。|`tryAcquireShared(int arg)`：返回负数表示失败；0表示成功但无剩余资源；正数表示成功且有剩余资源。|
|**尝试释放**​|`tryRelease(int arg)`|`tryReleaseShared(int arg)`|

##关联知识
- 

## 延伸阅读（后续补充）
- 
