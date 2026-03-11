# 线程池有哪些常见的类型？为什么不建议使用 Executors 创建线程池？

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
通过`Executors`工具类，可以快速创建几种常见类型的线程池，但它们各有适用场景：

| 线程池类型                     | 创建方法                                  | 特点与适用场景                                               |
| ------------------------- | ------------------------------------- | ----------------------------------------------------- |
| **FixedThreadPool**​      | `Executors.newFixedThreadPool(n)`     | 固定线程数，无界队列（`LinkedBlockingQueue`）。适用于负载较重、需要稳定线程数的场景。 |
| **CachedThreadPool**​     | `Executors.newCachedThreadPool()`     | 线程数可灵活回收，几乎无界。适用于执行大量短生命周期异步任务的场景。                    |
| **SingleThreadExecutor**​ | `Executors.newSingleThreadExecutor()` | 只有一个线程，保证任务顺序执行。适用于需要顺序执行任务的场景。                       |
| **ScheduledThreadPool**​  | `Executors.newScheduledThreadPool(n)` | 支持定时或周期性任务执行。适用于需要任务调度（如延时、周期执行）的场景。                  |
| **WorkStealingPool**​     | `Executors.newWorkStealingPool()`     | 基于ForkJoin框架，使用工作窃取算法。适用于计算密集型任务，能有效利用多核CPU。          |

**特别注意**：`FixedThreadPool`和`SingleThreadExecutor`使用的无界队列（默认容量为`Integer.MAX_VALUE`），以及`CachedThreadPool`和`ScheduledThreadPool`允许创建的最大线程数（`Integer.MAX_VALUE`），都可能因任务堆积或线程暴增导致OOM风险。因此，**生产环境建议直接使用`ThreadPoolExecutor`构造函数创建线程池**，以便明确设定合理的队列容量和最大线程数。

##关联知识
- 

## 延伸阅读（后续补充）
- 
