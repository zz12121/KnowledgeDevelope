# SPI 机制

> **核心关键词**：Service Provider Interface、ServiceLoader、META-INF/services、Java SPI vs Spring SPI vs Dubbo SPI、扩展点

---

## 一、SPI 是什么

**SPI（Service Provider Interface）**：一种服务发现机制，允许第三方为某个接口提供实现，框架通过约定的配置文件自动发现并加载这些实现，实现**框架与实现的解耦**。

```
核心思想：面向接口编程 + 运行时动态发现实现

经典类比：
  JDBC：java.sql.Driver 接口（JDK 定义）
        MySQL、PostgreSQL、Oracle 各自提供实现
        应用代码只写 Class.forName("com.mysql.jdbc.Driver")
        或者让 SPI 自动加载，完全不感知底层实现
```

---

## 二、Java 原生 SPI

### 2.1 使用步骤

```
1. 定义接口（在框架模块）
2. 实现接口（在实现模块）
3. 在实现模块的 META-INF/services/ 下创建配置文件
   文件名 = 接口全限定名
   文件内容 = 实现类全限定名（每行一个）
4. 通过 ServiceLoader 加载
```

```java
// Step 1：定义接口
package com.example.spi;
public interface MessageFormatter {
    String format(String message);
}

// Step 2：实现
package com.example.impl;
public class JsonFormatter implements MessageFormatter {
    @Override
    public String format(String message) {
        return "{\"msg\": \"" + message + "\"}";
    }
}

// Step 3：创建配置文件
// 文件路径：resources/META-INF/services/com.example.spi.MessageFormatter
// 文件内容：
// com.example.impl.JsonFormatter
// com.example.impl.XmlFormatter

// Step 4：加载并使用
ServiceLoader<MessageFormatter> loader = 
    ServiceLoader.load(MessageFormatter.class);
for (MessageFormatter formatter : loader) {
    System.out.println(formatter.format("Hello"));
}
// 或者取第一个
MessageFormatter formatter = loader.iterator().next();
```

### 2.2 Java SPI 的缺陷

```
1. 只能加载所有实现，无法按需加载指定实现
2. 不支持扩展点参数化（无法传参选择具体实现）
3. 线程安全问题（ServiceLoader 本身不是线程安全的）
4. 懒加载但实例化耦合：iterator() 时才实例化，但无法按名字获取
5. 无依赖注入支持（无法自动注入其他 Bean）
```

---

## 三、JDBC 中的 SPI 经典案例

```java
// JDBC 4.0 之前需要手动加载驱动
Class.forName("com.mysql.jdbc.Driver");

// JDBC 4.0 之后（利用 SPI 自动加载）
// mysql-connector-java.jar 中包含：
// META-INF/services/java.sql.Driver
// 内容：com.mysql.cj.jdbc.Driver

// DriverManager 在静态初始化块中调用 ServiceLoader 自动发现并注册驱动
Connection conn = DriverManager.getConnection(url, user, password);
// 无需 Class.forName，驱动已被 SPI 自动加载
```

---

## 四、Spring SPI（Spring Factories）

Spring 有自己的 SPI 机制，配置文件路径不同：

```
配置文件路径：META-INF/spring.factories（Spring Boot 2.x）
              META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
              （Spring Boot 3.x，改为独立文件）
```

```properties
# META-INF/spring.factories 示例
# 格式：接口全限定名=实现类1,实现类2（逗号分隔）
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.MyAutoConfiguration,\
  com.example.AnotherAutoConfiguration

org.springframework.context.ApplicationListener=\
  com.example.MyApplicationListener
```

```java
// Spring Boot 自动配置的核心就是 SPI
// @EnableAutoConfiguration → SpringFactoriesLoader.loadFactoryNames()
// 从所有 jar 包的 META-INF/spring.factories 中加载自动配置类
```

---

## 五、Dubbo SPI（增强版 SPI）

Dubbo 对 Java SPI 进行了大幅增强，是目前功能最完善的 SPI 实现：

```
配置文件路径：
  META-INF/dubbo/          （用户自定义扩展）
  META-INF/dubbo/internal/ （Dubbo 内置扩展）
  META-INF/services/       （兼容 Java SPI）
```

