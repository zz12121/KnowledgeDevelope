###### 1. 知道 Dubbo SPI 机制吗？
是的，Dubbo SPI（Service Provider Interface）是Dubbo框架的**扩展点加载机制**，也是Dubbo**微内核和高度可扩展性的基石**。它是在Java标准SPI思想基础上的全面增强和重构。
**核心概念**：
- **扩展点（Extension Point）**：一个Java接口，定义了某个可扩展的功能。例如`Protocol`、`LoadBalance`、`Cluster`。
- **扩展（Extension）**：扩展点的具体实现类。例如`DubboProtocol`、`RandomLoadBalance`、`FailoverCluster`。
- **扩展点加载器（ExtensionLoader）**：Dubbo SPI的核心类，负责加载和管理扩展点及其实现。
**与Java SPI对比**：
- **Java SPI**：在`META-INF/services/`目录下以接口全限定名为文件，文件内容为具体实现类全限定名。通过`ServiceLoader`加载，不支持按需加载、不支持注解、不支持依赖注入。
- **Dubbo SPI**：
    1. 有自己的配置文件目录：`META-INF/dubbo/`、`META-INF/dubbo/internal/`等。
    2. 配置文件内容为`key=实现类全限定名`，支持别名。
    3. 支持**自适应扩展点**（`@Adaptive`）。
    4. 支持**自动激活扩展点**（`@Activate`）。
    5. 支持**扩展点依赖注入**（IoC）。
    6. 支持**包装类装饰**（AOP）。
**源码入口**：`ExtensionLoader`是SPI机制的核心类。其`getExtension(String name)`方法根据扩展名获取扩展实例；`getAdaptiveExtension()`获取自适应扩展实例；`getActivateExtension(URL url, String key)`获取自动激活的扩展列表。
###### 2. Dubbo 为什么不采用 JDK 的 SPI 机制？而是要自己实现?
Dubbo放弃JDK标准SPI，选择自研SPI机制，主要基于以下**四大不足**的考量：
1. **不支持按需加载与缓存**：
    - **JDK SPI**：`ServiceLoader`在加载时，会一次性加载并实例化`META-INF/services/`下配置的所有实现类。如果某些扩展实现初始化耗时或资源消耗大，但当前并不需要，会造成资源浪费。
    - **Dubbo SPI**：采用`缓存+懒加载`。`ExtensionLoader`会缓存已加载的扩展类定义和实例（分`Class`缓存和实例缓存）。只有在调用`getExtension(name)`时，才会加载对应名称的实现类并实例化。源码中，`cachedClasses`缓存扩展名与扩展类的映射，`EXTENSION_INSTANCES`缓存扩展类与实例的映射。
2. **不支持扩展点依赖注入与生命周期管理**：
    - **JDK SPI**：仅仅通过无参构造器实例化对象，无法处理扩展点之间的依赖关系。
    - **Dubbo SPI**：支持**Setter依赖注入**。如果一个扩展点实现类的成员变量被`@Inject`注解（早期版本）或是一个setter方法，且该变量的类型也是一个扩展点，Dubbo会自动注入对应的扩展实现。源码中，`injectExtension()`方法负责此逻辑，它遍历所有setter方法，通过`objectFactory.getExtension()`获取依赖对象并注入。`objectFactory`本身也是一个自适应扩展点（`AdaptiveExtensionFactory`），可以适配多个IoC容器（如Spring）。
3. **缺乏扩展点激活与条件装配机制**：
    - **JDK SPI**：只能获取所有实现，无法根据上下文条件选择性地激活部分扩展。
    - **Dubbo SPI**：通过`@Activate`注解实现条件化自动激活。可以根据URL中的参数（`group`, `value`）来过滤和排序扩展点。例如，Filter链的构建就是通过`getActivateExtension()`方法，根据消费者或提供者身份以及URL参数，动态组装激活的Filter列表。
