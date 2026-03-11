# 为什么 Java 中只有值传递？⁠​

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
关于Java是值传递还是引用传递，是一个经典问题。结论是：**Java中只有值传递**。
- **值传递**：将实际参数的一个**副本**传递给方法。方法内对副本的修改**不会影响**原始变量。
- **引用传递**：将实际参数本身的**引用**（可以理解为内存地址）传递给方法。方法内对形式参数的修改**会影响**原始变量。
在Java中，当传递基本数据类型（如 `int`, `double`）时，传递的是值的副本，这很明显是值传递。当传递引用数据类型（如对象、数组）时，实际上传递的是**引用的副本**（即对象地址的一个拷贝）。这意味着，在方法内部：
- 你可以通过这个副本引用**修改对象的状态**（因为指向的是同一个对象）。
- 但**你不能通过这个副本引用来改变原始引用所指向的对象**（例如，让它指向一个新的对象），因为你操作的只是地址的一个副本。
```java
public class Test {
    public static void changeValue(int num, String str, StringBuilder sb) {
        num = 100; // 修改基本类型副本，不影响原值
        str = "Changed"; // 让副本引用指向新字符串对象，不影响原引用
        sb.append(" World"); // 通过副本引用修改共享对象的内容，原对象被修改
        // sb = new StringBuilder("New"); // 如果让副本引用指向新对象，则不影响原引用
    }

    public static void main(String[] args) {
        int a = 10;
        String s = "Hello";
        StringBuilder builder = new StringBuilder("Hello");
        changeValue(a, s, builder);
        System.out.println(a); // 输出 10
        System.out.println(s); // 输出 "Hello"
        System.out.println(builder); // 输出 "Hello World"
    }
}
```

##关联知识
- 

## 延伸阅读（后续补充）
- 
