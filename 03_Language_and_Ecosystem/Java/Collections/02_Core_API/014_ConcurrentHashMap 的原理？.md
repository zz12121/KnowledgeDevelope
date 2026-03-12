# ConcurrentHashMap 的原理？

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
`ConcurrentHashMap`是线程安全HashMap的高性能实现，其在JDK 1.7和1.8中有重大演进。
- **JDK 1.7：分段锁时代**
    采用**分段锁**机制。容器内包含一个 `Segment`数组，每个 `Segment`类似于一个小的 `HashMap`，并继承自 `ReentrantLock`。当线程修改数据时，只需锁住对应的 `Segment`，其他 `Segment`仍可被并发访问。这降低了锁的粒度，提升了并发度。
- **JDK 1.8及以后：`synchronized`+ CAS + 红黑树**
    放弃了 `Segment`，转而采用更细粒度的锁策略。
    - **锁粒度细化**：直接对数组中的每个桶（桶的首个节点）进行加锁（使用 `synchronized`关键字）。
    - **CAS无锁化**：在插入元素等操作中，大量使用**CAS操作**进行无锁化尝试，进一步减少线程阻塞。
    - **结构优化**：当链表长度超过阈值（默认8）且数组容量达到一定值（默认64）时，链表会转换为**红黑树**，以防止在极端哈希冲突下性能退化。查询时间复杂度从O(n)提升到O(log n)。

##关联知识
- 

## 延伸阅读（后续补充）
- 
