# Hessian协议

## 概述

Hessian是一种支持动态类型、跨语言、基于对象传输的网络协议。Java对象序列化的二进制流可以被其他语言（如C++、Python）反序列化。Dubbo默认使用Hessian2作为序列化方案。

## 特性

1. **自描述序列化类型**：不依赖外部描述文件或接口定义，使用一个字节表示常用基础类型，极大缩短二进制流
2. **语言无关**：支持脚本语言间的互操作
3. **协议简单**：比Java原生序列化高效
4. **Hessian2压缩编码**：
   - 序列化二进制流大小是Java序列化的50%
   - 序列化耗时是Java序列化的30%
   - 反序列化耗时是Java序列化的20%

## 在Dubbo中的使用

### 依赖配置

从Dubbo 3开始，Hessian协议不再内嵌在Dubbo中，需要单独添加依赖：

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>hessian-lite</artifactId>
    <version>4.0.13</version>
</dependency>
```

### Spring Boot配置

```yaml
dubbo:
  protocol:
    serialization: hessian2
```

### XML配置

```xml
<dubbo:protocol serialization="hessian2" />
<!-- 或针对特定服务 -->
<dubbo:reference interface="xxx" serialization="hessian2" />
```

## 序列化原理

### Hessian2Serialization

Dubbo中通过`Hessian2Serialization`类实现Serialization接口：

```java
public class Hessian2Serialization implements Serialization {
    public static final byte HEADER_ID = (byte)0x02;
    
    @Override
    public ObjectOutput serialize(URL url, OutputStream output) throws IOException {
        return new Hessian2ObjectOutput(output);
    }
    
    @Override
    public ObjectInput deserialize(URL url, InputStream input) throws IOException {
        return new Hessian2ObjectInput(input);
    }
}
```

### Hessian2SerializerFactory

序列化工厂负责创建序列化器和反序列化器，继承了Hessian的序列化机制。

## 与其他序列化协议对比

| 特性 | Hessian2 | Dubbo | JSON | Java原生 |
|------|----------|-------|------|----------|
| 跨语言 | 支持 | 不支持 | 支持 | 不支持 |
| 序列化效率 | 高 | 最高 | 中 | 低 |
| 可读性 | 二进制 | 二进制 | 文本 | 二进制 |
| 安全风险 | 存在 | 存在 | 较低 | 高 |

## 适用场景

1. **跨语言调用**：需要与其他语言（如Python、C++）互操作的场景
2. **性能要求较高**：相比JSON有更好的性能
3. **数据量适中**：适合中等大小的对象序列化

## 安全注意事项

Hessian反序列化存在安全风险，建议：
- 启用Hessian黑白名单过滤
- 避免暴露RPC端口到公网
- 及时更新Hessian版本到最新补丁版本

## 总结

Hessian协议是Dubbo中常用的序列化方案之一，通过自描述类型实现了高效紧凑的二进制序列化，同时支持跨语言调用，是分布式系统中理想的选择。
