# 关键字深度解析（final / static / this / super）

> **核心关键词**：final 不可变、static 类级别、this 当前对象、super 父类、编译期常量

---

## 一、final 关键字

### 1.1 修饰变量

**局部变量/实例变量**：一旦赋值不可再修改（引用不可变，但引用的对象内容可变）：

```java
final int x = 10;
x = 20;  // ❌ 编译错误

final List<String> list = new ArrayList<>();
list.add("hello");  // ✅ 引用不变，但 list 内部可以修改
list = new ArrayList<>();  // ❌ 引用不能重指向
```

**编译期常量（static final 基本类型/String）**：
```java
static final int MAX = 100;
// 编译期直接替换：所有引用 MAX 的地方都变成 100
// 修改这个常量后，引用它的类需要重新编译才能看到新值
```

**final 与内存模型**：
```java
// final 字段的初始化保证：构造器执行完成后，其他线程一定能看到 final 字段的值
// 不需要额外同步（这是 final 的"写屏障"语义）
class SafePoint {
    final int x, y;
    SafePoint(int x, int y) { this.x = x; this.y = y; }
    // 只要构造器没有"泄漏 this"，其他线程看到的 x/y 一定是构造好的值
}
```

### 1.2 修饰方法

被 `final` 修饰的方法**不能被子类重写**，可以被继承：

```java
class Parent {
    final void doWork() { System.out.println("Parent work"); }
}

class Child extends Parent {
    // @Override void doWork() {}  // ❌ 编译错误，不能重写 final 方法
}
```

**性能提示**：`final` 方法可以被 JIT 编译器内联（因为不会被重写，调用目标确定）。

### 1.3 修饰类

`final` 类**不能被继承**：

```java
public final class String { ... }   // String 是 final 类，安全、不可变
public final class Integer { ... }  // 所有包装类都是 final
```

---

## 二、static 关键字

### 2.1 static 变量（类变量）

属于**类本身**，所有实例共享，存储在**方法区/元空间**：

```java
class Counter {
    static int count = 0;  // 所有 Counter 实例共享
    int id;
    
    Counter() {
        this.id = ++count;  // 每创建一个实例，count+1
    }
}

Counter c1 = new Counter(); // c1.id=1, Counter.count=1
Counter c2 = new Counter(); // c2.id=2, Counter.count=2
System.out.println(Counter.count); // 2（通过类名访问，不要用实例）
```

### 2.2 static 方法（类方法）

- 可以不创建实例直接调用：`ClassName.method()`
- **不能使用 `this` 或 `super`**（没有当前对象的概念）
- **不能直接访问实例变量和实例方法**（只能访问 static 成员）

```java
class MathUtil {
    static int add(int a, int b) { return a + b; }   // 工具方法，不需要实例状态
}

MathUtil.add(1, 2);  // ✅ 直接通过类名调用

class Wrong {
    int instanceVar = 10;
    static void staticMethod() {
        System.out.println(instanceVar);  // ❌ 编译错误：不能在静态方法中访问实例变量
    }
}
```

### 2.3 static 代码块

类加载**初始化阶段**执行，只执行一次，常用于复杂的静态初始化：

```java
class Config {
    static final Properties props;
    
    static {
        props = new Properties();
        try {
            props.load(Config.class.getResourceAsStream("/config.properties"));
            System.out.println("配置加载完成");
        } catch (IOException e) {
            throw new ExceptionInInitializerError(e);
        }
    }
}
```

**执行顺序**：父类静态块 → 子类静态块 → 父类实例块/构造器 → 子类实例块/构造器

```java
class Parent {
    static { System.out.println("Parent static"); }
    { System.out.println("Parent instance"); }
    Parent() { System.out.println("Parent constructor"); }
}

class Child extends Parent {
    static { System.out.println("Child static"); }
    { System.out.println("Child instance"); }
    Child() { System.out.println("Child constructor"); }
}

new Child();
// 输出：
// Parent static
// Child static
// Parent instance
// Parent constructor
// Child instance
// Child constructor
```

### 2.4 static 内部类

