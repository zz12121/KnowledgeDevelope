# Java中的泛型为什么是伪泛型？⁠​什么是类型擦除？

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
Java 的泛型是通过 **类型擦除**​ 来实现的，这是理解 Java 泛型底层机制的关键。
- **什么是类型擦除？**
    编译器在编译 Java 源码时，会**移除所有泛型类型信息**。类型参数 `<T>`会被替换为其**上限**（未指定上限时默认为 `Object`），并在必要的地方插入强制类型转换。
```java
    // 编译前
    List<String> list = new ArrayList<>();
    list.add("Hello");
    String s = list.get(0);
    
    // 编译后（类型擦除后的近似效果）
    List list = new ArrayList(); // <String> 被擦除
    list.add("Hello");
    String s = (String) list.get(0); // 编译器自动插入转换
    ```
- **为什么说 Java 泛型是“伪泛型”？**
    因为泛型信息只存在于编译期，**运行时是没有泛型类型信息的**。例如，`List<String>`和 `List<Integer>`在运行时都是 `List`类，无法通过反射在运行时判断一个 `List`到底包含什么类型的元素。这与 C# 等语言在运行时保留类型信息的“真泛型”有本质区别，故被称为伪泛型。
    ```java
    List<String> strList = new ArrayList<>();
    List<Integer> intList = new ArrayList<>();
    System.out.println(strList.getClass() == intList.getClass()); // 输出 true
    ```
- **类型擦除带来的限制**
    - **不能使用基本类型作为类型参数**：如 `List<int>`是错误的，必须使用包装类 `List<Integer>`，因为擦除后 `T`是 `Object`，而基本类型不是对象。
    - **不能实例化类型参数**：`new T()`是非法的，因为擦除后 `T`变成 `Object`，这没有意义。
    - **不能创建泛型数组**：如 `new List<String>[10]`通常是非法的，因为数组需要在运行时知道其元素的确切类型。
    - **`instanceof`检查失效**：不能使用 `obj instanceof T<String>`。

##关联知识
- 

## 延伸阅读（后续补充）
- 
