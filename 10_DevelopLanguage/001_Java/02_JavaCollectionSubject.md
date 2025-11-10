### 一、集合框架基础

###### 1. 简单说说 List, Set, Map 三者的区别？
###### 2. 说说 Collection 和 Collections 的区别？
###### 3. Java 集合框架的层次结构是怎样的？
###### 4. Iterable 和 Iterator 接口有什么区别？
###### 5. 如何选择合适的集合类？
###### 6. 集合和数组的区别是什么？

### 二、List 相关

###### 1. ArrayList 和 LinkedList 的区别是什么？
###### 2. ArrayList 和 Vector 的区别是什么？
###### 3. ArrayList 的扩容机制是怎样的？
###### 4. ArrayList 的初始容量是多少？
###### 5. 如何实现数组和 List 之间的转换？
###### 6. ArrayList 和 LinkedList 的使用场景分别是什么？
###### 7. ArrayList 的线程安全问题如何解决？
###### 8. 说说 CopyOnWriteArrayList 的原理和应用场景？
###### 9. LinkedList 可以作为栈和队列使用吗？如何使用？
###### 10. subList 方法返回的是什么？使用时需要注意什么？

### 三、Set 相关

###### 1. HashSet 的实现原理是什么？
###### 2. HashSet 如何保证元素不重复？
###### 3. HashSet、LinkedHashSet 和 TreeSet 有什么区别？
###### 4. TreeSet 的排序规则是什么？
###### 5. HashSet 和 HashMap 的区别是什么？
###### 6. 如何实现一个线程安全的 Set？
###### 7. CopyOnWriteArraySet 的特点是什么？
###### 8. EnumSet 是什么？有什么优势？

### 四、Map 相关

###### 1. HashMap 的实现原理是什么？
###### 2. HashMap 的 put 方法的执行流程？
###### 3. HashMap 的扩容机制是怎样的？
###### 4. HashMap 为什么线程不安全？
###### 5. HashMap 和 Hashtable 的区别？
###### 6. HashMap 和 TreeMap 的区别？
###### 7. HashMap 和 LinkedHashMap 的区别？
###### 8. ConcurrentHashMap 的实现原理？
###### 9. ConcurrentHashMap 在 JDK 1.7 和 1.8 中的区别？
###### 10. 为什么 HashMap 的负载因子是 0.75？
###### 11. HashMap 的初始容量为什么是 2 的幂次方？
###### 12. HashMap 中的 hash 方法是如何实现的？
###### 13. HashMap 如何解决哈希冲突？
###### 14. 什么时候链表会转换为红黑树？
###### 15. TreeMap 的排序规则是什么？
###### 16. Java 中的 WeakHashMap 是什么？

### 五、Queue 相关

###### 1. Queue 接口的常用方法有哪些？
###### 2. PriorityQueue 的实现原理是什么？
###### 3. ArrayDeque 和 LinkedList 作为队列有什么区别？
###### 4. BlockingQueue 有哪些实现类？各有什么特点？
###### 5. ArrayBlockingQueue 和 LinkedBlockingQueue 的区别？
###### 6. DelayQueue 的应用场景是什么？
###### 7. SynchronousQueue 的特点是什么？
###### 8. ConcurrentLinkedQueue 和 LinkedBlockingQueue 的区别？

### 六、比较器与排序

###### 1. Comparable 和 Comparator 接口的区别？
###### 2. 如何对集合进行排序？
###### 3. 如何对自定义对象进行排序？
###### 4. Collections.sort() 和 Arrays.sort() 的实现原理？

### 七、并发集合

###### 1. 说说 Java 中的并发集合有哪些？
###### 2. CopyOnWriteArrayList 的原理和应用场景？
###### 3. ConcurrentHashMap 的原理？
###### 4. 如何选择合适的并发集合？
###### 5. 为什么不推荐使用 Vector 和 Hashtable？
###### 6. Collections.synchronizedXXX 方法的原理是什么？

### 八、集合操作与工具

###### 1. Collections 工具类的常用方法有哪些？
###### 2. 如何实现集合的浅拷贝和深拷贝？
###### 3. 如何实现集合的去重？
###### 4. 如何实现两个集合的交集、并集、差集？
###### 5. Arrays.asList() 方法有什么注意事项？
###### 6. 如何将集合转换为数组？
###### 7. Stream API 对集合操作有什么优势？
###### 8. 如何遍历集合？各种方式的优缺点是什么？

### 九、fail-fast 与 fail-safe

###### 1. 什么是 fail-fast 机制？在 Java 集合中是如何实现的？
###### 2. 什么是 fail-safe 机制？
###### 3. ConcurrentModificationException 异常是什么？如何避免？
###### 4. 如何在遍历集合时删除元素？

### 十、性能与优化

###### 1. 如何提高集合的性能？
###### 2. 集合的初始容量应该如何设置？
###### 3. ArrayList 和 LinkedList 在不同场景下的性能对比？
###### 4. HashMap 的性能优化技巧有哪些？