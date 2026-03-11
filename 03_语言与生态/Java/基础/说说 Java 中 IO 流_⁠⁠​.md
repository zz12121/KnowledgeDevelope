# 说说 Java 中 IO 流?⁠⁠​

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
关于Java IO流，它就像是程序与外界（文件、网络、控制台等）进行数据交流的桥梁。

|特性|字节流 (Byte Streams)|字符流 (Character Streams)|
|---|---|---|
|**数据单位**​|字节 (8-bit)|字符 (16-bit Unicode)|
|**基类**​|`InputStream`/ `OutputStream`|`Reader`/ `Writer`|
|**处理范围**​|所有二进制数据（如图片、视频、音频）|文本数据|
|**编码处理**​|不涉及字符编码，直接操作原始字节|**自动处理编码转换**，依赖指定或默认的字符集（如UTF-8）|
|**核心用途**​|处理非文本文件，保证数据精确无误|处理文本文件，简化字符操作，避免乱码|
|**典型类**​|`FileInputStream`, `FileOutputStream`  <br>`BufferedInputStream`, `BufferedOutputStream`  <br>`ObjectInputStream`（对象序列化）|`FileReader`, `FileWriter`  <br>`BufferedReader`, `BufferedWriter`  <br>`InputStreamReader`（转换流）|

##关联知识
- 

## 延伸阅读（后续补充）
- 
