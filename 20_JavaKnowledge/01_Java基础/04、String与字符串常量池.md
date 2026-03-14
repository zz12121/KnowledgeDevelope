# String 与字符串常量池

> **核心关键词**：不可变性、字符串常量池、intern、StringBuilder、StringBuffer、编码

---

## 一、String 的不可变性

### 1.1 为什么不可变？

`String` 的底层是 `private final char[]`（JDK 8）或 `private final byte[]`（JDK 9+，引入 Compact Strings），并且没有提供任何修改该数组内容的方法。

```java
// JDK 8 源码
public final class String implements Serializable, Comparable<String>, CharSequence {
    private final char value[];   // final 数组引用不可变，数组内容也不对外暴露
    private int hash;             // 缓存的 hashCode，首次计算后不变
    // ...
}

// JDK 9+ 源码（紧凑字符串优化）
public final class String {
    private final byte[] value;  // Latin-1 字符用 1 字节，其他用 2 字节（UTF-16）
    private final byte coder;    // LATIN1=0 / UTF16=1
}
```

### 1.2 不可变的好处

1. **线程安全**：无需同步，多线程共享 String 对象完全安全
2. **可作为 HashMap key**：hashCode 只计算一次并缓存，查找高效且结果稳定
3. **字符串常量池成为可能**：相同内容的字符串可以安全共享同一对象
4. **安全性**：类名、文件路径、网络连接参数等如果可变，容易被篡改引发安全漏洞

### 1.3 "修改" String 的本质

```java
String s = "hello";
s = s + " world";  // 并没有修改原来的 "hello" 字符串对象！
                   // 而是创建了新的 "hello world" 对象，s 指向新对象
                   // 原 "hello" 对象如无其他引用，等待 GC
```

---

## 二、字符串常量池（String Pool / String Intern Pool）

### 2.1 存储位置变迁

| JDK 版本 | 常量池位置 |
|---------|-----------|
| JDK 6 及之前 | 永久代（PermGen）中 |
| JDK 7+ | **堆（Heap）** 中 |

> JDK 7 迁移到堆的原因：永久代空间有限，大量字符串 intern 容易 OOM；堆空间更大且可 GC。

### 2.2 字面量 vs new String()

```
String s1 = "hello";           ← 直接使用常量池中的对象
String s2 = "hello";           ← 同一对象（常量池复用）
String s3 = new String("hello"); ← 在堆上创建新对象（内部 value[] 指向常量池的 "hello"）

内存布局：
常量池：["hello" 对象]
堆：    [s3 对应的 String 对象] → value[] → 常量池中 "hello" 的 char 数组

s1 == s2  → true  （同一常量池对象）
s1 == s3  → false （s3 是堆上新对象）
s1.equals(s3) → true （值相同）
```

### 2.3 编译期常量折叠

```java
String s1 = "hello" + " world";  // 编译期直接合并为 "hello world"，进常量池
String s2 = "hello world";
System.out.println(s1 == s2);     // true

String part = "world";            // 变量，非 final
String s3 = "hello " + part;      // 运行期拼接，new StringBuilder().append...，在堆上
System.out.println(s1 == s3);     // false

final String part2 = "world";     // final 常量，编译期可知
String s4 = "hello " + part2;     // 编译期折叠，等价于 "hello world" 字面量
System.out.println(s1 == s4);     // true
```

### 2.4 intern() 方法

`intern()` 将字符串手动放入常量池（或返回常量池中已有的引用）：

```java
String s1 = new String("hello");  // 堆上新对象
String s2 = s1.intern();          // 返回常量池中的 "hello"
String s3 = "hello";              // 常量池中的 "hello"

System.out.println(s1 == s2);     // false（s1 在堆，s2 在常量池）
System.out.println(s2 == s3);     // true（同一常量池对象）
```

**intern() 的实际应用**：解析大量重复字符串时（如 CSV 中同一城市名重复出现），用 intern() 减少堆内存占用。但 JDK 7+ 常量池在堆中，过多 intern 同样消耗堆内存，需权衡。

