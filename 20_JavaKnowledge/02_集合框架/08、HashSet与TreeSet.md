---
tags:
  - Java/集合框架
  - Java/Set
aliases:
  - HashSet原理
  - TreeSet有序集合
  - Set去重
date: 2026-03-18
---

# HashSet 与 TreeSet

> **核心关键词**：HashSet、LinkedHashSet、TreeSet、去重原理、有序性、equals/hashCode、Comparable/Comparator

---

## 一、Set 体系概览

```
Set（不允许重复元素）
  ├── HashSet         → 基于 HashMap，无序，O(1) 操作
  │     └── LinkedHashSet → 维护插入顺序，O(1) 操作
  └── TreeSet         → 基于 TreeMap（红黑树），有序（自然顺序/自定义），O(log n)
```

---

## 二、HashSet 深度解析

### 2.1 底层结构

HashSet 本质上就是一个 **HashMap**，只用了 key，value 统一存储为一个哑元对象。

```java
// 源码核心
public class HashSet<E> extends AbstractSet<E> {
    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object();  // 哑元 value

    public boolean add(E e) {
        return map.put(e, PRESENT) == null;  // key 就是元素
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }

    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;
    }
}
```

### 2.2 去重原理（equals + hashCode）

```java
// HashSet 判断"元素相同"的逻辑：
// 1. 先比较 hashCode（快速过滤）
// 2. hashCode 相同时再比较 equals（精确判断）
// 两者都相同 → 认为是同一元素，不重复添加

// 关键规则（必须遵守）：
// equals 为 true → hashCode 必须相同
// hashCode 相同 → equals 不一定为 true（哈希冲突）

// 反例：不重写 hashCode 导致的问题
public class User {
    String name;
    // 只重写了 equals，没重写 hashCode
    @Override
    public boolean equals(Object o) {
        return o instanceof User u && name.equals(u.name);
    }
    // 默认 hashCode 是对象地址，不同实例 hashCode 不同！
}

Set<User> set = new HashSet<>();
set.add(new User("张三"));
set.add(new User("张三"));  // 会被当成不同元素加入！！！
System.out.println(set.size());  // 2，而不是期望的 1

// 正确做法：使用 IDE 自动生成，或 Lombok @EqualsAndHashCode
@Override
public int hashCode() {
    return Objects.hash(name);
}
```

### 2.3 初始容量与扩容

与 HashMap 完全一致（因为底层就是 HashMap）：
- 默认初始容量 16，负载因子 0.75
- 超过 `capacity * loadFactor` 时扩容为 2 倍

---

## 三、LinkedHashSet

```java
// LinkedHashSet = LinkedHashMap 作底层
// 特点：保持元素的插入顺序，迭代时按插入顺序输出
// 性能：比 HashSet 略慢（维护链表指针），比 TreeSet 快

LinkedHashSet<String> lhs = new LinkedHashSet<>();
lhs.add("banana");
lhs.add("apple");
lhs.add("cherry");
lhs.add("apple");  // 重复，不加入

System.out.println(lhs);  // [banana, apple, cherry]（保持插入顺序）

// 应用场景：需要去重且保持插入顺序时使用
```

---

## 四、TreeSet 深度解析

### 4.1 底层结构与特性

```java
// TreeSet 底层是 TreeMap（红黑树）
// 特点：
//   ① 元素按自然顺序（Comparable）或自定义顺序（Comparator）排序
//   ② 不允许 null（会抛 NullPointerException）
//   ③ 增删查 O(log n)

TreeSet<Integer> ts = new TreeSet<>();
ts.add(5); ts.add(2); ts.add(8); ts.add(1);
System.out.println(ts);  // [1, 2, 5, 8]（自然升序）
```

### 4.2 自然排序（Comparable）

```java
// 元素类必须实现 Comparable 接口
public class Student implements Comparable<Student> {
    String name;
    int score;

    @Override
    public int compareTo(Student other) {
        // 按分数降序，分数相同按姓名升序
        int cmp = Integer.compare(other.score, this.score);
        return cmp != 0 ? cmp : this.name.compareTo(other.name);
    }
}

TreeSet<Student> students = new TreeSet<>();
students.add(new Student("张三", 90));
students.add(new Student("李四", 85));
// 按 compareTo 结果排序
```

### 4.3 自定义排序（Comparator）

