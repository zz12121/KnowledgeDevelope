# 说说你对 equals 与== 的理解？⁠​

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
|操作符/方法|`==`|`equals`|
|---|---|---|
|**比较本质**​|**比较的是值（Value）**​|**比较的是对象的内容（Content）**​|
|**对于基本类型**​|比较的是**变量中存储的数值**是否相等。`int a=10; int b=10; a==b // true`|基本类型不能调用`equals`方法。|
|**对于引用类型**​|比较的是**两个引用是否指向内存中的同一个对象**（即比较内存地址）。|默认行为与`==`相同（Object类中的实现）。但绝大多数类（如String、Integer）会**重写**此方法，用于比较两个对象**在逻辑上是否等价**。|

**示例：**
```java
String str1 = new String("hello");
String str2 = new String("hello");
String str3 = str1;

System.out.println(str1 == str2);   // false，比较地址，不是同一个对象
System.out.println(str1 == str3);   // true，指向同一个对象
System.out.println(str1.equals(str2)); // true，比较内容，内容相同
```

##关联知识
- 

## 延伸阅读（后续补充）
- 