4. **不支持AOP和Wrapper装饰**：
    - **JDK SPI**：返回的是原始的扩展实例。
    - **Dubbo SPI**：如果扩展点的实现类构造器参数是该扩展点接口类型，则该类被视为**Wrapper类**（装饰器）。Dubbo在加载扩展点时，会自动用这些Wrapper层层包装原始扩展点实例，实现AOP功能。例如，`ProtocolFilterWrapper`和`ProtocolListenerWrapper`就是`Protocol`的Wrapper，它们为`Protocol`实例添加了过滤器和监听器链。源码中，`createExtension()`方法在实例化扩展对象后，会调用`injectExtension()`进行依赖注入，然后遍历所有Wrapper类，通过反射构造器进行包装。
**总结**：Dubbo SPI通过缓存、依赖注入、条件激活、Wrapper装饰等机制，提供了一个功能完整、性能优良、灵活性极高的微内核扩展框架，完美支撑了Dubbo各个层面的可扩展性。
###### 3. Dubbo SPI 的实现原理是什么？
Dubbo SPI的实现围绕`ExtensionLoader`这个核心类展开，其原理可分为**加载、缓存、实例化、注入、包装**几个阶段。
**1. 加载与解析配置文件**：
- 当通过`ExtensionLoader.getExtensionLoader(Class<T> type)`获取某个扩展点类型的加载器时，会首先检查缓存。如果没有，则创建新的`ExtensionLoader`实例。
- 加载器会从多个路径读取配置文件：`META-INF/dubbo/internal/`、`META-INF/dubbo/`、`META-INF/services/`。优先级依次降低。
- 配置文件名为扩展点接口的全限定名，内容为`key=实现类全限定名`。
- 加载器解析文件，将`key`和对应的`Class`对象存入`Map<String, Class<?>> cachedClasses`缓存。源码见`loadExtensionClasses()`和`loadDirectory()`方法。
**2. 获取扩展实例**：
- 调用`getExtension(String name)`获取扩展实例。
- 首先检查实例缓存`ConcurrentMap<String, Object> cachedInstances`。如果存在，直接返回。
- 如果不存在，则进入同步代码块，进行**双重检查**。
- 通过`cachedClasses`获取该`name`对应的扩展类`Class`对象。
- 调用`createExtension(String name)`创建实例。
**3. 创建扩展实例**：
- `createExtension(String name)`是核心方法：
    a. **反射创建实例**：`clazz.newInstance()`。
    b. **依赖注入**：调用`injectExtension(instance)`。遍历所有setter方法，如果方法以`set`开头，且参数只有一个，并且该参数类型是扩展点接口，则通过`objectFactory.getExtension()`获取依赖的扩展实例并注入。`objectFactory`本身是`ExtensionFactory`的自适应扩展，可以查找Spring等容器中的Bean。
    c. **Wrapper包装**：检查`cachedClasses`中所有类，如果某个类的构造器只有一个参数，且参数类型就是当前扩展点接口，则该类是一个Wrapper。用这个Wrapper类包装当前实例（`wrapperClass.getConstructor(type).newInstance(instance)`）。Wrapper可以有多个，层层包装，形成责任链。例如，对于`Protocol`，`ProtocolFilterWrapper`和`ProtocolListenerWrapper`会依次包装原始实例。
