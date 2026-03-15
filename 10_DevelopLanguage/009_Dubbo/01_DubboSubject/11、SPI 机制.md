###### 1. 知道 Dubbo SPI 机制吗？

Dubbo SPI 是整个框架**可扩展性的基石**，也是面试必考点。

**核心三要素**：
- **扩展点（Extension Point）**：带 `@SPI` 注解的接口，如 `Protocol`、`LoadBalance`、`Cluster`。
- **扩展（Extension）**：接口的具体实现，如 `DubboProtocol`、`RandomLoadBalance`。
- **ExtensionLoader**：核心加载器，负责按需加载、缓存、依赖注入、Wrapper 包装。

**与 JDK SPI 的核心区别**：

| 能力 | JDK SPI | Dubbo SPI |
|---|---|---|
| **加载方式** | 全量加载，浪费资源 | 按名称懒加载，有缓存 |
| **依赖注入** | 不支持 | 支持 Setter 自动注入 |
| **条件激活** | 不支持 | `@Activate` 条件过滤+排序 |
| **AOP 包装** | 不支持 | Wrapper 装饰器自动叠加 |
| **自适应** | 不支持 | `@Adaptive` 运行时动态选择实现 |

**配置文件格式**（在 `META-INF/dubbo/` 目录下，文件名是接口全限定名）：
```
dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
tri=org.apache.dubbo.rpc.protocol.tri.TripleProtocol
```
键=值，支持别名，加载时按需创建实例。

**源码入口**：`ExtensionLoader.getExtension(name)` 按名称获取；`getAdaptiveExtension()` 获取自适应实例；`getActivateExtension(url, key)` 获取条件激活列表。

---

###### 2. Dubbo 为什么不采用 JDK SPI？

JDK SPI 有四个致命不足，Dubbo 全部解决了：

**① 不支持按需加载**
JDK `ServiceLoader` 一次性加载并实例化配置文件里的**所有**实现类。Dubbo SPI 只在 `getExtension(name)` 时才加载对应名称的类，并缓存到 `cachedClasses`（类缓存）和 `EXTENSION_INSTANCES`（实例缓存），避免资源浪费。

**② 不支持依赖注入**
JDK SPI 只用无参构造器创建对象，没有 IoC 能力。Dubbo SPI 的 `injectExtension()` 方法会扫描所有 setter 方法，如果参数类型是另一个扩展点接口，就通过 `objectFactory.getExtension()` 自动注入。`objectFactory` 是 `AdaptiveExtensionFactory`，可以从 Dubbo 自身或 Spring 容器中获取依赖。

**③ 没有条件激活机制**
JDK SPI 只能获取全部实现，无法根据上下文筛选。Dubbo 的 `@Activate(group, value, order)` 注解让 Filter 链可以根据 Consumer/Provider 角色和 URL 参数动态组装，且有顺序控制。

**④ 没有 AOP/Wrapper 机制**
JDK SPI 返回原始实例，没有装饰能力。Dubbo SPI 识别"Wrapper 类"（构造器参数是扩展点接口本身），在创建扩展实例后自动用 Wrapper 层层包装，形成责任链。例如 `ProtocolFilterWrapper` 和 `ProtocolListenerWrapper` 就是 `Protocol` 的 Wrapper，为 `Protocol` 实例添加 Filter 链和监听器链。

---

###### 3. Dubbo SPI 的实现原理是什么？

`ExtensionLoader` 的工作分五个阶段：

**① 加载配置文件**
`loadExtensionClasses()` 扫描三个路径：`META-INF/dubbo/internal/`、`META-INF/dubbo/`、`META-INF/services/`，解析 `key=实现类全限定名`，存入 `cachedClasses: Map<String, Class<?>>` 缓存。

**② 获取扩展实例（双重检查）**
`getExtension(name)` 先查 `cachedInstances`，命中直接返回；否则进同步块再检查一次，最后调 `createExtension(name)`。

**③ 创建扩展实例（三步走）**
```
clazz.newInstance()          // 反射无参构造
→ injectExtension(instance)  // Setter 依赖注入
→ Wrapper 包装               // 逆序叠加所有 Wrapper 类
```

**④ Wrapper 包装原理**
`ExtensionLoader` 扫描配置文件时，会识别构造器只有一个参数且参数类型是扩展点接口的类——这就是 Wrapper。创建原始实例后，逐个用 Wrapper 构造器包装：
```java
instance = wrapperClass.getConstructor(type).newInstance(instance);
// 多个Wrapper → 链式叠加，形成装饰器链
```

**⑤ 缓存贯穿始终**
类缓存（`cachedClasses`）、实例缓存（`EXTENSION_INSTANCES`）、自适应实例缓存（`cachedAdaptiveInstance`），大量 `ConcurrentHashMap` 确保线程安全和高性能。

---

###### 4. Dubbo SPI 的 @Adaptive 注解是做什么的？

`@Adaptive` 实现**自适应扩展**——在运行时根据 URL 参数动态决定用哪个具体实现，而不是在启动时写死。

**两种用法**：

1. **标注在类上**（少见）：该类是自适应实现，如 `AdaptiveExtensionFactory`。
2. **标注在接口方法上**（主要用法）：告诉框架"这个方法需要动态生成代理逻辑"。

**动态生成原理**：
对于 `Protocol` 接口，`getAdaptiveExtension()` 会生成 `Protocol$Adaptive` 代理类（字符串拼接 Java 代码，然后用 `JavassistCompiler` 编译）：

