# ConcurrentHashMap 能保证复合操作的原子性吗？

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
**ConcurrentHashMap 只能保证单个方法调用（如 `put`, `get`）的线程安全，但不能保证多个连续操作组成的"复合操作"的原子性**​。
**错误示例**：
```java
// 非原子操作，线程不安全
if (!map.containsKey(key)) {
    map.put(key, value); // 在containsKey和put之间，可能有其他线程插入了key
}
```
**解决方案**：使用 ConcurrentHashMap 提供的**原子复合操作方法**​。
- `putIfAbsent(key, value)`: 如果 key 不存在，则放入 value，并返回 null；如果存在，则不操作，返回已存在的 value。
- `computeIfAbsent(key, function)`: 如果 key 不存在，则使用 function 计算出的 value 放入 map，并返回该 value。非常适合"如果不存在则计算并添加"的场景，如懒加载缓存。
- `computeIfPresent(key, function)`: 如果 key 存在，则根据 function 重新计算 value 并更新。
- `replace(key, oldValue, newValue)`: 只有当前 key 对应的值等于 oldValue 时，才替换为 newValue。

##关联知识
- 

## 延伸阅读（后续补充）
- 
