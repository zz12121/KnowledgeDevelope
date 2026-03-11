# ConcurrentHashMap 的 put 操作流程是怎样的？

## 一句话说明（白话）

hashCode 是对象散列标识，用于加速查找。

## 它解决什么问题 / 为什么重要

HashMap/HashSet先用 hashCode 定位桶，再用 equals 判断。

## 核心原理（一步步讲清楚）

equals 相等必须 hashCode 相等。

##典型使用场景

Map key、Set 去重。

## 简单例子 /伪代码

equals 基于 id，hashCode也应基于 id。

## 常见坑与误区

hashCode 相同不代表 equals 相同。

##题库要点（原始材料）
JDK 1.8 中的 `put`方法流程精巧地融合了多种并发技术，其核心步骤可参考下图：


```mermaid
flowchart TD
    A[开始put操作] --> B[计算key的hash值]
    B --> C{检查table是否已初始化?}
    C -->|否| D[执行初始化]
    C -->|是| E[根据hash定位到桶i]
    D --> E
    E --> F{判断桶i状态}
    F -->|桶为空| G[使用CAS尝试插入新Node]
    G -->|CAS成功| H[插入成功，break]
    G -->|CAS失败| E
    
    F -->|桶为ForwardingNode<br>表示正在扩容| I[当前线程协助扩容]
    I --> J[扩容后重试]
    
    F -->|桶不为空| K[使用synchronized锁住头节点]
    K --> L{遍历链表或树}
    L -->|找到已存在的key| M[覆盖value]
    L -->|未找到key| N[插入新节点到链表或树]
    
    M & N --> O[判断是否需要树化或扩容]
    O --> P[释放锁]
    P --> H
```

##关联知识
- 

## 延伸阅读（后续补充）
- 
