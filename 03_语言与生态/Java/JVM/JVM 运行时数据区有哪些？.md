# JVM 运行时数据区有哪些？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
JVM 运行时数据区是 Java 虚拟机在执行 Java 程序时所管理的内存区域，可根据线程共享与否进行划分。其核心组件与关系如下：

```mermaid
flowchart TD
    A[JVM 运行时数据区] --> B[线程私有]
    A --> C[线程共享]
    
    B --> B1[程序计数器<br>PC Register]
    B --> B2[Java 虚拟机栈<br>JVM Stack]
    B --> B3[本地方法栈<br>Native Method Stack]
    
    C --> C1[堆<br>Heap]
    C --> C2[方法区<br>Method Area]
    C --> C3[运行时常量池<br>Runtime Constant Pool]
    
    B2 --> B2a["栈帧（Stack Frame）"]
    B2a --> B2a1[局部变量表]
    B2a --> B2a2[操作数栈]
    B2a --> B2a3[动态链接]
    B2a --> B2a4[方法返回地址]
    
    C1 --> C1a[新生代 Young]
    C1a --> C1a1[Eden区]
    C1a --> C1a2[Survivor From]
    C1a --> C1a3[Survivor To]
    C1 --> C1b[老年代 Old]
```

##关联知识
- 

## 延伸阅读（后续补充）
- 
