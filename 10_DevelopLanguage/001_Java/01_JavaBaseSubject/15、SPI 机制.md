###### 1. 什么是 SPI？

[[../../../20_JavaKnowledge/01_Java基础/12、SPI机制#一、SPI 是什么|📖]]
SPI（Service Provider Interface，服务提供者接口）是一种**服务发现机制**，核心目标是解耦。

原理是这样的：框架或标准层定义好接口规范（SPI），具体实现由第三方服务提供者来完成。程序运行时，框架通过`ServiceLoader`动态发现并加载这些实现，不需要在代码里硬编码具体实现类。

最典型的例子是JDBC：`java.sql.Driver`是SPI接口，MySQL、PostgreSQL等数据库厂商提供各自的驱动实现，应用程序不依赖具体的数据库驱动类，换数据库只需要换JAR包就行。

###### 2. SPI 和 API 的区别是什么？

[[../../../20_JavaKnowledge/01_Java基础/12、SPI机制#七、SPI 与 API 的区别|📖]]
用一句话说清楚：**API是"我提供功能，你来调用"；SPI是"你来提供实现，我来调用"**。

**API（Application Programming Interface）**：实现方定义接口并提供实现，调用方直接用接口功能。控制权在调用方，类似于"服务商给你工具，你来用"。

**SPI（Service Provider Interface）**：调用方（框架）定义接口规范，实现方（第三方）来提供具体实现，框架在运行时选择使用哪个实现。控制权在框架，是一种**回调思想**，"框架定规范，你来适配我"。

JDBC的例子：`DriverManager.getConnection()`是API（应用开发者调用）；`java.sql.Driver`是SPI（数据库厂商实现）。

###### 3. SPI 的实现原理是什么？

[[../../../20_JavaKnowledge/01_Java基础/12、SPI机制#二、Java 原生 SPI|📖]]
Java SPI的标准实现靠的是`java.util.ServiceLoader`类，遵循三个要素：

**SPI接口**：框架定义的服务规范，标准Java接口。

**实现类**：第三方JAR包提供的具体实现，实现了SPI接口。

**配置文件**：在JAR包的`/META-INF/services/`目录下，创建一个以接口全限定名命名的文件，文件内容是实现类的全限定名，每行一个。

运行时流程：调用`ServiceLoader.load(Interface.class)` → 扫描classpath里所有JAR包的`/META-INF/services/`目录 → 找到对应接口的配置文件 → 读取配置里的实现类名 → 通过反射加载并实例化这些类。

```java
ServiceLoader<Animal> loader = ServiceLoader.load(Animal.class);
for (Animal animal : loader) {
    animal.sound(); // 遍历所有发现的实现
}
```

**优点**：解耦彻底，加新实现只需要加JAR包，不改原来代码，符合开闭原则。

**缺点**：`ServiceLoader`是全量加载，把配置文件里所有实现类都实例化，即使用不上也会占内存；不能按需获取某个特定实现（只能遍历），灵活性不足。Dubbo、Spring等框架对SPI做了增强，支持按名称获取指定实现，解决了标准SPI的局限性。
