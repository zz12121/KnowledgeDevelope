# HashMap 的实现原理是什么？

## 一句话说明（白话）

hashCode 是对象散列标识，用于加速查找。

## 它解决什么问题 / 为什么重要

HashMap/HashSet先用 hashCode 定位桶，再用 equals 判断。

## 核心原理（一步步讲清楚）

equals 相等必须 hashCode 相等。

##典型使用场景

Map key、Set 去重。

## 简单例子 /伪代码

equals 基于 id，hashCode也应基于 id。

## 常见坑与误区

hashCode 相同不代表 equals 相同。

##题库要点（原始材料）
HashMap 的底层实现是“数组 + 链表 + 红黑树”的组合。您可以将其理解为一个“桶数组”，每个桶（数组元素）可以用来存放键值对。
- **数组（桶）**：主干，用于快速定位，默认初始大小为 16。
- **链表**：解决哈希冲突。当不同的键通过哈希计算落到同一个桶时，会以链表形式存储（拉链法）。
- **红黑树**：优化长链表查询。当链表长度超过阈值（默认为8）且数组容量达到一定值（默认为64）时，链表会转为红黑树，将查询效率从 O(n) 提升到 O(log n)。
为了更直观地理解其核心操作流程，我们可以参考下面的序列图，它描绘了 `put`和 `get`方法的关键步骤：

```mermaid
sequenceDiagram
    participant U as User
    participant HM as HashMap
    participant H as Hash Function
    participant A as Array (Buckets)
    participant L as Linked List
    participant T as Red-Black Tree

    U->>HM: put(key, value) / get(key)
    HM->>H: compute hash(key)
    H->>A: index = (n-1) & hash
    Note over A: Locate the bucket

    alt put Operation
        Note over HM: Start putVal process
        A->>A: Check if bucket is empty
        alt Bucket Empty
            A-->>A: Store new Node directly
        else Bucket Not Empty (Hash Collision)
            alt Key exists in first Node
                A-->>A: Update value
            else Node is TreeNode (Red-Black Tree)
                A->>T: Call putTreeVal
            else Traverse Linked List
                loop Until end or key found
                    A->>L: Traverse next Node
                end
                alt Key found in List
                    L-->>L: Update value
                else Key not found
                    L-->>L: Create new Node at end
                    Note over L: Check list length
                    alt List length >= TREEIFY_THRESHOLD(8)<br>and Array capacity >= 64
                        L->>T: Convert to Red-Black Tree
                    end
                end
            end
        end
        Note over HM: Check if size > threshold
        alt Needs Resize
            HM->>HM: resize()
        end
    else get Operation
        A->>A: Check first Node in bucket
        alt Key matches first Node
            A-->>HM: Return value
        else Node is TreeNode (Red-Black Tree)
            A->>T: Call getTreeNode
            T-->>HM: Return value
        else Traverse Linked List
            loop Until key found or end
                A->>L: Traverse next Node
            end
            alt Key found
                L-->>HM: Return value
            else Key not found
                L-->>HM: Return null
            end
        end
    end
```

##关联知识
- 

## 延伸阅读（后续补充）
- 
