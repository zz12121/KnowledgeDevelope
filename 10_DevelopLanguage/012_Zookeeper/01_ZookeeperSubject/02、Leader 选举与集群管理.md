# 二、Leader 选举与集群管理

> 双链：[[37_ZookeeperKnowledge/02_ZAB协议与选举机制]]

---

###### 1. Zookeeper 初始化是如何进行 Leader 选举的

集群**首次启动**时，所有节点都处于 `LOOKING` 状态，此时没有任何数据，进行全新选举。

**初始化选举流程（以 3 台为例）：**

1. **每台服务器先投票给自己**：投票格式是 `(myid, zxid)`，初始时 zxid 都是 0，所以 `(1,0)` / `(2,0)` / `(3,0)`

2. **广播自己的投票**给其他所有服务器

3. **比较投票，更新自己的投票**：
   - 先比 **zxid**，zxid 大的优先（数据最新）
   - zxid 相同再比 **myid**，myid 大的优先
   - 如果收到的选票比自己当前的票更优，就更新票并重新广播

4. **统计票数**：某个服务器获得超过半数（>N/2）的票，选举结束

5. **角色确定**：被选中的服务器进入 `LEADING` 状态，其他进入 `FOLLOWING` 状态

**举例（3台：myid=1,2,3，初始 zxid 都为0）：**
- 初始：1投给1，2投给2，3投给3
- 服务器1收到服务器2的票(2,0)，比较发现 myid 2>1，更新投票给2并广播
- 服务器1收到服务器3的票(3,0)，比较发现 myid 3>2，更新投票给3并广播
- 最终3号获得3票（全票），超过半数2，服务器3成为 Leader

---

###### 2. 如果 Leader 挂了,进入崩溃恢复,怎么选举 Leader?

Leader 挂掉后，Follower 检测到心跳超时，自动进入 `LOOKING` 状态，重新触发选举——这次不再是 zxid 全为 0 的初始状态，**谁的数据最新（zxid最大）谁优先当 Leader**。

**崩溃恢复选举流程：**

1. Follower 检测到 Leader 心跳超时（默认 `tickTime × syncLimit` 毫秒），切换为 `LOOKING`
2. 各 Follower 广播自己的投票 `(myid, zxid)`，zxid 代表自己最后处理的事务
3. 同初始化选举：先比 zxid，再比 myid，不断更新投票直到有节点获得半数以上票
4. 新 Leader 当选后，进入**数据同步阶段**，确保所有 Follower 与 Leader 数据一致
5. 同步完成后，集群恢复服务

**为什么优先选 zxid 最大的？**
因为 zxid 最大意味着处理过的事务最多、数据最新，选它当 Leader 能最大程度减少数据回滚。

---

###### 3. 选举 leader 后是怎么进行数据同步的

新 Leader 当选后，需要与所有 Follower 进行数据同步，保证一致性。ZooKeeper 根据差异大小选择不同的同步方式：

**同步策略（从轻到重）：**

| 同步方式 | 触发条件 | 内容 |
|---------|---------|------|
| **DIFF 差量同步** | Follower 的 zxid 在 Leader 的事务日志范围内 | 只发送差异的事务日志 |
| **TRUNC 回滚** | Follower 的 zxid 比 Leader 大（边缘情况：旧Leader提交了但未广播的事务）| 通知 Follower 回滚到 Leader 的 zxid |
| **SNAP 全量快照** | Follower 的 zxid 太老，不在 Leader 日志范围内 | 发送完整的内存快照 |

**同步流程：**
1. Leader 发送 `NEWLEADER` 消息，带上自己的 zxid
2. Follower 响应自己的 zxid，Leader 决定用哪种方式同步
3. 完成同步后，Follower 发送 `ACK`
4. **超过半数 Follower 完成同步**，Leader 向所有完成同步的节点发送 `UPTODATE`
5. Follower 收到 `UPTODATE` 后开始接受客户端请求，集群恢复服务

---

###### 4. Zookeeper 选举核心参数有哪些

选举相关的关键参数（在 `zoo.cfg` 中配置）：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `electionAlg` | 3 | 选举算法，3 = FastLeaderElection（唯一推荐） |
| `tickTime` | 2000ms | ZooKeeper 的基本时间单位，心跳间隔 |
| `initLimit` | 10 | Follower 初始连接 Leader 的超时：`initLimit × tickTime` = 20s |
| `syncLimit` | 5 | Follower 与 Leader 同步的超时：`syncLimit × tickTime` = 10s |
| `leaderServes` | yes | Leader 是否接受客户端连接（建议关闭，让 Leader 专注协调） |

