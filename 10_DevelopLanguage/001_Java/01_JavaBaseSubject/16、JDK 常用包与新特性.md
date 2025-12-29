###### 1. 熟悉 JDK 哪些包？⁠⁠​
JDK提供了丰富的标准库（包），它们是Java开发的基石。除了表格中提到的核心包，这里再做一些强调和补充：
- **`java.lang`**：这是最核心的包，**默认会被自动导入**。它包含了程序运行的基础，如`Object`、`String`、`Math`、`Thread`和基本数据类型的包装类（如`Integer`）。
- **`java.util`**：这是工具类的宝库，尤其是**集合框架（Collection Framework）**，如`List`、`Set`、`Map`及其常用实现（`ArrayList`、`HashMap`）是日常开发中使用最频繁的。此外，还包括日期时间（旧版）、随机数、扫描器等实用工具。
- **`java.io`与 `java.nio`**：`java.io`提供了标准的输入输出流，用于文件读写和数据流处理。`java.nio`（New I/O）则提供了更高效、非阻塞的I/O操作方式，适用于网络和高并发场景。
- **`java.time`(JDK 8+)**：这是JDK 8引入的新的日期和时间API，彻底解决了旧版`java.util.Date`和`Calendar`类的设计缺陷，强烈推荐使用。
- **其他重要包**：`java.net`（网络编程）、`java.sql`（数据库连接）、`java.util.concurrent`（并发编程）等都是特定领域开发必须掌握的。
###### 2. JDK 1.8 有哪些新特性？⁠⁠​
- **Lambda 表达式**：这是对函数式编程范式的支持，极大地简化了代码，尤其是在遍历集合和实现回调接口时。其语法为 `(参数) -> 表达式`或 `(参数) -> { 语句块 }`。
- **Stream API**：提供了一种高效处理数据集合的声明式编程模型。它可以让你通过流水线式的操作（过滤、映射、归约等）来处理集合，并且能透明地利用多核架构进行并行计算，极大提升了代码的可读性和处理效率。
- **方法引用**：是Lambda表达式的一种语法糖，让你可以直接通过类名或实例名来引用已有的方法，使得代码更加简洁，例如 `System.out::println`。
- **接口的默认方法和静态方法**：允许在接口中提供具有实现的方法。**默认方法**使得在不破坏现有实现类的情况下对接口进行扩展成为可能；**静态方法**允许为接口定义工具方法。
- **新的日期时间 API (`java.time`)**：引入了 `LocalDate`、`LocalTime`、`LocalDateTime`等不可变类，线程安全且设计清晰，完美替代了易错的旧日期API。
- **Optional 类**：一个用于包装可能为`null`值的容器类，旨在强制开发者显式地检查值是否存在，从而避免空指针异常，写出更健壮的代码。
###### 3. Java 8-22 新特性总结?⁠⁠​
Java版本迭代迅速，以下是后续LTS（长期支持）版本和一些重要版本的核心特性摘要：
- **Java 11 (LTS)**：
    - **局部变量类型推断 (`var`)**：允许使用 `var`声明局部变量，编译器会自动推断类型，减少代码冗余。
    - **标准HTTP客户端**：将Java 9/10中孵化的HTTP客户端API标准化，支持HTTP/2和WebSocket，易用且功能强大。
    - **字符串API增强**：新增了如 `isBlank()`、`lines()`、`repeat()`等实用方法。
- **Java 17 (LTS)**：
    - **密封类（Sealed Classes）**：允许类或接口明确规定哪些其他类或接口可以扩展或实现它，提供了对继承关系更精确的控制。
    - **文本块（Text Blocks）**：简化了多行字符串的书写，无需大量的转义和连接操作。
- **Java 21 (LTS) - 又一个重大更新：
    - **虚拟线程（Virtual Threads）**：这是Project Loom的成果，是极其轻量的线程，可以大幅降低编写、维护和观测高吞吐量并发应用程序的难度，是并发编程模型的重大革新。
    - **记录模式（Record Patterns）**：增强了对记录类（Record，Java 16引入）的解构能力，可以更方便地分解记录对象的数据。
    - **序列化集合（Sequenced Collections）**：为集合定义了新的接口，明确提供了获取首尾元素的方法。
###### 4. Stream API 的常用操作有哪些？
Stream的操作分为两类：**中间操作**（返回Stream，可连接）和**终端操作**（返回具体结果或副作用，触发执行）。
- **常用中间操作**：
    - `filter(Predicate p)`：过滤元素。
    - `map(Function f)`：将元素映射为另一种形式。
    - `sorted()`/ `sorted(Comparator com)`：排序。
    - `distinct()`：去重。
- **常用终端操作**：
    - `forEach(Consumer c)`：遍历每个元素。
    - `collect(Collector c)`：将流转换为集合（如List、Set、Map）。
    - `reduce(...)`：将流中的元素反复组合，得到一个值（如求和、求最大）。
    - `count()`：返回流中元素个数。
    - `findFirst()`/ `anyMatch(Predicate p)`：查找或匹配元素。
###### 5. Optional 类的作用是什么？如何使用？
`Optional`的核心思想不是替换所有的`null`，而是作为一种通信工具，明确表示一个返回值可能为空，强制调用方进行处理。
- **创建Optional**：
    - `Optional.of(value)`：创建一个非空值的Optional。
    - `Optional.ofNullable(value)`：创建一个可能为`null`的Optional。
    - `Optional.empty()`：创建一个空的Optional。
- **使用Optional**（应避免直接调用 `get()`，因其在为空时会抛异常）：
    - `isPresent()`：判断值是否存在。
    - `ifPresent(Consumer c)`：如果值存在，则执行给定的操作。
    - `orElse(T other)`：如果值存在则返回，否则返回指定的默认值。
    - `orElseGet(Supplier other)`：延迟版本的`orElse`，只有在值为空时才调用Supplier。
    - `orElseThrow(Supplier exceptionSupplier)`：如果值为空，则抛出指定的异常。
###### 6. Lambda 表达式的底层实现原理是什么？
Lambda表达式的实现并不神秘，它主要依赖于Java 7引入的 `invokedynamic`指令和函数式接口。
1. **函数式接口**：Lambda表达式需要被赋值给一个**只有一个抽象方法的接口**（函数式接口，如 `Runnable`, `Comparator`）。
2. **编译时“脱糖”（Desugar）**：在编译阶段，编译器会为Lambda表达式生成一个对应的**静态私有方法**，该方法包含了Lambda体的逻辑。
3. **`invokedynamic`指令**：字节码中，Lambda表达式出现的位置会被替换为一条 `invokedynamic`指令。这条指令的核心任务是**在首次运行时**动态地解析出应该调用哪个方法（即第2步中生成的静态方法）。
4. **运行时链接**：当JVM首次执行到 `invokedynamic`指令时，它会调用一个称为“Lambda元工厂”（LambdaMetafactory）的机制。这个工厂会根据Lambda的上下文，动态生成一个实现了目标函数式接口的类，并将这个类的实例（即Lambda对象）与生成的静态方法绑定起来。
5. **后续调用**：一旦链接完成，后续再执行到该指令时，就会直接调用之前动态生成的类实例，避免了重复的解析和类生成开销。
这种基于 `invokedynamic`的实现方式非常高效，实现了**按需生成类**，并且具有很好的性能。你可以近似地认为，一个Lambda表达式 `() -> System.out.println("Hello")`最终在运行时等价于一个实现了 `Runnable`接口的匿名内部类，但其实现机制更为优雅和高效。ssh