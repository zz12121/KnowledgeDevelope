# Stream API 对集合操作有什么优势？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
Java 8 引入的 Stream API 为集合操作带来了革命性的变化，其核心优势包括：
- **声明式编程**：代码更简洁、易读。你只需说明"要做什么"（如`filter`, `map`, `collect`），而不是"如何做"（写循环和if条件）。
- **无缝并行**：只需将`stream()`改为`parallelStream()`，就能（在数据量大且操作耗时时）尝试利用多核优势，简化了并发编程的复杂性。
- **链式操作**：可以将多个中间操作（如`filter`, `map`, `sorted`）和一个终端操作（如`collect`, `forEach`）连接起来，形成流畅的管道（Pipeline），逻辑清晰。
- **优化潜力**：Stream API 在内部可能进行延迟执行、短路等优化，性能可能优于传统的循环。
**示例：找出一个列表中所有偶数，去重后转换为平方，并收集到新列表。
```java
// 传统方式
List<Integer> numbers = Arrays.asList(1, 2, 3, 2, 4, 5);
Set<Integer> evenNumbers = new HashSet<>();
for (Integer num : numbers) {
    if (num % 2 == 0) {
        evenNumbers.add(num);
    }
}
List<Integer> squares = new ArrayList<>();
for (Integer even : evenNumbers) {
    squares.add(even * even);
}

// 使用Stream API
List<Integer> result = numbers.stream()
        .filter(n -> n % 2 == 0)
        .distinct()
        .map(n -> n * n)
        .collect(Collectors.toList());
```

##关联知识
- 

## 延伸阅读（后续补充）
- 
