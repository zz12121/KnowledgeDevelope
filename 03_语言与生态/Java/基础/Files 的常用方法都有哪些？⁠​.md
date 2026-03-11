# Files 的常用方法都有哪些？⁠​

## 一句话说明（白话）

这是一个 Java 基础概念，用来解释该语法/特性在实际开发中的意义。

## 它解决什么问题 / 为什么重要

帮助理解语言规则，避免写出低质量或错误代码。

## 核心原理（一步步讲清楚）

先说明语法或特性，再说明编译/运行时的行为与影响。

##典型使用场景

面试常问点、日常编码中会用到。

## 简单例子 /伪代码

给出最小可运行的示例帮助理解。

## 常见坑与误区

列出1-2个常见误用。

##题库要点（原始材料）
在Java 7中引入的 `java.nio.file.Files`类，提供了大量静态方法来操作文件，极大简化了IO操作，是现代Java开发的首选。

| 操作类型      | 常用方法                                                             | 说明                    |
| --------- | ---------------------------------------------------------------- | --------------------- |
| **读写文件**​ | `Files.readAllBytes(Path path)`                                  | 一次性读取所有字节，适用于小文件。     |
|           | `Files.readAllLines(Path path)`                                  | 读取所有行到一个List<String>。 |
|           | `Files.write(Path path, byte[] bytes)`                           | 将字节数组写入文件。            |
|           | `Files.write(Path path, Iterable<? extends CharSequence> lines)` | 将字符串集合逐行写入文件。         |
|           | `Files.newBufferedReader(Path path)`                             | 创建高效的BufferedReader。  |
| **文件操作**​ | `Files.exists(Path path)`                                        | 检查文件或目录是否存在。          |
|           | `Files.createFile(Path path)`                                    | 创建新文件。                |
|           | `Files.createDirectories(Path path)`                             | 创建目录（包括不存在的父目录）。      |
|           | `Files.copy(InputStream in, Path target)`                        | 从输入流复制到文件。            |
|           | `Files.move(Path source, Path target)`                           | 移动或重命名文件。             |
|           | `Files.delete(Path path)`                                        | 删除文件或目录（不存在则抛异常）。     |
|           | `Files.deleteIfExists(Path path)`                                | 安全删除。                 |
| **获取信息**​ | `Files.size(Path path)`                                          | 返回文件大小（字节）。           |
|           | `Files.isDirectory(Path path)`                                   | 判断是否是目录。              |
|           | `Files.getLastModifiedTime(Path path)`                           | 获取最后修改时间。             |

##关联知识
- 

## 延伸阅读（后续补充）
- 
