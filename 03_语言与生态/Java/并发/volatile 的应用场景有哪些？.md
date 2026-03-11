# volatile 的应用场景有哪些？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
基于 `volatile`的特性，它适用于以下场景：
1. **状态标志位**：用一个 `volatile boolean`变量作为线程运行或中断的标志。由于可见性有保证，其他线程修改标志后，当前线程能立即感知并安全退出。
  ```java
    public class WorkerThread extends Thread {
        private volatile boolean running = true;
    
        public void run() {
            while (running) {
                // 执行任务...
            }
        }
    
        public void stopWorker() {
            running = false; // 其他线程调用此方法，WorkerThread能立即看到false
        }
    }
    ```
1. **双重检查锁定**：在单例模式的DCL中，`volatile`可以防止指令重排序带来的问题，确保其他线程获取到的是完全初始化后的实例，而不是一个半成品的对象
 ```java
    public class Singleton {
        private static volatile Singleton instance;
    
        public static Singleton getInstance() {
            if (instance == null) { // 第一次检查
                synchronized (Singleton.class) {
                    if (instance == null) { // 第二次检查
                        instance = new Singleton(); // volatile 防止这行代码重排序
                    }
                }
            }
            return instance;
        }
    }
    ```
1. **“读多写少”的简单共享变量**：当某个变量读操作远多于写操作，且写操作不依赖于变量的当前值时（即直接赋值，而非先读后写），可以使用 `volatile`来保证可见性，这比使用锁的性能开销要小。

##关联知识
- 

## 延伸阅读（后续补充）
- 
