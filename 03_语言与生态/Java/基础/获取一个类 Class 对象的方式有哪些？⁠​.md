# 获取一个类 Class 对象的方式有哪些？⁠​

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
要进行任何反射操作，第一步都是获取该类的 `Class`对象。一个类在 JVM 中只有唯一的一个 `Class`对象。主要有三种核心方式：
1. **`类名.class`（类字面常量）**
    这是最安全、性能最好的方式，因为在编译期就能确定。
```java
    Class<String> stringClass = String.class;
    Class<Integer> intClass = int.class; // 也适用于基本数据类型
    ```
2. **`对象实例.getClass()`**
    ```java
    String str = "Hello";
    Class<? extends String> strClass = str.getClass();
    ```
3. **`Class.forName("类的全限定名")`**
    这是**最常用**的方式，尤其适用于框架开发。你可以通过配置文件、数据库等外部资源传入类的完整路径名，实现动态加载。
    ```java
    // 可能会抛出 ClassNotFoundException
    Class<?> aClass = Class.forName("java.lang.String");
    ```
此外，还可以通过类加载器（`ClassLoader`）的 `loadClass`方法获取，这提供了更底层的控制。

##关联知识
- 

## 延伸阅读（后续补充）
- 