```java
// 构造时传入 Comparator，不要求元素实现 Comparable
TreeSet<String> ts = new TreeSet<>(Comparator.comparingInt(String::length)
    .thenComparing(Comparator.naturalOrder()));  // 先按长度，再按字典序

// 注意：TreeSet 的去重判断依赖 compareTo/compare 的返回值
// compare 返回 0 → 认为是同一元素（不插入），与 equals 无关！
TreeSet<String> set = new TreeSet<>(Comparator.comparingInt(String::length));
set.add("abc");
set.add("def");  // compareTo 结果为 0（长度相同），不插入！
System.out.println(set);  // [abc]
```

### 4.4 TreeSet 的导航方法（NavigableSet）

```java
TreeSet<Integer> ts = new TreeSet<>(Arrays.asList(1, 3, 5, 7, 9));

ts.first()         // 1（最小）
ts.last()          // 9（最大）
ts.floor(4)        // 3（≤4 的最大值）
ts.ceiling(4)      // 5（≥4 的最小值）
ts.lower(5)        // 3（<5 的最大值）
ts.higher(5)       // 7（>5 的最小值）
ts.headSet(5)      // [1, 3]（<5 的子集）
ts.tailSet(5)      // [5, 7, 9]（≥5 的子集）
ts.subSet(3, 7)    // [3, 5]（[3,7) 的子集）
ts.pollFirst()     // 移除并返回最小元素
ts.pollLast()      // 移除并返回最大元素
ts.descendingSet() // 反向视图
```

---

## 五、三种 Set 对比

| 特性 | HashSet | LinkedHashSet | TreeSet |
|------|---------|--------------|---------|
| 底层 | HashMap | LinkedHashMap | TreeMap（红黑树） |
| 顺序 | 无序 | 插入顺序 | 排序（自然/自定义） |
| null | 允许 1 个 | 允许 1 个 | ❌ 不允许 |
| 增删查 | O(1) | O(1) | O(log n) |
| 去重依据 | equals + hashCode | equals + hashCode | compareTo/compare |
| 适用场景 | 通用去重 | 去重 + 保序 | 去重 + 排序/范围查询 |

---

## 六、实战场景

```java
// 场景1：统计文章中出现过的单词（去重）
Set<String> words = new HashSet<>(Arrays.asList(text.split("\\s+")));

// 场景2：保持访问记录的唯一性且有序
LinkedHashSet<String> visited = new LinkedHashSet<>();
visited.add("page1"); visited.add("page2"); visited.add("page1");
// → [page1, page2]（去重 + 保持访问顺序）

// 场景3：实时排行榜（分数去重 + 有序）
TreeSet<Integer> scores = new TreeSet<>(Comparator.reverseOrder());
scores.add(90); scores.add(85); scores.add(95);
System.out.println(scores.first());  // 95（最高分）

// 场景4：查找某范围内的元素
TreeSet<Integer> primes = new TreeSet<>(Arrays.asList(2,3,5,7,11,13,17,19));
System.out.println(primes.subSet(5, 15));  // [5, 7, 11, 13]
```

---

## 七、面试要点速查

| 问题 | 要点 |
|------|------|
| HashSet 底层实现 | HashMap，元素作为 key，value 统一用哑元 PRESENT |
| HashSet 去重原理 | 先比 hashCode，再比 equals，两者都相同才认为重复 |
| 只重写 equals 不重写 hashCode 会怎样 | HashSet 无法正确去重（hashCode 不同时直接认为不同元素） |
| TreeSet 去重依据 | compareTo/compare 返回 0 就认为相同，与 equals 无关 |
| TreeSet 为什么不允许 null | 排序时调用 compareTo，null 调用会抛 NPE |
| LinkedHashSet 与 HashSet 区别 | 多了双向链表维护插入顺序 |
| TreeSet 如何实现范围查询 | 实现了 NavigableSet 接口，提供 subSet/headSet/tailSet 等方法 |
| hashCode 设计原则 | 相等对象必须有相同 hashCode；不相等对象 hashCode 尽量不同（减少哈希碰撞）|
| 为什么 Objects.hash() 使用 31 作乘子 | 31 = 2^5 - 1，可被 JIT 优化为移位+减法；且足够大，碰撞少 |


---

**相关面试题** → [[../../10_Developlanguage/001_Java/02_JavaCollectionSubject/03、Set 相关|03、Set 相关]]
