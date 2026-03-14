###### 1. 熟悉 JDK 哪些包？

[[../../../20_JavaKnowledge/01_Java基础/13、JDK新特性全览（8-21）#一、JDK 8（2014，里程碑版本）|📖]]
JDK自带的标准库很丰富，面试最常考的几个包：

**`java.lang`**：最核心的包，**自动导入不需要import**。包含Object、String、Math、Thread、基本类型包装类（Integer等）。几乎任何Java程序都会用到。

**`java.util`**：工具类宝库。最重要的是**集合框架**（List、Set、Map及其实现类）、日期时间（旧版Date/Calendar）、Scanner、Random等。日常开发里`java.util`的东西用得最多。

**`java.io`和`java.nio`**：IO操作。java.io提供流式IO，java.nio提供非阻塞IO，nio性能更好适合高并发网络场景。

**`java.time`**（JDK 8+）：新日期时间API，`LocalDate`、`LocalDateTime`、`ZonedDateTime`等，不可变、线程安全、设计优雅，完全替代旧版Date/Calendar，**现在的项目一律用这个**。

**`java.util.concurrent`**：并发编程工具箱，线程池、锁、并发集合、CountDownLatch等都在这里。并发编程必学。

**`java.net`**：网络编程；`java.sql`：数据库连接；`java.lang.reflect`：反射。

###### 2. JDK 1.8 有哪些新特性？

[[../../../20_JavaKnowledge/01_Java基础/13、JDK新特性全览（8-21）#一、JDK 8（2014，里程碑版本）|📖]]
Java 8是划时代的版本，这些特性改变了Java编程方式：

**Lambda表达式**：用`(参数) -> 表达式`的方式简洁地实现函数式接口，代替冗长的匿名内部类。

```java
// 以前
Collections.sort(list, new Comparator<String>() {
    public int compare(String a, String b) { return a.compareTo(b); }
});
// Java 8
Collections.sort(list, (a, b) -> a.compareTo(b));
```

**Stream API**：对集合进行声明式、流水线式操作，可以链式调用filter/map/reduce等，还支持并行流处理。

```java
list.stream().filter(s -> s.length() > 3).map(String::toUpperCase).collect(Collectors.toList());
```

**方法引用**：Lambda的简化写法，直接引用已有方法：`System.out::println`、`String::valueOf`。

**接口默认方法和静态方法**：接口可以有`default`方法（带实现），让接口可以在不破坏现有实现类的情况下增加新方法，解决了接口扩展的历史难题。

**新日期时间API（java.time）**：`LocalDate`、`LocalTime`、`LocalDateTime`，不可变线程安全，替代混乱的旧版Date/Calendar。

**Optional类**：用来包装可能为null的值，强制调用方显式处理空值，减少NPE。

###### 3. Java 8-21 新特性总结？

[[../../../20_JavaKnowledge/01_Java基础/13、JDK新特性全览（8-21）#六、JDK 21（LTS，2023）|📖]]
**Java 11（LTS）**：`var`局部变量类型推断（编译器自动推断类型，少写类型声明）；标准HTTP Client API，支持HTTP/2；字符串新方法`isBlank()`、`lines()`、`repeat(n)`。

**Java 14/15**：`instanceof`模式匹配（`if (obj instanceof String s)`，检查和转型一步完成）；文本块（`"""`多行字符串，告别字符串拼接地狱）；Records记录类（不可变数据类，自动生成equals/hashCode/toString）。

**Java 17（LTS）**：密封类（`sealed class`，明确限制哪些子类可以继承，让继承体系更可控）；文本块、switch表达式、模式匹配正式发布。

**Java 21（LTS，重大版本）**：虚拟线程（Project Loom，极轻量的线程，百万并发不是梦，彻底改变高并发编程模型）；Sequenced Collections（为集合增加获取首尾元素的标准API）；Record Patterns（对记录类的解构匹配）。

###### 4. Stream API 的常用操作有哪些？

[[../../../20_JavaKnowledge/01_Java基础/13、JDK新特性全览（8-21）#一、JDK 8（2014，里程碑版本）|📖]]
Stream操作分两类：中间操作（返回Stream可以链式调用）和终端操作（产生结果，触发执行）。

**常用中间操作（懒执行，只在终端操作触发时才真正执行）**：
- `filter(Predicate)` - 过滤，保留满足条件的元素
- `map(Function)` - 映射，把每个元素转换成另一种形式
- `flatMap(Function)` - 扁平化映射，把流中的流合并成一个流
- `sorted()`/`sorted(Comparator)` - 排序
- `distinct()` - 去重
- `limit(n)` - 取前n个
- `skip(n)` - 跳过前n个

**常用终端操作（触发整个流水线执行）**：
- `collect(Collector)` - 收集结果，转换成List/Set/Map等
- `forEach(Consumer)` - 遍历
- `count()` - 统计数量
- `reduce()` - 归约操作（求和、求积等）
- `findFirst()`/`findAny()` - 查找元素
- `anyMatch()`/`allMatch()`/`noneMatch()` - 匹配检查
- `min()`/`max()` - 找最值

###### 5. Optional 类的作用是什么？如何使用？

[[../../../20_JavaKnowledge/01_Java基础/13、JDK新特性全览（8-21）#一、JDK 8（2014，里程碑版本）|📖]]
Optional不是为了彻底消灭null，而是作为返回值类型，明确表示"这个返回值可能为空"，强制调用方处理空值情况，减少NPE。

**创建**：`Optional.of(value)`（非空值）、`Optional.ofNullable(value)`（可能为空）、`Optional.empty()`。

**使用（避免直接`get()`，它在空时抛异常）**：

```java
Optional<String> opt = Optional.ofNullable(user.getName());

opt.isPresent();                    // 判断是否有值
opt.ifPresent(name -> log(name));   // 有值才执行
opt.orElse("默认值");               // 有值返回，否则返回默认值
opt.orElseGet(() -> computeDefault()); // 延迟计算默认值
opt.orElseThrow(() -> new BusinessException("名称不能为空")); // 空则抛异常
opt.map(String::toUpperCase);       // 映射转换，空时返回empty
```

最佳实践：主要用作方法返回值类型，不要用在字段或参数上；不要用`Optional.get()`，用上面那些安全的方法。

###### 6. Lambda 表达式的底层实现原理是什么？

[[../../../20_JavaKnowledge/01_Java基础/13、JDK新特性全览（8-21）#一、JDK 8（2014，里程碑版本）|📖]]
Lambda的实现原理不是内部类（尽管效果类似），而是依赖Java 7引入的`invokedynamic`指令：

**编译期**：编译器把Lambda体的逻辑提取成一个私有静态方法，Lambda调用的位置替换成一条`invokedynamic`指令。

**首次运行时**：JVM执行`invokedynamic`时，调用LambdaMetafactory（Lambda元工厂）动态生成一个实现了目标函数式接口的类，并把这个类与之前生成的静态方法绑定，返回一个实例。

**后续调用**：第一次链接完成后缓存结果，后续直接用，不再重复生成类。

这种方式的好处是高效（按需生成，一次生成反复使用）且灵活（将来JVM可以用更好的方式实现，不需要改编译器）。跟匿名内部类相比，不用每次都生成一个新的类文件，也不需要每次都创建对象（对于无状态的Lambda可以复用同一个实例）。