```java
// 生成的代理类（伪代码）
public class Protocol$Adaptive implements Protocol {
    public Exporter export(Invoker invoker) {
        URL url = invoker.getUrl();
        // 从URL取protocol参数，没有就用默认值"dubbo"
        String extName = url.getProtocol() == null ? "dubbo" : url.getProtocol();
        // 根据extName动态加载真正的实现
        Protocol extension = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName);
        return extension.export(invoker);
    }
}
```

**URL 是灵魂**：`@Adaptive("loadbalance")` 表示从 URL 的 `loadbalance` 参数取扩展名；如果不指定，则根据接口名转换（`LoadBalance` → `load.balance` → `loadbalance`）。

**作用**：让框架在同一个代码路径上，根据不同的调用 URL，透明地切换底层实现（如 dubbo 协议还是 hessian 协议、random 还是roundrobin 负载均衡）。

---

###### 5. Dubbo SPI 的 @Activate 注解是做什么的？

`@Activate` 实现**条件激活**——让框架根据上下文（URL 参数、角色）批量、有序地激活一组扩展，典型应用是 **Filter 链的自动组装**。

**注解三个核心属性**：
- `group()`：`PROVIDER` 或 `CONSUMER`，限制在哪个角色端生效
- `value()`：URL 包含该参数时才激活，如 `@Activate(value = "accesslog")` 表示 URL 有 `accesslog` 参数才激活 `AccessLogFilter`
- `order()`：数字越小越靠外（先执行前置逻辑）

**工作流程**：
调用 `ExtensionLoader.getActivateExtension(url, "filter")` 时：
1. 加载所有带 `@Activate` 的实现，存入 `cachedActivates`
2. 按当前 `group`（Consumer/Provider）过滤
3. 按 URL 参数和 `@Activate.value` 过滤
4. 处理手动配置的 `-xxx`（排除）或显式列出的 Filter
5. 按 `order` 排序，去重，返回有序列表

**实际场景**：`ProtocolFilterWrapper.buildInvokerChain()` 就是通过 `getActivateExtension()` 获取所有激活的 Filter，然后逆序构建调用链。URL 中有 `accesslog` 参数时，`AccessLogFilter` 自动加入；在 Consumer 端时，`ExecuteLimitFilter`（Provider专属）不会出现。

---

###### 6. 如何自定义 Dubbo 扩展点？

以自定义 Filter 为例，三步走：

**步骤一：实现 Filter 接口，加 @Activate 注解**
```java
package com.example.filter;

@Activate(group = PROVIDER, order = 100)
public class AuditFilter implements Filter {
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        long start = System.currentTimeMillis();
        String method = invocation.getMethodName();
        try {
            Result result = invoker.invoke(invocation);
            log.info("调用 {} 成功，耗时 {}ms", method, System.currentTimeMillis() - start);
            return result;
        } catch (RpcException e) {
            log.error("调用 {} 失败: {}", method, e.getMessage());
            throw e;
        }
    }
}
```

**步骤二：创建 SPI 配置文件**
在 `src/main/resources/META-INF/dubbo/` 目录下创建文件 `org.apache.dubbo.rpc.Filter`，内容：
```
auditFilter=com.example.filter.AuditFilter
```

**步骤三：启用（有了 @Activate 可自动激活，也可手动指定）**
```yaml
dubbo:
  provider:
    filter: auditFilter  # 手动启用
  # 或者不配置，依赖 @Activate 自动激活
```

**其他扩展点同理**：自定义 LoadBalance、Router、Cluster 都是这三步，只是实现的接口和 SPI 文件名不同。

**Dubbo 3.x 新增方式**：可以用 Spring Bean 注册扩展，在 Spring 容器启动后动态添加，更灵活。

---

###### 7. Dubbo SPI 和 Spring IOC 如何协作？（新增）

这是面试容易被追问的深度题。

**问题背景**：Dubbo 有自己的扩展加载机制（`ExtensionLoader`），Spring 有 IOC 容器（`ApplicationContext`）。二者如何协同工作，使得 Dubbo SPI 扩展点可以注入 Spring Bean？

**关键：AdaptiveExtensionFactory**

`ExtensionLoader` 在执行依赖注入时，调用的是 `objectFactory`（`ExtensionFactory` 类型的自适应扩展）来获取依赖。`ExtensionFactory` 的 SPI 配置包含了两个实现：
- `SpiExtensionFactory`：从 Dubbo SPI 容器中获取依赖（目标是扩展点类型）
- `SpringExtensionFactory`：从 Spring `ApplicationContext` 中获取依赖（目标是普通 Spring Bean）

`AdaptiveExtensionFactory` 作为适配器，持有这两个工厂，按顺序尝试：先查 SPI，再查 Spring。

```java
// 伪代码：AdaptiveExtensionFactory.getExtension()
for (ExtensionFactory factory : factories) {
    T extension = factory.getExtension(type, name);
    if (extension != null) return extension;
}
return null;
```

**实际效果**：你的自定义 Dubbo Filter 可以通过 setter 注入 Spring Bean：
```java
@Activate(group = PROVIDER)
public class PermissionFilter implements Filter {
    private PermissionService permissionService; // Spring Bean
    
    // Dubbo会自动调用这个setter，从Spring容器注入PermissionService
    public void setPermissionService(PermissionService permissionService) {
        this.permissionService = permissionService;
    }
    
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        permissionService.check(invocation);  // 使用Spring Bean
        return invoker.invoke(invocation);
    }
}
```

**Spring 注入到 ExtensionLoader 的时机**：`DubboSpringInitializer` 在 Spring 容器刷新完成后，调用 `SpringExtensionFactory.addApplicationContext(context)` 将 `ApplicationContext` 注册进来，之后 `ExtensionLoader` 就能从 Spring 容器获取 Bean 了。