不持有外部类实例的引用，可以单独实例化：

```java
class Outer {
    private int x = 10;
    
    static class StaticNested {
        void show() {
            // System.out.println(x);  // ❌ 不能访问外部类实例变量
            Outer outer = new Outer();
            System.out.println(outer.x); // ✅ 可以通过创建外部类实例访问
        }
    }
    
    class Inner {  // 非静态内部类，持有外部类实例
        void show() {
            System.out.println(x);  // ✅ 可以直接访问外部类成员
        }
    }
}

// 使用
Outer.StaticNested nested = new Outer.StaticNested();  // 不需要 Outer 实例
Outer.Inner inner = new Outer().new Inner();           // 需要 Outer 实例
```

---

## 三、this 关键字

### 3.1 引用当前对象

```java
class User {
    String name;
    
    User(String name) {
        this.name = name;  // this.name 是字段，name 是参数，消除歧义
    }
    
    User withName(String name) {
        this.name = name;
        return this;  // 返回当前对象，支持链式调用
    }
}

user.withName("张三").withName("李四");  // 链式调用
```

### 3.2 调用其他构造器

```java
class Point {
    double x, y, z;
    
    Point(double x, double y) {
        this.x = x;
        this.y = y;
    }
    
    Point(double x, double y, double z) {
        this(x, y);  // 调用上面的构造器，必须是第一行
        this.z = z;
    }
}
```

---

## 四、super 关键字

### 4.1 调用父类构造器

```java
class Animal {
    String name;
    Animal(String name) { this.name = name; }
}

class Dog extends Animal {
    String breed;
    Dog(String name, String breed) {
        super(name);  // 必须是第一行，调用父类构造器
        this.breed = breed;
    }
}
```

### 4.2 调用父类方法/访问父类字段

```java
class Animal {
    String describe() { return "Animal"; }
}

class Dog extends Animal {
    @Override
    String describe() {
        return super.describe() + " (Dog)";  // 调用父类的 describe，然后追加
    }
}
```

### 4.3 在接口中调用特定父接口的 default 方法

```java
interface A { default void hello() { System.out.println("A"); } }
interface B { default void hello() { System.out.println("B"); } }

class C implements A, B {
    @Override
    public void hello() {
        A.super.hello();  // 显式调用接口 A 的 default 方法
    }
}
```

---

## 五、常见面试题解析

### final 修饰的集合能添加元素吗？

```java
final List<String> list = new ArrayList<>();
list.add("hello");  // ✅ 可以！final 只保证引用不变，不保证对象内容不变
list = new ArrayList<>();  // ❌ 不行，引用不能改变
```

### 为什么匿名内部类/Lambda 中使用的局部变量必须是 final 或 effectively final？

```java
void method() {
    int x = 10;
    // x = 20;  // 加了这行就会报错（x 就不是 effectively final 了）
    
    Runnable r = () -> System.out.println(x);  // 使用了外部变量 x
    // JVM 需要将 x 的值"捕获"复制进 Lambda，
    // 如果 x 可变，复制的值和原变量就不同步，容易引发歧义
}
```

---

## 六、面试要点速查

| 问题 | 要点 |
|------|------|
| final 变量能保证线程安全吗 | final 字段在构造器完成后对其他线程可见（final 写屏障），但 final 引用的对象内容仍不安全 |
| static 方法能重写吗 | 不能重写（Override），但可以被子类的同名 static 方法隐藏（Hide）。隐藏与重写不同，没有多态 |
| this() 和 super() 能同时出现 | 不能，两者都要求在构造器第一行，只能二选一 |
| static 内部类和内部类的区别 | static 内部类不持有外部类引用，可单独实例化，不会导致外部类内存泄漏；非 static 内部类持有外部类引用 |
| final 方法比普通方法快吗 | final 方法更容易被 JIT 内联，但现代 JIT 即使没有 final 也能通过逃逸分析实现内联 |


---

**相关面试题** → [[../../10_Developlanguage/001_java/01_JavaBaseSubject/04、关键字与运算符|04、关键字与运算符]]
