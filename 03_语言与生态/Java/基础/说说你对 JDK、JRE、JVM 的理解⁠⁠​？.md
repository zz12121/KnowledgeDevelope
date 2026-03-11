# 说说你对 JDK、JRE、JVM 的理解⁠⁠​？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
JDK、JRE 和 JVM 是 Java 技术的核心组件，关系紧密但职责不同：
- **JVM（Java Virtual Machine，Java 虚拟机）**：是虚拟的计算机，负责将 Java 字节码（.class 文件）解释或编译为特定平台的机器指令执行。JVM 是 Java 跨平台的基础，不同平台有对应的 JVM 实现。
- **JRE（Java Runtime Environment，Java 运行环境）**：是运行 Java 程序所需的环境，包含 JVM 和 Java 核心类库（如 java.lang、java.util）。用户只需安装 JRE 即可运行已编译的 Java 程序，但无法进行开发。
- **JDK（Java Development Kit，Java 开发工具包）**：是开发者使用的工具集，包含 JRE 以及开发工具（如编译器 javac、调试器 jdb、打包工具 jar）。JDK 用于编写、编译和调试 Java 代码。
- **三者的关系**：JDK ⊃ JRE ⊃ JVM。即 JDK 包含 JRE，JRE 包含 JVM。开发 Java 程序需要 JDK，而运行程序只需 JRE

##关联知识
- 

## 延伸阅读（后续补充）
- 
