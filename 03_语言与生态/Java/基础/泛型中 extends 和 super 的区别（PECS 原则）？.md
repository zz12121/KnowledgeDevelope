# 泛型中 extends 和 super 的区别（PECS 原则）？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
PECS 的全称是 **"Producer-Extends, Consumer-Super"**。这个原则是 Joshua Bloch 在《Effective Java》中提出的，用于指导在方法参数中何时使用 `extends`通配符，何时使用 `super`通配符。
- **Producer (生产者) - 使用 `extends`**：如果你需要一个**提供（生产）`T`类型对象的数据源，应该使用 `<? extends T>`。因为你主要目的是从结构中**读取**数据。
- **Consumer (消费者) - 使用 `super`**：如果你需要一个**接收（消费）`T`类型对象的数据池，应该使用 `<? super T>`。因为你主要目的是向结构中**写入**数据。
**经典示例：`java.util.Collections.copy`方法**
```java
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    // dest 是消费者，我们向其中写入T类型的元素，所以用 ? super T
    // src 是生产者，我们从中读取T类型的元素，所以用 ? extends T
    for (int i=0; i<src.size(); i++) {
        dest.set(i, src.get(i));
    }
}
```

##关联知识
- 

## 延伸阅读（后续补充）
- 