---

## 三、String 常用 API 全览

### 3.1 查询类

```java
String s = "Hello, World!";

s.length()                // 13
s.charAt(7)               // 'W'
s.indexOf("World")        // 7
s.lastIndexOf('l')        // 10
s.contains("World")       // true
s.startsWith("Hello")     // true
s.endsWith("!")           // true
s.isEmpty()               // false
s.isBlank()               // false（JDK 11+，空或只含空白字符）
```

### 3.2 截取与分割

```java
s.substring(7)            // "World!"
s.substring(7, 12)        // "World"（含头不含尾）
s.split(", ")             // ["Hello", "World!"]
s.split(",", 2)           // 最多分2段：["Hello", " World!"]
```

### 3.3 变换类

```java
s.toLowerCase()                    // "hello, world!"
s.toUpperCase()                    // "HELLO, WORLD!"
s.trim()                           // 去除首尾半角空白
s.strip()                          // JDK 11+，去除首尾 Unicode 空白（更全面）
s.replace('l', 'r')               // "Herro, Worrd!"
s.replace("World", "Java")        // "Hello, Java!"
s.replaceAll("\\s+", "_")         // 正则替换
"  hello  ".stripLeading()        // "hello  "（JDK 11+）
"  hello  ".stripTrailing()       // "  hello"（JDK 11+）
```

### 3.4 转换类

```java
String.valueOf(123)               // "123"
String.valueOf(true)              // "true"
String.format("Hello, %s!", "Java") // "Hello, Java!"（JDK 15+：formatted() 等价）

char[] chars = s.toCharArray();   // 转 char 数组
byte[] bytes = s.getBytes(StandardCharsets.UTF_8); // 编码为字节数组

// JDK 11+ 新增
String.join("-", "a", "b", "c")  // "a-b-c"
"a,b,c".repeat(2)                // "a,b,ca,b,c"
"hello\nworld\n".lines()         // Stream<String>，按行分割
```

### 3.5 比较类

```java
s.equals("Hello, World!")        // true（值比较，区分大小写）
s.equalsIgnoreCase("hello, world!") // true（忽略大小写）
s.compareTo("Hello, World!")     // 0（字典序比较）
```

---

## 四、String、StringBuilder、StringBuffer 对比

### 4.1 核心区别

| 特性 | String | StringBuilder | StringBuffer |
|------|--------|---------------|--------------|
| 可变性 | **不可变** | **可变** | **可变** |
| 线程安全 | ✅ 安全（不可变）| ❌ 不安全 | ✅ 安全（synchronized）|
| 性能 | 拼接时最差 | **最快** | 比 StringBuilder 略慢 |
| 适用场景 | 不频繁修改 | 单线程字符串拼接 | 多线程字符串拼接 |

### 4.2 StringBuilder 内部原理

`StringBuilder` 内部维护一个 `char[]`，初始容量 **16**，扩容策略：`新容量 = 旧容量 * 2 + 2`

```java
StringBuilder sb = new StringBuilder();  // capacity=16
sb.append("Hello");          // length=5,  capacity=16（未超出）
sb.append(", World!");       // length=13, capacity=16（未超出）
sb.append(" Java is great"); // length=28, capacity=34（触发扩容：16*2+2=34）
```

**预分配容量减少扩容次数**：

```java
// 已知最终字符串大约500字符，预分配
StringBuilder sb = new StringBuilder(512);
```

### 4.3 循环拼接的性能对比

