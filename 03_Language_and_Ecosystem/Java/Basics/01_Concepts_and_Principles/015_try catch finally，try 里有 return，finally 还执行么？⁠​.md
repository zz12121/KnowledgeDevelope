# try里有 return，finally还执行么？

## 一句话说明（白话）
会执行，除非 JVM直接退出。

## 它解决什么问题 / 为什么重要
保证资源释放逻辑不会被 return 跳过。

## 核心原理（一步步讲清楚）
- return只是“标记返回值”。
- finally 在实际返回前执行。

##典型使用场景
-保证关闭资源、释放锁。

## 简单例子 /伪代码
```java
try { return1; } finally { System.out.println("cleanup"); }
```

## 常见坑与误区
- finally里 return 会覆盖 try 的 return。

##关联知识
- finally
- try-with-resources

## 延伸阅读（后续补充）
- Java 异常控制流
