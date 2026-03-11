# 如何开启 GC 日志？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
日志是分析 GC 行为和进行调优的根本依据。

| 参数                                   | 作用                                     |
| ------------------------------------ | -------------------------------------- |
| **-Xloggc:<file>**​                  | 指定 GC 日志文件的输出路径。                       |
| **-XX:+PrintGC**​                    | 输出简单的 GC 日志。                           |
| **-XX:+PrintGCDetails**​             | 输出**详细的 GC 日志**（包括各区内存变化、耗时等），这是分析的关键。 |
| **-XX:+PrintGCDateStamps**​          | 在 GC 日志中输出**日期时间戳**，便于定位。              |
| **-XX:+PrintGCTimeStamps**​          | 在 GC 日志中输出**相对于 JVM 启动的时间戳**​。         |
| **-XX:+HeapDumpOnOutOfMemoryError**​ | 在发生 **OOM 时自动生成堆转储文件**，用于事后分析内存泄漏。     |
| **-XX:HeapDumpPath=<path>**​         | 指定堆转储文件的生成路径。                          |

| 收集器              | 启用参数                      | 关键调优参数                                                                |
| ---------------- | ------------------------- | --------------------------------------------------------------------- |
| **Serial GC**​   | `-XX:+UseSerialGC`        | 适用于客户端或微服务场景。                                                         |
| **Parallel GC**​ | `-XX:+UseParallelGC`      | `-XX:ParallelGCThreads`（GC线程数）`,`-XX:MaxGCPauseMillis`（最大暂停时间目标）。     |
| **CMS GC**​      | `-XX:+UseConcMarkSweepGC` | `-XX:CMSInitiatingOccupancyFraction`（触发回收的老年代占用率）。                    |
| **G1 GC**​       | `-XX:+UseG1GC`            | `-XX:MaxGCPauseMillis`, `-XX:InitiatingHeapOccupancyPercent`（IHOP阈值）。 |
| **Z GC**​        | `-XX:+UseZGC`             | `-XX:MaxGCPauseMillis`。                                               |

##关联知识
- 

## 延伸阅读（后续补充）
- 