```java
// ❌ 性能差：每次 + 都创建新 String 对象
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i;  // 创建 10000 个临时 String 对象
}

// ✅ 高效：只有一个 StringBuilder 对象
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

> 注：编译器会自动将**非循环**的 `+` 拼接优化为 `StringBuilder`，但**循环内**的 `+` 无法优化（每次循环都 new 一个新 StringBuilder）。

### 4.4 StringBuilder 常用 API

```java
StringBuilder sb = new StringBuilder("Hello");
sb.append(", World");         // "Hello, World"
sb.insert(5, " Java");        // "Hello Java, World"
sb.delete(5, 10);             // "Hello, World"
sb.replace(7, 12, "Java");    // "Hello, Java"
sb.reverse();                 // "avaJ ,olleH"
sb.deleteCharAt(0);           // "vaJ ,olleH"
sb.toString();                // 转为 String
sb.length();                  // 当前长度
sb.capacity();                // 当前内部数组容量
```

---

## 五、字符编码

### 5.1 Java 内部编码

- JDK 8：内部统一使用 **UTF-16**（char 是 16 位，U+0000 ~ U+FFFF BMP 字符用 1 个 char，辅助平面字符用 2 个 char）
- JDK 9+：引入 **Compact Strings**，Latin-1（ISO-8859-1）字符用 1 字节存储，其余用 UTF-16，节省内存

### 5.2 String 与字节数组互转

```java
String s = "你好 Java";

// String → byte[]（指定编码）
byte[] utf8Bytes  = s.getBytes(StandardCharsets.UTF_8);   // "你好" 各占 3 字节
byte[] gbkBytes   = s.getBytes("GBK");                    // "你好" 各占 2 字节

// byte[] → String（必须用相同编码）
String restored = new String(utf8Bytes, StandardCharsets.UTF_8);
```

**乱码的根本原因**：编码与解码使用了不同的字符集。

---

## 六、实战场景与踩坑

### 场景1：字符串比较陷阱

```java
// ❌ 永远不要用 == 比较字符串值
String a = new String("test");
String b = new String("test");
if (a == b) { }        // 永远 false（两个不同堆对象）
if (a.equals(b)) { }   // ✅ 正确

// ✅ 常量放前面，避免 NPE
"expected".equals(userInput);  // userInput 为 null 也不会抛 NPE
userInput.equals("expected");  // userInput 为 null → NPE！
```

### 场景2：大量字符串拼接

```java
// 场景：构建 SQL 语句、HTML 内容等
// ✅ 推荐 StringJoiner（JDK 8）
StringJoiner sj = new StringJoiner(", ", "(", ")");
for (String col : columns) {
    sj.add(col);
}
String sql = "SELECT " + sj; // "(id, name, age)"

// ✅ 或 String.join
String cols = String.join(", ", "id", "name", "age"); // "id, name, age"

// ✅ 或流式 Collectors.joining
String result = list.stream().collect(Collectors.joining(", ", "[", "]"));
```

### 场景3：字符串分割的陷阱

```java
// split 的参数是正则表达式！
"a.b.c".split(".");    // ❌ "." 是正则通配符，结果是空数组
"a.b.c".split("\\.");  // ✅ 转义，结果：["a", "b", "c"]

"a|b|c".split("|");    // ❌ "|" 是正则或操作符，结果异常
"a|b|c".split("\\|"); // ✅ 正确
```

---

## 七、面试要点速查

| 问题 | 要点 |
|------|------|
| String 为什么不可变 | `private final char[]` + 无修改方法 + `final class` |
| `==` 和 `equals` 的区别 | `==` 比较引用地址，`equals` 比较字符串内容 |
| 字符串常量池在哪里 | JDK 7+ 在堆中（之前在 PermGen）|
| `new String("abc")` 创建了几个对象 | 最多2个：常量池中的 "abc"（如果没有）+ 堆上的 String 对象 |
| `intern()` 的作用 | 将字符串放入常量池并返回常量池引用，用于节省内存 |
| StringBuilder vs StringBuffer | 前者单线程高性能，后者线程安全（方法加 synchronized）|
| 为什么 String 可以做 HashMap key | 不可变 → hashCode 稳定且可缓存，天然线程安全 |
| JDK 9 的 Compact Strings | Latin-1 字符用 byte 存储，节省约一半内存 |


---

**相关面试题** → [[../../10_Developlanguage/001_java/01_JavaBaseSubject/05、String 与常用类|05、String 与常用类]]
