# Files 常用方法有哪些？

## 一句话说明（白话）
Files 是 NIO 的文件工具类，提供创建/复制/读取等操作。

## 它解决什么问题 / 为什么重要
统一、简洁地操作文件系统。

## 核心原理（一步步讲清楚）
- 静态工具类，封装底层IO与权限处理。

##典型使用场景
-读写文件、遍历目录。

## 简单例子 /伪代码
```java
Files.readAllLines(path);
Files.copy(src, dst);
```

## 常见坑与误区
- 大文件用 readAllLines可能内存爆。

##关联知识
- Path
- NIO

## 延伸阅读（后续补充）
- java.nio.file.Files
