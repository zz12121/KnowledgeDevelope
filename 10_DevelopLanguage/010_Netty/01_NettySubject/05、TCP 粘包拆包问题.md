### 五、TCP 粘包/拆包问题

###### 1. TCP 粘包/拆包的原因及解决方法

**什么是粘包/拆包**：

- **粘包**：发送方发的多条消息，在接收方被合并成一条读出
- **拆包**：发送方的一条消息，在接收方被分成多次读取

**根本原因**：TCP 是面向字节流的协议，**不维护消息边界**，数据就是一串没有结构的字节流。

具体触发原因：
- **Nagle 算法**：TCP 会把多个小数据块合并成一个 MSS 大小的数据段再发送，导致粘包
- **发送缓冲区**：应用写入的数据小于缓冲区，TCP 等待更多数据才发
- **接收方读取慢**：多个数据包在接收缓冲区累积，应用一次读出多条
- **MTU 限制**：数据包在 IP 层被分片，TCP 层重组时可能重组出不完整的应用层消息

**解决方案（本质：在应用层定义消息边界）**：

1. **固定长度**：每条消息固定长度，不足填充。简单但浪费空间
2. **分隔符**：消息末尾加特殊分隔符（如 `\n`、`$$`）。不适合内容本身包含分隔符的场景
3. **长度字段（最推荐）**：消息头包含一个固定长度的字段表示消息体长度，动态确定边界
4. **复杂协议**：如 HTTP/1.1 用 Content-Length 或 chunked 编码界定消息体

---

###### 2. 什么是 TCP 粘包/拆包？Netty 是如何解决的？

**Netty 的解决方案**：在 ByteToMessageDecoder 抽象基类上，提供了一系列**帧解码器**，将接收到的字节流（可能是半包、多包）还原成一个个完整的应用层数据包。

**核心机制**：

所有帧解码器都继承自 ByteToMessageDecoder，其核心方法是：
```java
decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
```
- `in`：累积的输入缓冲区，可能包含半个包、一个包或多个包
- `out`：解码出的完整消息对象列表

**工作原理**：解码器不断尝试从 in 中识别出一个完整消息，识别出来就加入 out；数据不足则什么都不做，等待后续数据到来（ByteToMessageDecoder 会自动累积）。

---

###### 3. Netty 提供了哪些解码器来处理粘包/拆包？

Netty 在 `io.netty.handler.codec` 包中内置了四种主要解码器：

1. **FixedLengthFrameDecoder**：固定长度解码器
2. **LineBasedFrameDecoder**：行分隔符解码器（\n 或 \r\n）
3. **DelimiterBasedFrameDecoder**：自定义分隔符解码器
4. **LengthFieldBasedFrameDecoder**：基于长度字段的解码器（最强大最常用）

此外还有协议相关解码器：HttpObjectDecoder、WebSocketFrameDecoder 等，内部也处理了粘包。

**使用原则**：这些解码器应该添加在 Pipeline 最前端，确保字节流首先被正确帧化，后续 Handler 处理的都是完整的应用层数据包。

---

###### 4. LineBasedFrameDecoder 的工作原理是什么？

LineBasedFrameDecoder 以**行分隔符（\n 或 \r\n）**为消息边界，常用于文本协议（如 Redis 协议、SMTP）。

**工作原理**：

1. 遍历输入 ByteBuf 的可读字节，寻找 \n 或 \r\n 的位置
2. 找到分隔符后，从读指针到分隔符之间的字节切片作为一帧加入 out
3. 如果可读字节数达到 maxLength 仍未找到分隔符，抛出 TooLongFrameException 防止内存耗尽
4. 通过 stripDelimiter 参数控制是否将分隔符从帧中剥离（通常设 true）

**使用示例**：
```java
pipeline.addLast(new LineBasedFrameDecoder(1024));   // 最大行长度 1024 字节
pipeline.addLast(new StringDecoder());                // ByteBuf → String
pipeline.addLast(new MyBusinessHandler());
```

---

###### 5. DelimiterBasedFrameDecoder 的作用是什么？

DelimiterBasedFrameDecoder 是 LineBasedFrameDecoder 的通用版本，允许使用**任意用户指定的分隔符（支持多个分隔符）**作为消息边界，分隔符本身可以是多字节序列。

**工作原理**：

1. 构造时传入一个或多个 ByteBuf 作为分隔符。Netty 提供了便捷方法：`Delimiters.nulDelimiter()`（\0）和 `Delimiters.lineDelimiter()`（\n 和 \r\n）
2. 遍历输入缓冲区，查找任意一个预定义分隔符的首次出现位置
3. 找到后提取该帧，同样支持 maxFrameLength 和 stripDelimiter