**4. 自适应扩展点**：
- 对于标注了`@Adaptive`的类，它是该扩展点的默认自适应实现。Dubbo会为每个扩展点接口动态生成一个自适应类（如果不存在默认的`@Adaptive`类）。这个类是一个代理，会根据URL中的参数动态选择使用哪个具体扩展。生成逻辑在`createAdaptiveExtensionClass()`中，使用字符串拼接生成Java源码，然后编译加载。
**5. 自动激活扩展点**：
- 通过`getActivateExtension(URL url, String key)`获取。该方法会根据URL中的参数（如`group`消费者/提供者，以及`key`对应的值）来过滤所有标注了`@Activate`的扩展实现，并进行排序（通过`@Activate`的`order`属性或`org.apache.dubbo.common.extension.ActivateComparator`）。
**缓存贯穿始终**：`ExtensionLoader`大量使用缓存（`ConcurrentMap`）来存储扩展类定义、实例、自适应实例等，确保高性能。
###### 4. Dubbo SPI 的 Adaptive 注解是做什么的？
`@Adaptive`注解用于实现**自适应扩展点**，它是Dubbo SPI**动态适配能力**的核心。它的作用是：**在运行时，根据方法参数中的`URL`（或可从`URL`推导出的信息）动态决定使用哪个具体的扩展实现。**
**两种用法**：
1. **标注在类上**：表示该类是该扩展点的**默认自适应实现**。Dubbo框架中只有少数几个类如此标注，例如`AdaptiveExtensionFactory`。当没有在URL中找到指定的扩展名时，会使用这个默认实现。
2. **标注在接口方法上**（更常见）：表示需要为该接口生成一个**动态自适应代理类**。Dubbo会为该方法生成代理逻辑，根据传入的URL参数选择具体的扩展。
**动态生成代理类的原理**：
- 以`Protocol`接口为例，其上有`@Adaptive`注解的方法`export`和`refer`。
- `ExtensionLoader`的`getAdaptiveExtension()`方法会检查是否有类标注了`@Adaptive`。如果没有，则会调用`createAdaptiveExtensionClass()`动态生成一个Java类。
- 生成的类名通常为`接口名$Adaptive`（例如`Protocol$Adaptive`）。
- 该类实现目标接口，并在每个标注了`@Adaptive`的方法中，编写模板代码：从`URL`中获取扩展名（如`protocol`），然后通过`ExtensionLoader.getExtension(extName)`加载对应的具体扩展实例，并委托调用。
**示例：`Protocol$Adaptive`的生成逻辑**：
```java
public class Protocol$Adaptive implements Protocol {
    public Exporter export(Invoker invoker) throws RpcException {
        URL url = invoker.getUrl();
        // 从url中获取protocol参数，如果没有，则使用默认值"dubbo"
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        // 根据extName加载具体的Protocol实现，如DubboProtocol, HttpProtocol
        Protocol extension = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName);
        return extension.export(invoker);
    }
    // refer方法类似
}
```
**URL是灵魂**：自适应机制严重依赖`URL`参数。`@Adaptive`注解可以指定从URL的哪个`key`获取扩展名，例如`@Adaptive("loadbalance")`会从URL的`loadbalance`参数取值。如果未指定，则根据类名转换（如`LoadBalance`-> `load.balance`-> `loadbalance`）。
**源码位置**：`ExtensionLoader.createAdaptiveExtensionClass()`方法拼接Java源码，然后使用`Compiler`扩展（默认`JavassistCompiler`）编译成Class并加载。
###### 5. Dubbo SPI 的 Activate 注解是做什么的？
`@Activate`注解用于实现**扩展点的条件化自动激活**。它允许框架根据**调用上下文**（主要是`URL`中的参数）**批量、有序地**加载和启用一组扩展实现。最常见的应用场景是**Filter链**的自动组装。
**注解属性**：
- `group()`：指定扩展点生效的组别。常用值有`Constants.PROVIDER`（服务提供者端）和`Constants.CONSUMER`（服务消费者端）。还可以自定义。
- `value()`：指定触发条件。它是一个数组，只有当URL中包含**指定的key**（无论value为何值）或者**指定的key且其value在注解的value列表中**时，该扩展才会被激活。例如`@Activate(value = "cache")`表示URL中有`cache`参数时激活；`@Activate(value = {"cache", "validation"}, group = Constants.PROVIDER)`表示在提供者端，URL有`cache`或`validation`参数时激活。
- `order()`：排序值，数字越小越靠前。用于确定多个被激活的扩展点的执行顺序。
**工作原理**：
- 调用`ExtensionLoader.getActivateExtension(URL url, String key)`方法。
- `key`参数通常用于指定要显式启用的扩展名（通过URL参数，如`-filter`指定，或默认值）。
- 该方法内部逻辑：
    1. 加载所有该扩展点的实现类，并缓存带有`@Activate`注解的类。
    2. 根据`group`和当前上下文（是Provider还是Consumer）进行第一轮筛选。
    3. 根据URL中的参数和`@Activate`的`value`进行第二轮筛选。
    4. 处理显式通过`key`指定的扩展（通过URL参数`-xxx`或`default`等）。
    5. 合并列表，并按`order`排序，去除重复。
