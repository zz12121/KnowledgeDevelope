# 如何使用 JMH 进行并发性能测试？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
JMH是Java官方推出的专门用于进行Java微基准测试的工具，对于并发性能测试尤其重要，因为它能有效避免JVM的JIT编译器优化等因素对测试结果的干扰。
1. **添加JMH依赖**。
2. **编写测试用例**：
```java
    import org.openjdk.jmh.annotations.*;
    import java.util.concurrent.TimeUnit;
    import java.util.concurrent.atomic.AtomicLong;
    import java.util.concurrent.locks.ReentrantLock;
    
    @BenchmarkMode(Mode.Throughput) // 测试模式：吞吐量
    @OutputTimeUnit(TimeUnit.SECONDS) // 输出时间单位
    @State(Scope.Group) // 定义测试状态的作用范围
    public class LockBenchmark {
        private final ReentrantLock lock = new ReentrantLock();
        private long counter = 0;
        private AtomicLong atomicCounter = new AtomicLong(0);
    
        @Benchmark
        @Group("lock")
        public long testWithLock() {
            lock.lock();
            try {
                counter++;
                return counter;
            } finally {
                lock.unlock();
            }
        }
    
        @Benchmark
        @Group("atomic")
        public long testAtomic() {
            return atomicCounter.incrementAndGet();
        }
    }
    ```
3. **运行与解读**：通过JMH可以科学地比较`synchronized`、`ReentrantLock`和无锁方案在真实高并发环境下的吞吐量、平均耗时等指标，从而做出正确的技术选型。

##关联知识
- 

## 延伸阅读（后续补充）
- 
