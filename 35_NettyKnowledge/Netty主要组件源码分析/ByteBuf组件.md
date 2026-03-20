# ByteBuf组件

## 概述

ByteBuf是Netty重新设计的字节容器，用于替代JDK的ByteBuffer。Netty针对高并发网络编程场景，在内存管理、API设计、架构扩展性等方面进行了全面优化。

## 与ByteBuffer对比

| 特性 | ByteBuffer | ByteBuf |
|------|------------|---------|
| 内存管理 | 依赖position/limit/capacity，需手动flip() | **读写索引分离**，无需手动翻转 |
| 内存分配 | 无池化，每次创建新缓冲区 | **支持池化**，对象复用 |
| 容量 | 固定容量 | **动态扩容** |
| 扩展性 | 单一实现 | 支持池化/非池化、自定义分配 |

## 核心设计

### 读写索引分离

- **readerIndex**：读指针，标识可读区域起始
- **writerIndex**：写指针，标识可写区域起始
- **capacity**：总容量

```java
// 读写操作
ByteBuf buffer = Unpooled.buffer(1024);
buffer.writeBytes(data);  // 写入，移动writerIndex
byte[] readData = new byte[length];
buffer.readBytes(readData);  // 读取，移动readerIndex
```

### 动态扩容

当写入数据超过当前容量时，ByteBuf自动扩容：

```java
// 扩容规则
if (writerIndex + minWritableBytes > capacity) {
    // 新容量 = 容量 * 2 或 所需最小容量
}
```

## 内存类型

### HeapByteBuf

基于JVM堆内存（byte[]），分配快，GC回收。

### DirectByteBuf

基于堆外内存（直接内存），减少IO时的内存拷贝，适合高并发场景。

### PooledByteBuf

池化内存，复用缓冲区，减少分配开销。

## 核心API

### 读写操作

```java
// 写入
buffer.writeByte(1);
buffer.writeInt(100);
buffer.writeBytes(data);

// 读取
byte b = buffer.readByte();
int i = buffer.readInt();
buffer.readBytes(bytes);
```

### 索引操作

```java
// 设置索引
buffer.readerIndex(10);
buffer.writerIndex(20);

// 标记和重置
buffer.markReaderIndex();
buffer.resetReaderIndex();
```

### 切片操作

```java
// 零拷贝切片
ByteBuf slice = buffer.slice(start, length);
```

## 内存管理

### 池化机制

PooledByteBufAllocator将大块内存（Chunk，默认16MB）切分成固定粒度的Page（默认8KB），按需分配。

### 引用计数

ByteBuf实现ReferenceCounted接口：

```java
// 增加引用
buffer.retain();

// 释放引用
buffer.release();
```

## 零拷贝技术

### CompositeByteBuf

组合多个ByteBuf，无需数据拷贝：

```java
CompositeByteBuf composite = Unpooled.compositeBuffer();
composite.addComponent(true, header);
composite.addComponent(true, body);
```

### slice()和duplicate()

返回原ByteBuf的视图，不复制数据。

## 适用场景

1. **高并发网络通信**：池化减少GC压力
2. **大数据传输**：动态扩容适应不同数据量
3. **协议解析**：丰富API简化开发

## 总结

ByteBuf是Netty对ByteBuffer的全面升级，通过读写索引分离、池化内存、动态扩容、零拷贝等技术，为高性能网络编程提供了强大的数据容器。