**使用示例**：
```java
// 使用 "$$" 作为分隔符
ByteBuf delimiter = Unpooled.copiedBuffer("$$".getBytes());
pipeline.addLast(new DelimiterBasedFrameDecoder(2048, true, true, delimiter));
pipeline.addLast(new StringDecoder(CharsetUtil.UTF_8));
```

---

###### 6. FixedLengthFrameDecoder 是如何工作的？

FixedLengthFrameDecoder 按**固定字节长度**拆分消息，原理最简单：

1. 构造时指定 frameLength
2. decode() 中检查 in 的可读字节数是否 ≥ frameLength
3. 若是，切片 frameLength 字节加入 out，移动读指针
4. 若不足，什么都不做，等待更多数据

**源码极简**：
```java
protected Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
    if (in.readableBytes() < frameLength) {
        return null;                              // 不够，等待
    } else {
        return in.readRetainedSlice(frameLength); // 切片并增加引用计数
    }
}
```

**适用场景**：协议消息长度严格相同，如简单的控制指令或定长数据上报。

**缺点**：不灵活，消息长度不固定则无法使用。

---

###### 7. LengthFieldBasedFrameDecoder 的使用场景是什么？

LengthFieldBasedFrameDecoder 是**最强大、最常用**的粘包解决方案，适用于绝大多数自定义二进制协议，通过解析消息头中的长度字段动态确定每帧边界。

**使用场景**：RPC 框架（Dubbo、gRPC）、消息中间件（RocketMQ、Kafka 客户端协议）、游戏服务器自定义二进制协议……几乎所有私有 TCP 协议。

**五个核心参数**：

| 参数 | 含义 |
|------|------|
| maxFrameLength | 最大帧长，防止畸形数据 OOM |
| lengthFieldOffset | 长度字段的偏移量（跳过几个字节后才是长度字段） |
| lengthFieldLength | 长度字段本身占几个字节（1/2/3/4/8） |
| lengthAdjustment | 长度调整值，读出长度字段值后加减多少才是实际数据长度 |
| initialBytesToStrip | 从解码后的帧头部剥离几个字节再传给后续 Handler |

**解码计算公式**：`frameLength = lengthFieldOffset + lengthFieldLength + length + lengthAdjustment`

**两个典型例子**：

**例1**：协议格式 `[魔数(2B)][版本(1B)][长度(4B)][数据]`，长度字段只表示数据部分字节数：
```java
new LengthFieldBasedFrameDecoder(
    65536,  // maxFrameLength
    3,      // lengthFieldOffset  = 跳过魔数(2)+版本(1)
    4,      // lengthFieldLength  = 长度字段本身 4 字节
    0,      // lengthAdjustment   = 长度字段就是数据长度，不需要调整
    7       // initialBytesToStrip = 跳过魔数(2)+版本(1)+长度字段(4)，只传数据给后续 Handler
)
```

**例2**：协议格式 `[长度(2B)][数据]`，长度字段值表示整个包（含长度字段自身）的长度：
```java
new LengthFieldBasedFrameDecoder(
    65536,
    0,      // lengthFieldOffset = 长度字段在最前面
    2,      // lengthFieldLength
    -2,     // lengthAdjustment  = 长度字段包含了自身 2 字节，需减去
    2       // initialBytesToStrip = 跳过长度字段
)
```

---

###### 8. 【高频追问】ByteToMessageDecoder 的 decode 方法为什么每次都要检查是否有足够数据？不够的话如何处理？

这是设计上的核心约定：**ByteToMessageDecoder 会自动处理数据累积，开发者只需关注单次解码逻辑**。

当 decode() 中数据不足时，直接 return 什么都不输出到 out，ByteToMessageDecoder 的父类实现会：
1. 保留当前 in 中未消费的字节（不丢弃）
2. 等待更多数据到来，TCP 层的后续数据到达后合并到同一个 ByteBuf
3. 重新调用 decode()，此时 in 中有了更多数据

**必须要做的操作**：在解码过程中，如果读取了部分字段发现数据不完整，要通过 `markReaderIndex()` + `resetReaderIndex()` 把读指针恢复到这次尝试前的位置，避免消费了一部分数据导致下次解码从错误位置开始：

```java
@Override
protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    if (in.readableBytes() < HEADER_SIZE) return;  // 直接 return
    
    in.markReaderIndex();          // 标记位置
    byte type = in.readByte();
    int bodyLen = in.readInt();
    
    if (in.readableBytes() < bodyLen) {
        in.resetReaderIndex();     // 数据不够，回退到 markReaderIndex 的位置
        return;
    }
    // 数据足够，正常解码
    out.add(buildMessage(type, in.readBytes(bodyLen)));
}
```
