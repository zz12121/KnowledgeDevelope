# Java 序列化中如果有些字段不想进行序列化，怎么办？⁠​

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
如果你希望类的某些成员变量不被序列化，有几种方法可以实现：
1. **使用 `transient`关键字**：这是最常用和直接的方式。用一个例子来说明，对于一个 `User`类，密码字段非常敏感，我们不希望它被序列化后保存或传输
```java
    public class User implements Serializable {
        private String name;
        private transient String password; // 使用transient修饰，不会被序列化
    
        // ... 构造方法、getter、setter ...
    }
    ```
被 `transient`修饰的字段，在序列化时会被完全忽略。在反序列化后，这些字段的值会被设置为对应类型的默认值（例如，对象引用为 `null`，`int`为 `0`，`boolean`为 `false`）。
2. **使用 `static`修饰符**：静态变量属于类本身，而不属于任何一个对象实例。而序列化是针对对象实例状态的。因此，**静态变量不会被序列化**。
3. **自定义序列化过程**：对于需要更精细控制的场景，可以实现 `Externalizable`接口（它继承自 `Serializable`），并重写 `writeExternal`和 `readExternal`方法，完全自主地决定哪些字段需要被序列化和如何序列化。此外，即使使用 `Serializable`接口，也可以通过在类中定义 `writeObject`和 `readObject`方法来实现自定义序列化逻辑。

##关联知识
- 

## 延伸阅读（后续补充）
- 
