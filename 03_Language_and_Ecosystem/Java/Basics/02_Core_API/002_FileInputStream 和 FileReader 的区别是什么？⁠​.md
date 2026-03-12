# FileInputStream 和 FileReader 的区别是什么？

## 一句话说明（白话）
前者按字节读，后者按字符读。

## 它解决什么问题 / 为什么重要
避免编码错误，正确处理文本/二进制。

## 核心原理（一步步讲清楚）
- FileInputStream处理二进制。
- FileReader依赖字符集解码。

##典型使用场景
- 图片/文件用字节流。
- 文本用字符流。

## 简单例子 /伪代码
```java
new FileInputStream(file)
new FileReader(file)
```

## 常见坑与误区
- 用字符流读二进制文件导致损坏。

##关联知识
- 字节流/字符流

## 延伸阅读（后续补充）
- Java IO
