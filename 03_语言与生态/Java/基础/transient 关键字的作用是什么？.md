# transient 关键字的作用是什么？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
`transient`关键字的主要作用是**在对象序列化时，标记一个成员变量不被序列化**。
当一个类实现了 `java.io.Serializable`接口后，它的对象就可以被序列化（即转换成字节流以便存储或传输）。默认情况下，对象的所有非静态和非 `transient`的成员变量都会被序列化。使用 `transient`修饰变量，可以将其排除在序列化过程之外。
**为什么要使用 transient？**
1. **敏感信息保护**：如密码、银行卡号等字段，不希望被持久化到磁盘或通过网络传输。
2. **节省空间与提升性能**：对于某些可以由其他字段推导出的派生数据或大型临时数据，无需序列化。
3. **序列化无意义的字段**：如线程句柄等依赖于特定JVM运行环境的字段，序列化它们没有意义。
**示例：**
```java
import java.io.Serializable;

public class User implements Serializable {
    private String username;
    private transient String password; // 密码不被序列化

    // ... 构造方法、getter、setter ...
}
```
当序列化一个 `User`对象后，再反序列化回来，`password`字段的值将是 `null`（对于基本数据类型，则是其默认值，如 `0`、`false`等）。
**关键点：**
- `transient`只能修饰变量，不能修饰方法和类。
- 静态变量（`static`）无论是否被 `transient`修饰，都不会被序列化，因为序列化是针对对象实例状态的。
- 如果类实现的是 `Externalizable`接口（而非 `Serializable`），序列化过程完全由程序员控制的 `writeExternal`和 `readExternal`方法决定，`transient`关键字在此无效。

##关联知识
- 

## 延伸阅读（后续补充）
- 