另外还有影响选举的**动态参数**：
- **myid**：每台服务器的唯一 ID（1~255），写在 `dataDir/myid` 文件里
- **zxid**：64 位事务 ID，高 32 位是 epoch（Leader 纪元），低 32 位是事务序号

---

###### 5. 分布式集群中为什么会有 Master 主节点?

分布式系统中 Master 节点解决**协调一致性**问题——当多个节点都能处理操作时，必然出现冲突（谁先谁后？谁说了算？），Master 是这些冲突的裁判。

**有 Master 的好处：**
- **写操作集中串行化**：所有写请求经过 Master，天然保证顺序，避免并发冲突
- **全局状态管理**：Master 维护全局视图（哪些节点存活？哪些任务在跑？）
- **简化一致性模型**：客户端只需和 Master 通信写操作，逻辑简单

**缺点（单点问题）：**
- Master 成为瓶颈：所有写压力集中在一台
- Master 挂了需要重新选举，期间服务不可用

ZooKeeper 通过 **ZAB 协议**和**快速选举**把 Leader 不可用时间压缩到秒级（通常 200ms~2s），代价换来了强一致性。

---

###### 6. zk 节点宕机如何处理?

分两种情况：

**① Follower/Observer 宕机**
- 影响较小，集群依然正常工作（只要剩余节点超过半数）
- Observer 宕机：完全无影响，Observer 不参与投票
- Follower 宕机：投票节点减少，但只要满足半数以上存活，Leader 仍可提交事务
- 恢复后重新连接 Leader，经过 DIFF/SNAP 同步后重新提供服务

**② Leader 宕机**
- 所有 Follower 检测到心跳超时，进入 `LOOKING` 状态
- 触发重新选举（FastLeaderElection），通常 200ms~2s 内完成
- 新 Leader 完成数据同步后恢复服务
- 原 Leader 恢复后以 Follower 角色加入集群

**关键原则：超过半数节点宕机时，集群拒绝所有写请求（保护数据一致性），处于不可用状态。**

---

###### 7. 服务器有哪些角色

ZooKeeper 集群中有 **3 种角色**：

**Leader（领导者）**：
- 全局唯一
- 处理所有写请求（事务请求）
- 发起 Proposal，收集 ACK，提交事务
- 不建议直接接受大量客户端读请求（建议 `leaderServes=no`，让 Leader 专注协调）

**Follower（跟随者）**：
- 处理读请求
- 转发写请求给 Leader
- 参与 Leader 选举投票（占大多数）
- 参与写事务的 ACK 投票（半数以上 ACK 才提交）

**Observer（观察者）**：
- 处理读请求，不参与任何投票
- 不参与 Leader 选举
- 专为**扩展读能力**设计，增加 Observer 不影响写吞吐
- 配置方式：`server.4=hostname:2888:3888:observer`

口诀：**Leader 写+协调，Follower 读+投票，Observer 只读不说话。**

---

###### 8. Zookeeper 下 Server 工作状态

ZooKeeper 服务器的工作状态（服务视角）：

| 状态 | 含义 |
|------|------|
| **LOOKING** | 正在寻找 Leader（选举中），不对外提供服务 |
| **FOLLOWING** | 是 Follower，跟随 Leader，正常服务 |
| **LEADING** | 是 Leader，正在领导集群，正常服务 |
| **OBSERVING** | 是 Observer，同步数据，处理读请求 |

正常情况下，集群只有一个 LEADING，其余 FOLLOWING（或部分 OBSERVING）。当 Leader 挂掉，所有节点切换到 LOOKING，选举结束后恢复正常状态。

---

###### 9. Zookeeper server 有哪些状态?

这道题和第8题有所区别——这里问的是**服务器自身的运行状态**（从代码层面的 `ServerState` 枚举角度）：

实际上 ZooKeeper 源码中的 `QuorumPeer.ServerState` 就是：
- `LOOKING`：寻找 Leader 中
- `FOLLOWING`：Follower 角色
- `LEADING`：Leader 角色
- `OBSERVING`：Observer 角色

与第8题描述的四种工作状态完全一致。面试中有时也会追问**服务器的数据状态**：
- **初始化中**：刚启动，还没有加载快照和事务日志
- **数据同步中**：连接 Leader 后正在接受 DIFF/SNAP
- **正常服务中**：同步完成，正常处理请求

