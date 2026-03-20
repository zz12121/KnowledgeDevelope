# Mock与服务降级

## 概述

服务降级是指在服务出现故障或响应时间超过阈值的情况下，对服务进行降级处理，避免服务雪崩。Dubbo通过Mock机制实现服务降级和本地伪装。

## 使用场景

1. **降级非核心业务**：腾出系统资源，保证核心业务正常运行
2. **超时或不可用处理**：上游基础服务超时或不可用时，执行快速响应的降级预案
3. **服务容错**：某服务不可用时，返回预设的默认值

## Mock实现方式

Dubbo提供两种Mock实现：

### 1. 返回默认值

```java
// 消费者配置
@DubboReference(mock="return null")
private DemoService demoService;

// 或
@DubboReference(mock="return 666")
private DemoService demoService;
```

### 2. 调用自定义Mock类

```java
// 实现Mock类
public class DemoServiceMock implements DemoService {
    @Override
    public String sayHello(String name) {
        return "容错返回：Hello " + name;
    }
}

// 配置引用
@DubboReference(mock="com.example.DemoServiceMock")
private DemoService demoService;
```

## 配置方式

### Spring Boot配置

```yaml
dubbo:
  consumer:
    mock: true
    timeout: 3000
```

### XML配置

```xml
<!-- 强制返回null，不发起远程调用 -->
<dubbo:reference mock="force:return null" />

<!-- 失败后返回null -->
<dubbo:reference mock="fail:return null" />

<!-- 调用本地Mock类 -->
<dubbo:reference mock="com.example.DemoServiceMock" />
```

### Mock类规则

1. 实现与服务接口相同的接口
2. 类名以`Mock`结尾（如`DemoServiceMock`）
3. 放置在`mock`包下

## 工作原理

### MockClusterInvoker

Dubbo通过`MockClusterInvoker`实现Mock功能：

```java
public class MockClusterInvoker<T> implements Invoker<T> {
    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        // 检查是否配置了mock
        boolean isMock = false;
        // 根据配置选择调用逻辑
        if (isMock) {
            // 执行本地Mock
            return doMockInvoke(invocation);
        }
        // 正常远程调用
        return this.invoker.invoke(invocation);
    }
}
```

### 调用流程

1. 服务消费者发起调用
2. 集群容错层检查是否配置Mock
3. 如果配置了Mock且远程调用失败
4. 执行本地Mock方法或返回默认值

## Mock配置类型

| 配置 | 说明 |
|------|------|
| `mock="true"` | 使用接口名+Mock后缀的类 |
| `mock="force:return null"` | 强制返回null，不发起远程调用 |
| `mock="fail:return null"` | 远程调用失败后返回null |
| `mock="com.example.MockClass"` | 调用指定Mock类 |

## 与集群容错的关系

Mock机制是集群容错的补充：
- **Cluster**：处理网络层面的失败（重试、快速失败等）
- **Mock**：处理业务层面的降级（返回默认值）

## 适用场景

1. **非核心服务降级**：非关键业务出现故障时，快速返回备用数据
2. **系统保护**：避免因单个服务故障导致整个系统雪崩
3. **开发测试**：服务提供者未启动时，消费者可以进行功能测试

## 总结

Dubbo的Mock机制提供了灵活的服务降级能力，通过本地伪装和返回值配置，可以在服务不可用时保证系统整体可用性，是分布式系统中重要的容错手段。
