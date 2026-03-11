# 如何正确关闭 IO 流？⁠⁠​

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
关闭流是为了释放系统资源（如文件句柄），否则会导致资源泄漏。有几种方式：
1. **传统的 try-catch-finally**（繁琐，不推荐）：
```java
    FileInputStream fis = null;
    try {
        fis = new FileInputStream("file.txt");
        // ... 使用流
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        if (fis != null) {
            try {
                fis.close(); // 确保在finally块中关闭
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    ```
2. **Try-with-Resources**（**强烈推荐**，Java 7+）：这是现代Java的标准做法。只要资源（如流）实现了 `AutoCloseable`接口，就可以在try后的括号中声明，JVM会**自动**在try块结束后正确关闭它们，即使发生异常也会关闭，代码非常简洁。
```java
    try (FileInputStream fis = new FileInputStream("source.jpg");
         FileOutputStream fos = new FileOutputStream("target.jpg")) {
        // ... 使用流
    } catch (IOException e) {
        e.printStackTrace();
    } // 无需显式关闭，自动处理
    ```

##关联知识
- 

## 延伸阅读（后续补充）
- 
