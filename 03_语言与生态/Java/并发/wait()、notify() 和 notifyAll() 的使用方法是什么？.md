# wait()、notify() 和 notifyAll() 的使用方法是什么？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
这是最基础的线程协作机制，使用时需遵循特定模式。
- **`wait()`**：使**当前线程释放锁**并进入等待（WAITING）状态，直到被其他线程唤醒。
- **`notify()`**：**随机唤醒一个**在该对象上等待的单个线程。
- **`notifyAll()`**：**唤醒所有**在该对象上等待的线程。
**标准使用范式**：
```java
// 等待方 (Consumer)
synchronized (lockObject) { // 1. 获取锁
    while (!condition) { // 2. 循环检查条件（防止虚假唤醒）
        try {
            lockObject.wait(); // 3. 条件不满足，释放锁并等待
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // 恢复中断状态
        }
    }
    // 4. 条件满足，执行任务
}

// 通知方 (Producer)
synchronized (lockObject) { // 1. 获取锁
    changeCondition(); // 2. 改变条件
    lockObject.notifyAll(); // 3. 通知所有等待线程
}
```
**关键点**：
- **循环检查条件**：必须使用 `while`而非 `if`来检查条件，以应对**虚假唤醒**（线程未被通知也可被唤醒）。
- **同步块**：必须在 `synchronized`同步块或同步方法内调用这些方法。

##关联知识
- 

## 延伸阅读（后续补充）
- 
