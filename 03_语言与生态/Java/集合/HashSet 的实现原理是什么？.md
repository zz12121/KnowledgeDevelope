# HashSet 的实现原理是什么？

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
`HashSet`的核心实现原理是**基于 `HashMap`的键（Key）来存储元素**。
- **内部封装**：当你创建一个 `HashSet`时，其内部实际上封装了一个 `HashMap`实例。默认初始容量是16，负载因子是0.75。
- **值存储**：当您调用 `hashSet.add(element)`时，这个 `element`会被放入底层 `HashMap`的 **Key**​ 上。而 `HashMap`的 **Value**​ 则统一指向一个静态的、不可变的虚拟对象，名为 `PRESENT`。
- 源码如下：
 ```java
    // HashSet 的核心成员变量
    private transient HashMap<E,Object> map;
    private static final Object PRESENT = new Object();
    
    // add 方法的实现
    public boolean add(E e) {
        return map.put(e, PRESENT) == null; // 如果map的put方法返回null，表示添加成功
    }
    ```
- **操作委托**：因此，`HashSet`的所有操作，如 `add`, `remove`, `contains`，本质上都是直接调用底层 `HashMap`的相应方法完成的。

##关联知识
- 

## 延伸阅读（后续补充）
- 