```properties
# 配置文件格式：key=value（支持按名字获取，这是 Java SPI 没有的！）
# 文件名：org.apache.dubbo.rpc.Protocol
dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
http=org.apache.dubbo.rpc.protocol.http.HttpProtocol
grpc=org.apache.dubbo.rpc.protocol.grpc.GrpcProtocol
```

```java
// Dubbo SPI 核心特性

// 1. 按名字获取（Java SPI 不支持）
@SPI("dubbo")  // 默认扩展名
public interface Protocol {
    int getDefaultPort();
    // ...
}

ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);
Protocol dubboProtocol = loader.getExtension("dubbo");  // 按名字获取

// 2. 自适应扩展（@Adaptive）—— 运行时根据 URL 参数动态选择实现
@Adaptive
Protocol adaptiveProtocol = loader.getAdaptiveExtension();
// 调用时根据 URL 中的 protocol 参数动态路由到对应实现

// 3. 自动激活（@Activate）—— 满足条件时自动激活
// 例如：Filter 在消费者/提供者端分别激活不同的实现

// 4. 依赖注入——通过 setter 注入其他扩展
public class DubboProtocol implements Protocol {
    private ProxyFactory proxyFactory;
    // Dubbo SPI 自动注入 proxyFactory（通过 setter）
    public void setProxyFactory(ProxyFactory proxyFactory) {
        this.proxyFactory = proxyFactory;
    }
}

// 5. AOP 包装（Wrapper 模式）
// 实现类名包含 Wrapper 的会被自动识别为包装类
// 多个 Wrapper 形成调用链（类似 AOP）
```

---

## 六、三种 SPI 对比

| 特性 | Java SPI | Spring SPI | Dubbo SPI |
|------|----------|-----------|-----------|
| 配置文件 | META-INF/services/接口名 | META-INF/spring.factories | META-INF/dubbo/接口名 |
| 按名字获取 | ❌ | ❌ | ✅ |
| 懒加载 | 部分 | ❌ | ✅ |
| 依赖注入 | ❌ | ✅（Spring 容器） | ✅（Setter 注入） |
| AOP 支持 | ❌ | ✅ | ✅（Wrapper） |
| 自适应扩展 | ❌ | ❌ | ✅（@Adaptive） |
| 条件激活 | ❌ | 部分 | ✅（@Activate） |

---

## 七、SPI 与 API 的区别

```
API（Application Programming Interface）：
  框架/库提供接口和实现，调用者使用
  接口和实现都在框架内部
  方向：框架 → 调用者（你调用框架）

SPI（Service Provider Interface）：
  框架定义接口，调用者提供实现
  接口在框架，实现在调用者
  方向：调用者 → 框架（框架调用你的实现）

例子：
  JDBC API：你调用 Connection、Statement（API）
  JDBC SPI：MySQL 实现 Driver 接口供 DriverManager 调用（SPI）
```

---

## 八、面试要点速查

| 问题 | 要点 |
|------|------|
| SPI 的作用 | 解耦框架与实现，通过配置文件发现并加载第三方实现 |
| Java SPI 缺陷 | 不支持按名字获取、不能按需加载、无依赖注入、非线程安全 |
| JDBC 如何用 SPI | `META-INF/services/java.sql.Driver` 配置驱动类，DriverManager 自动加载 |
| Spring SPI 配置文件 | `META-INF/spring.factories`（Boot 2.x）/ AutoConfiguration.imports（Boot 3.x） |
| Dubbo SPI 相比 Java SPI 增强了什么 | 按名字获取 + 依赖注入 + Wrapper AOP + @Adaptive 自适应 + @Activate 条件激活 |
| @Adaptive 的原理 | Dubbo 在运行时用 Javassist/ASM 动态生成代理类，代理类从 URL 参数中取扩展名，再调用对应实现 |
| Wrapper 包装类是什么 | Dubbo SPI 中类名含 Wrapper 的会被识别为包装类，自动形成调用链，实现 AOP |
| SPI 和 API 的区别 | API 是你调框架；SPI 是框架调你的实现 |


---

**相关面试题** → [[../../10_Developlanguage/001_java/01_JavaBaseSubject/15、SPI 机制|15、SPI 机制]]