---

###### 10. Leader 选举的 FastLeaderElection 算法原理是什么?

FastLeaderElection（FLE）是 ZooKeeper 3.4+ 唯一使用的选举算法，核心思路：**用全连接 UDP/TCP 通信快速收敛到有 Leader 的状态**。

**核心数据结构：**
```
Vote {
    long proposedLeader;  // 提议的 Leader myid
    long proposedZxid;    // 提议 Leader 的 zxid
    long proposedEpoch;   // 提议 Leader 的 epoch（选举轮次）
}
```

**选举规则（优先级从高到低）：**
1. 比 **epoch**：epoch 更大的票优先（更新的选举轮次）
2. 比 **zxid**：zxid 更大的优先（数据更新）
3. 比 **myid**：myid 更大的优先

**收敛过程：**
1. 每台服务器启动一个 QuorumCnxManager，建立全连接网络（只和 myid 更大的节点建连接，避免双向重复）
2. 发起投票，广播给所有节点
3. 收到其他节点的票，按上述规则比较：如果对方的票"更好"，更新自己的票并重新广播
4. 每次收到投票后，检查当前的投票箱是否有节点获得 **>N/2** 票
5. 达到半数后，等待短暂时间（finalizeWait=200ms）确认没有更新的投票，然后确定结果

**为什么叫"Fast"？**
因为使用独立的选举通信端口（默认3888），不依赖 Leader/Follower 的数据同步通道，选举和数据传输并行，通常 200ms 内完成。

---

###### 11. 什么是 myid 和 epoch?

**myid：**
- 每台 ZooKeeper 服务器的**唯一标识符**，取值范围 1~255
- 存储在 `zoo.cfg` 中 `dataDir` 指定目录下的 `myid` 文件里，手动配置
- 选举时 myid 作为最后的决胜条件（zxid 相同时 myid 大的优先）
- 在 `zoo.cfg` 中配合 server 列表使用：`server.1=host1:2888:3888`（这里的1就是myid=1的机器）

**epoch（纪元）：**
- epoch 是 **Leader 的任期编号**，每次 Leader 选举成功后 epoch +1
- epoch 体现在 zxid 的高 32 位：`zxid = epoch(32bit) | counter(32bit)`
- 作用：防止旧 Leader "复活"后误以为自己还是 Leader，发出过期的事务
- 具体机制：新 Leader 当选后 epoch+1，所有 Follower 更新自己的 epoch，之后如果旧 Leader 发来 epoch 更小的请求，直接拒绝

类比：epoch 就像皇帝的年号，新皇登基改年号，旧年号发来的圣旨一律作废。

---

###### 12. Observer 角色的作用是什么?

Observer 是 ZooKeeper 3.3+ 引入的第三种角色，专门解决**扩展读能力但不破坏写性能**的问题。

**为什么需要 Observer？**
- 直接加 Follower 能扩展读能力，但会增加写吞吐压力：
  - 每次写事务需要半数以上节点 ACK
  - Follower 越多，Leader 需要等待的 ACK 越多，写延迟上升
- Observer **不参与投票**，增加再多 Observer 也不影响写吞吐

**Observer 的特点：**
- 接受读请求，直接响应
- 接受写请求，转发给 Leader
- 接受 Leader 的事务广播，与 Leader 保持数据同步
- **不参与 Leader 选举**
- **不参与写事务的 ACK 投票**

**典型使用场景：**
- 异地机房部署：把跨机房的节点配置为 Observer，避免跨机房写延迟影响 Leader 提交速度
- 读多写少场景：大量 Observer 承担读压力
- 配置：`server.4=hostname:2888:3888:observer`

---

> **追问：ZooKeeper 集群为什么推荐奇数台服务器（如3、5、7台）？**
>
> 核心原因是**容错能力与机器成本的最优比**。
>
> ZooKeeper 正常工作的条件：**存活节点 > 总节点数 / 2**，即多数派原则。
>
> | 节点总数 | 允许宕机数 | 容错率 |
> |---------|---------|------|
> | 3 | 1 | 33% |
> | 4 | 1 | 25% |
> | 5 | 2 | 40% |
> | 6 | 2 | 33% |
>
> 4 台和 3 台的容错能力完全一样（都只允许1台宕机），但 4 台多花一台机器——白花钱。
> 6 台和 5 台同理（都允许2台宕机）。
>
> 所以**偶数台没有意义**，奇数台是性价比最高的选择。
