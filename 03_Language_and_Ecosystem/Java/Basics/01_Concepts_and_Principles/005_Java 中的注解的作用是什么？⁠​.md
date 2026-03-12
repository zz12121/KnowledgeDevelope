# Java 中的注解的作用是什么？

## 一句话说明（白话）
注解就是给代码贴“标签”，供编译器或框架读取使用。

## 它解决什么问题 / 为什么重要
-减少 XML 配置。
-让工具和框架自动完成校验、注入、生成代码等动作。

## 核心原理（一步步讲清楚）
- 注解是**元数据**，本身不改变逻辑。
-通过**编译期/运行期**处理器读取。
-典型读取方式：反射、APT（注解处理器）。

##典型使用场景
- Spring：`@Component`、`@Autowired`。
- JPA：`@Entity`、`@Table`。
- Lombok：`@Data`生成样板代码。

## 简单例子 /伪代码
```java
@Retention(RUNTIME)
@Target(METHOD)
public @interface Log {}
```

## 常见坑与误区
-误用 `@Retention` 导致运行期读不到。
- 滥用注解导致代码难读。

##关联知识
- 元注解（Retention/Target/Documented）
-反射
- APT

## 延伸阅读（后续补充）
- Java 注解与处理器
