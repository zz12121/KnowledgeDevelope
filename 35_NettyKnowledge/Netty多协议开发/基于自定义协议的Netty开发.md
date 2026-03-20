# 基于自定义协议的Netty开发

## 概述

在实际应用场景中（如与下位机、PLC通信），需要自定义二进制协议。Netty提供了灵活的编解码接口，可以方便地实现自定义协议。

## 自定义协议设计

### 协议格式

常见的自定义协议格式：

```
+--------+--------+--------+--------+--- ... ---+
|  魔数  | 版本号 |  指令  | 长度   |   数据    |
| 4字节  | 1字节  |  1字节  | 4字节  |   N字节  |
+--------+--------+--------+--------+--- ... ---+
```

- **魔数**：标识协议开始（如0xCAFEBABE）
- **版本号**：协议版本
- **指令**：消息类型
- **长度**：数据部分长度
- **数据**：实际业务数据

## 自定义解码器

### ByteToMessageDecoder

```java
public class MyDecoder extends ByteToMessageDecoder {
    // 协议头部长度
    private static final int HEADER_SIZE = 10;
    
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        // 可读字节数不足，跳过
        if (in.readableBytes() < HEADER_SIZE) {
            return;
        }
        
        // 标记读取位置
        in.markReaderIndex();
        
        // 读取魔数
        int magic = in.readInt();
        // 读取版本号
        byte version = in.readByte();
        // 读取指令
        byte cmd = in.readByte();
        // 读取数据长度
        int length = in.readInt();
        
        // 数据长度不足，等待下次读取
        if (in.readableBytes() < length) {
            in.resetReaderIndex();
            return;
        }
        
        // 读取数据
        byte[] data = new byte[length];
        in.readBytes(data);
        
        // 构造消息对象
        MyMessage message = new MyMessage();
        message.setMagic(magic);
        message.setVersion(version);
        message.setCmd(cmd);
        message.setData(data);
        
        out.add(message);
    }
}
```

## 自定义编码器

### MessageToByteEncoder

```java
public class MyEncoder extends MessageToByteEncoder<MyMessage> {
    @Override
    protected void encode(ChannelHandlerContext ctx, MyMessage msg, ByteBuf out) {
        // 写入魔数
        out.writeInt(msg.getMagic());
        // 写入版本号
        out.writeByte(msg.getVersion());
        // 写入指令
        out.writeByte(msg.getCmd());
        // 写入数据长度
        byte[] data = msg.getData();
        out.writeInt(data.length);
        // 写入数据
        out.writeBytes(data);
    }
}
```

## 组合编解码器

### ByteToMessageCodec

```java
public class MyCodec extends ByteToMessageCodec<MyMessage> {
    // 编码和解码逻辑合并
}
```

## 解决粘包/半包

### LengthFieldBasedFrameDecoder

Netty提供了基于长度字段的解码器：

```java
// 参数说明：最大帧长度、长度字段偏移、长度字段字节数
pipeline.addLast("frameDecoder", 
    new LengthFieldBasedFrameDecoder(65535, 4, 4));
pipeline.addLast("decoder", new MyDecoder());
pipeline.addLast("encoder", new MyEncoder());
```

## Pipeline配置

```java
bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) {
        ChannelPipeline pipeline = ch.pipeline();
        // 添加帧解码器（解决粘包/半包）
        pipeline.addLast("frameDecoder", 
            new LengthFieldBasedFrameDecoder(65535, 4, 4));
        // 添加自定义解码器
        pipeline.addLast("decoder", new MyDecoder());
        // 添加自定义编码器
        pipeline.addLast("encoder", new MyEncoder());
        // 添加业务处理器
        pipeline.addLast("handler", new MyBusinessHandler());
    }
});
```

## 适用场景

1. **物联网通信**：与硬件设备通信
2. **私有协议**：企业内部通信协议
3. **二进制协议**：数据传输要求高效的场景
4. **嵌入式系统**：资源受限环境

## 总结

Netty提供了强大的编解码框架，通过继承ByteToMessageDecoder和MessageToByteEncoder，可以灵活实现各种自定义协议。结合LengthFieldBasedFrameDecoder，可以有效解决TCP粘包/半包问题。