- 最终返回一个有序的、符合条件的扩展实例列表。
**示例：Filter链的构建**：
在`ProtocolFilterWrapper.buildInvokerChain()`中，会调用`ExtensionLoader.getActivateExtension()`来获取所有自动激活的Filter，例如`MonitorFilter`、`TimeoutFilter`等。URL中可能包含`accesslog`参数，这会激活`AccessLogFilter`。最终，这些Filter被组装成一个调用链。
**源码关键**：`ExtensionLoader.getActivateExtension()`方法。它维护了`cachedActivates`缓存（存储扩展名与`@Activate`注解信息的映射），并根据复杂的逻辑进行筛选和排序。
###### 6. 如何自定义 Dubbo 的扩展点？
自定义Dubbo扩展点需要遵循Dubbo SPI的规范，以下是详细步骤：
**步骤一：定义扩展点接口**
扩展点必须是一个接口，并用`@SPI`注解声明。`@SPI`注解可以指定默认的扩展实现名。
```java
package com.example;
import org.apache.dubbo.common.extension.SPI;
@SPI("defaultImpl") // 指定默认实现别名
public interface MyFilter {
    String filter(String input);
}
```
**步骤二：实现扩展点**
编写一个或多个实现类。可以使用`@Adaptive`（如果需要自适应）、`@Activate`（如果需要自动激活）。
```java
package com.example.impl;
import com.example.MyFilter;
import org.apache.dubbo.common.extension.Activate;
@Activate(group = {Constants.PROVIDER, Constants.CONSUMER}, order = 100)
public class MyFilterImpl implements MyFilter {
    @Override
    public String filter(String input) {
        return "processed: " + input;
    }
}
```
**步骤三：创建SPI配置文件**
在项目的`resources`目录下创建文件：`META-INF/dubbo/com.example.MyFilter`。文件内容为`key=实现类全限定名`。
复制
```
myFilter=com.example.impl.MyFilterImpl
defaultImpl=com.example.impl.MyFilterImpl
```
**步骤四：使用自定义扩展点**
1. **通过ExtensionLoader获取**（编程方式）：
    ```java
    ExtensionLoader<MyFilter> loader = ExtensionLoader.getExtensionLoader(MyFilter.class);
    MyFilter filter = loader.getExtension("myFilter"); // 获取指定名称
    // 或者获取自适应扩展（如果接口方法有@Adaptive）
    MyFilter adaptiveFilter = loader.getAdaptiveExtension();
    // 或者获取激活的扩展列表
    List<MyFilter> filters = loader.getActivateExtension(url, "myfilter");
    ```
2. **通过Dubbo配置使用**：如果扩展点是Dubbo已知的类型（如`Filter`、`LoadBalance`），可以直接在配置文件中使用别名。
    ```xml
    <!-- 作为Filter使用 -->
    <dubbo:provider filter="myFilter"/>
    <!-- 作为LoadBalance使用 -->
    <dubbo:reference loadbalance="myLoadbalance"/>
    ```
    此时，Dubbo框架内部会通过`ExtensionLoader.getExtension("myFilter")`来加载你的实现。
**核心机制理解**：
- **扩展点自动发现**：Dubbo在启动时，会扫描所有`META-INF/dubbo/`、`META-INF/dubbo/internal/`、`META-INF/services/`目录下的SPI配置文件，并建立扩展名到实现类的映射关系，缓存在`ExtensionLoader`中。
- **依赖注入**：如果你的实现类中有setter方法引用了其他扩展点，Dubbo会自动注入。例如：
    ```java
    public class MyFilterImpl implements MyFilter {
        private LoadBalance loadBalance;
        // Dubbo会自动注入LoadBalance的适配实例
        public void setLoadBalance(LoadBalance loadBalance) {
            this.loadBalance = loadBalance;
        }
    }
    ```
- **Wrapper装饰**：如果你的实现类构造器参数是扩展点接口类型，它会被当作Wrapper，用于装饰其他扩展实例，实现AOP。
**调试与验证**：可以开启Dubbo的日志（`org.apache.dubbo.common.extension`）来查看扩展点加载过程。