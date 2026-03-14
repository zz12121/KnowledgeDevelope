###### 1. String、StringBuffer 和 StringBuilder？

[[../../../20_JavaKnowledge/01_Java基础/04、String与字符串常量池#四、String、StringBuilder、StringBuffer 对比|📖]]
这三个都是处理字符串的，但有重要区别：

**String**：不可变，每次看似修改都是创建新对象。线程安全（因为不可变）。适合字符串内容不变或改动很少的场景，比如字符串常量。

**StringBuffer**：可变，方法都加了`synchronized`，线程安全。适合多线程环境下频繁修改字符串。但加锁有开销，性能比StringBuilder差。

**StringBuilder**：可变，无同步，线程不安全。单线程下性能最好。**实际开发里99%的字符串拼接场景都用这个**。

**选择规则**：内容不会改就用String；单线程大量拼接用StringBuilder；多线程大量拼接用StringBuffer。在Java 8之后，编译器会把`"a" + "b" + "c"`这样的字符串拼接自动优化成StringBuilder，所以简单的`+`操作不用手动改。

###### 2. String s = new String ("abc") 创建了几个 String 对象？

[[../../../20_JavaKnowledge/01_Java基础/04、String与字符串常量池#二、字符串常量池（String Pool）|📖]]
取决于字符串常量池里有没有"abc"：

**如果常量池里已经有"abc"**：只创建1个对象，就是堆里`new`出来的那个。

**如果常量池里还没有"abc"**：创建2个对象——先在常量池里建一个"abc"，再在堆里建`new String()`那个对象。变量`s`指向堆里的那个。

实际面试中，如果没有特别说明，通常答"最多2个，至少1个"，然后解释清楚这个条件就行了。

注意：`new String("abc")`和直接`"abc"`的区别是，直接赋值`String s = "abc"`拿的就是常量池里的那个，而`new`出来的在堆里是个独立的对象。

###### 3. 说说你对 String 类的 intern () 方法的理解？

[[../../../20_JavaKnowledge/01_Java基础/04、String与字符串常量池#二、字符串常量池（String Pool）|📖]]
`intern()`的作用是：**把字符串放进常量池，并返回常量池中对应字符串的引用**。

JDK 7+的行为（和JDK 6不同）：调用`intern()`时，如果常量池里已有内容相同的字符串，直接返回池里那个引用；如果没有，不会复制一份到常量池，而是在常量池里**记录一个指向堆中该对象的引用**，然后返回这个引用。

```java
String s1 = new String("a") + new String("a"); // s1在堆里，内容"aa"，常量池还没有"aa"
s1.intern(); // 把s1的引用记录到常量池
String s2 = "aa"; // 直接拿常量池里的，就是s1那个
System.out.println(s1 == s2); // true，同一个对象
```

主要用途是**节省内存**：如果程序中有大量重复内容的字符串，通过intern池化可以只保留一份，减少内存占用。但不宜滥用，要根据实际场景判断。

###### 4. String 类的常用方法都有那些？

[[../../../20_JavaKnowledge/01_Java基础/04、String与字符串常量池#三、String 常用 API 全览|📖]]
按功能分类记忆：

**获取信息**：`length()`获取长度，`isEmpty()`/`isBlank()`判断空（isBlank还检查空白符），`charAt(index)`获取指定位置的字符。

**比较判断**：`equals()`比较内容，`equalsIgnoreCase()`忽略大小写比较，`compareTo()`字典顺序比较，`startsWith()`/`endsWith()`判断前后缀，`contains()`判断是否包含。

**操作子串**：`substring(begin, end)`截取子串，`concat()`拼接。

**查找索引**：`indexOf()`找第一次出现的位置，`lastIndexOf()`找最后一次出现的位置。

**修改替换**：`replace(old, new)`替换，`toLowerCase()`/`toUpperCase()`转大小写，`trim()`去首尾空格，`strip()`（Java 11+）去首尾空白符（包括Unicode空白符）。

**切割转换**：`split(regex)`按正则切割，`toCharArray()`转字符数组，`valueOf()`把其他类型转成String。

###### 5. String 字符串为什么说是不可变？

[[../../../20_JavaKnowledge/01_Java基础/04、String与字符串常量池#一、String 的本质|📖]]
String不可变是由三个层面保证的：

**final修饰的类**：String不能被继承，从根本上防止子类破坏不可变性。

**private final的底层数组**：实际存数据的是`private final byte[] value`（JDK 9+，之前是`char[]`）。`final`保证这个引用不能改指向，`private`保证外部无法直接操作这个数组。

**没有修改方法**：String对外没有提供任何修改底层数组内容的方法，所有看似修改的操作（拼接、替换）都是返回新对象。

不可变的好处很多：可以安全地共享（字符串常量池的基础）；hashCode稳定，适合作HashMap的key；天然线程安全；用作网络连接、文件路径等安全参数不会被意外修改。

###### 6. Object 有哪些常用方法？大致说一下每个方法的含义？

[[../../../20_JavaKnowledge/01_Java基础/02、面向对象三大特性#一、封装（Encapsulation）|📖]]
Object是所有类的根类，它的常用方法：

`equals(Object obj)`：逻辑相等判断，默认行为是比较地址（和`==`一样），大多数类会重写来比较内容。

`hashCode()`：返回对象哈希码，用于哈希表快速定位。**重写equals必须同时重写hashCode**，这是铁律。

`toString()`：对象的字符串表示，建议每个类都重写，方便调试和日志。

`getClass()`：获取运行时Class对象，用于反射。

`clone()`：对象克隆，需要实现Cloneable接口，默认是浅拷贝。

`finalize()`：垃圾回收前调用，**已废弃**，不要用，因为执行时机不可控。

`wait()`、`notify()`、`notifyAll()`：线程间协调用的，必须在synchronized块里调用。wait让当前线程等待并释放锁，notify唤醒等待的一个线程，notifyAll唤醒所有等待的线程。

###### 7. 一个空 Object 对象的占多大空间？

[[../../../20_JavaKnowledge/04_JVM/04、对象的创建与内存布局#二、对象的内存布局|📖]]
HotSpot VM里，一个空Object对象占**16字节**。

组成：对象头Mark Word占8字节（存哈希码、GC年龄、锁状态等），类型指针占8字节（64位JVM开启指针压缩后是4字节）。这样是12字节，然后JVM要求对象必须是8字节对齐，所以填充到16字节。

开启指针压缩（默认开启）时，对象头= MarkWord(8字节) + 压缩类指针(4字节) = 12字节，对齐补4字节 = 16字节。

注意区分：对象本身16字节是堆上的开销，你持有这个对象的引用变量本身在64位JVM上占8字节（在栈或堆里）。

###### 8. 说说你对 equals 与 == 的理解？

[[../../../20_JavaKnowledge/01_Java基础/02、面向对象三大特性#六、设计原则与实战场景|📖]]
`==`比较的是**值本身**：对于基本类型，就是数值是否相等；对于引用类型，就是两个引用是否指向同一个对象（比较内存地址）。

`equals()`是一个方法，Object默认实现和`==`一样（比地址），但大多数类会重写它来比较**逻辑上是否相等**（比较内容）。比如String的`equals()`比较的是字符序列是否相同。

```java
String s1 = new String("hello");
String s2 = new String("hello");
System.out.println(s1 == s2);        // false，不同对象
System.out.println(s1.equals(s2));   // true，内容相同
```

**结论**：比较基本类型用`==`；比较对象的内容/逻辑相等用`equals()`；比较对象是否是同一个实例用`==`。日常开发中字符串比较永远用equals。

###### 9. 说说 Hash Code 的作用？

[[../../../20_JavaKnowledge/01_Java基础/04、String与字符串常量池#一、String 的本质|📖]]
哈希码是对象导出的一个整型值，最重要的用途是：**让哈希表（HashMap、HashSet）能快速定位元素**。

哈希表用哈希码来确定存储桶的位置，平均情况下时间复杂度O(1)，不用挨个比较。没有哈希码这个机制，每次查找都要遍历所有元素，复杂度O(n)。

重要规范：**equals相同的对象，hashCode必须相同**；反过来hashCode相同的对象，equals不一定相同（这叫哈希冲突，是允许的）。所以重写equals必须同时重写hashCode，否则放进HashMap/HashSet会出现找不到的问题。

###### 10. 有没有可能两个不相等的对象有相同的 Hash Code？

[[../../../20_JavaKnowledge/01_Java基础/04、String与字符串常量池#一、String 的本质|📖]]
**有可能，这叫哈希冲突（Hash Collision）**，是正常现象。

哈希码是int类型，只有2^32种可能；但对象的状态组合是无限的。所以必然存在不同对象映射到同一哈希码的情况，这是鸽巢原理决定的，无法避免。

哈希表（如HashMap）用链地址法（数组+链表/红黑树）来解决冲突：同一个桶里的多个元素放链表/树，查找时先hash定桶，再用equals逐个比较。

好的哈希函数应该尽量让不同对象分布均匀，减少冲突，提升性能。

###### 11. 两个对象值相同 (x.equals(y) == true)，但却可有不同的 Hash Code，这句话对不对？

[[../../../20_JavaKnowledge/01_Java基础/04、String与字符串常量池#一、String 的本质|📖]]
**不对，这句话是错误的**。

这直接违反了hashCode和equals的合约：**如果x.equals(y)为true，那么x.hashCode()必须等于y.hashCode()**，这是Java语言规范强制要求的。

反过来可以：hashCode相同的两个对象，equals不一定为true（这是哈希冲突）。

如果违反这个合约，后果很严重：把对象放进HashMap后找不到，HashSet也会出现重复元素等问题。所以重写equals一定要同时重写hashCode，这是铁律，任何Java IDE都会提示你这个。

###### 12. StringBuilder 和 StringBuffer 的底层实现原理是什么？

[[../../../20_JavaKnowledge/01_Java基础/04、String与字符串常量池#四、String、StringBuilder、StringBuffer 对比|📖]]
两者都继承自`AbstractStringBuilder`，底层原理基本一样：

内部维护一个**可变的字符数组**（`char[] value`，没有final修饰，所以可以修改），这就是和String的根本区别。

**动态扩容**：追加内容时，如果数组容量不够，就创建一个更大的新数组（一般是原来的2倍+2），把旧内容复制进去，再继续追加。机制类似ArrayList。所以如果能预估最终大小，用`new StringBuilder(initialCapacity)`指定初始容量，减少扩容次数，性能更好。

**线程安全区别**：StringBuffer的方法加了`synchronized`，StringBuider没有。一般业务里用StringBuilder就够了，线程安全交给上层调用方去保证。

###### 13. String 为什么要设计成不可变的？

[[../../../20_JavaKnowledge/01_Java基础/04、String与字符串常量池#一、String 的本质|📖]]
这是个设计选择题，好处有几点：

**字符串常量池的基础**：不可变才能安全共享，多个变量可以指向同一个常量池对象，节省内存。

**线程安全**：天然线程安全，可以在多线程间自由共享，无需额外同步开销。

**hashCode缓存**：String内部缓存了第一次计算的hashCode，后续直接返回，不用重算。这让String作HashMap的key性能非常好。

**安全性**：网络连接、文件路径、类名等关键信息用String传递，不可变防止被中途篡改，避免安全漏洞。

这些好处加在一起，让String成为Java里使用最频繁的类，设计成不可变是非常正确的决策。
