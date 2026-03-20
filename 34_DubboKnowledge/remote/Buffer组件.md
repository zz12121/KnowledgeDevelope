# Buffer组件

## 概述

Buffer（缓冲区）是Dubbo网络通信中的字节容器，抽象为ChannelBuffer接口。设计借鉴了Netty的ByteBuf，统一了不同NIO框架（如Java NIO的ByteBuffer、Netty的ByteBuf）中的字节容器实现。

## 核心设计

### 指针机制

- **readerIndex**：读指针，标识可读区域起始位置
- **writerIndex**：写指针，标识可写区域起始位置
- **markedReaderIndex/markedWriterIndex**：标记位置，支持reset回滚

### 读写方法

- **getBytes()/setBytes()**：读写数据，不移动指针
- **readBytes()/writeBytes()**：读写数据，移动指针

## 核心组件

### ChannelBuffer接口

定义读写、标记/重置、容量查询等方法：

```java
public interface ChannelBuffer {
    int readerIndex();
    int writerIndex();
    ChannelBuffer markReaderIndex();
    ChannelBuffer resetReaderIndex();
    // ... 其他方法
}
```

### AbstractChannelBuffer

实现readerIndex和writerIndex的维护，读写方法委托给子类。

### 实现类

| 实现类 | 底层存储 | 特点 |
|--------|----------|------|
| HeapChannelBuffer | 字节数组 | 使用System.arraycopy()高效拷贝 |
| DynamicChannelBuffer | 装饰器模式 | 支持动态扩容（容量不足时翻倍） |
| ByteBufferBackedChannelBuffer | Java NIO ByteBuffer | 兼容堆外内存管理 |
| NettyBackedChannelBuffer | Netty ByteBuf | 直接封装Netty的Buffer |

## 动态扩容机制

DynamicChannelBuffer在写入数据前调用`ensureWritableBytes()`检查空间：
- 若不足，创建新Buffer（容量翻倍）
- 拷贝原数据并替换内部引用

## 工具类ChannelBuffers

提供工厂方法创建各类Buffer：
- `buffer(capacity)`：创建指定大小的HeapChannelBuffer
- `dynamicBuffer()`：创建默认256字节的DynamicChannelBuffer
- `directBuffer()`：创建堆外内存Buffer
- `equals()/compare()`：比较两个Buffer内容

## 配套输入输出流

- **ChannelBufferInputStream**：封装ChannelBuffer，提供流式读接口
- **ChannelBufferOutputStream**：封装ChannelBuffer，提供流式写接口

## 设计优势

1. **统一抽象**：兼容多种NIO框架的底层实现
2. **动态扩容**：自动适应不同数据大小
3. **高效操作**：使用数组拷贝而非逐字节操作
4. **指针灵活**：支持mark/reset回滚操作

## 总结

Dubbo的Buffer设计通过抽象接口和多种实现，兼容了多种NIO框架的底层字节容器，提供了动态扩容、内存高效管理等特性，适合高并发网络通信场景。
