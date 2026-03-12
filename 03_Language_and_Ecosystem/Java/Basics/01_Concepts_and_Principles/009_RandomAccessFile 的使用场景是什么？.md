# RandomAccessFile 的使用场景是什么？

## 一句话说明（白话）
它允许在文件的任意位置读写，不必顺序读。

## 它解决什么问题 / 为什么重要
实现断点续传、多线程分块写入等随机访问能力。

## 核心原理（一步步讲清楚）
-维护一个文件指针，可通过 `seek()`移动。
-支持 `r`/`rw` 模式随机读写。

##典型使用场景
-断点续传下载。
-日志文件末尾追加。
-固定长度记录文件（简单数据库）。

## 简单例子 /伪代码
```java
RandomAccessFile raf = new RandomAccessFile("a.log","rw");
raf.seek(raf.length());
raf.writeBytes("append");
```

## 常见坑与误区
-忽略文件指针位置导致覆盖数据。
-多线程并发写需自行加锁。

##关联知识
- NIO FileChannel
- 文件锁

## 延伸阅读（后续补充）
- Java IO 与 NIO
