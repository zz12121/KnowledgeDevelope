# HashMap 和 Hashtable 的区别？

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
| 特性            | HashMap                                              | Hashtable                                      |
| ------------- | ---------------------------------------------------- | ---------------------------------------------- |
| **线程安全**​     | **非线程安全**，性能更高。                                      | **线程安全**，方法使用 `synchronized`修饰，性能较低。           |
| **Null 键/值**​ | **允许**一个 `null`键和多个 `null`值。                         | **不允许**键或值为 `null`，会抛出 `NullPointerException`。 |
| **继承体系**​     | 是 Java Collections Framework 的一部分，继承自 `AbstractMap`。 | 是遗留类，继承自陈旧的 `Dictionary`类。                     |
| **迭代器**​      | 是 **fail-fast**​ 的。                                  | 是 **fail-fast**​ 的。                            |

**结论**：Hashtable 是过时的类，不应在新代码中使用。 需要线程安全时，应使用 `ConcurrentHashMap`；不需要线程安全时，使用 `HashMap`。

##关联知识
- 

## 延伸阅读（后续补充）
- 
