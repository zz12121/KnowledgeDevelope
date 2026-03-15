# 02 ZAB 协议与选举机制

> 关联面试题：[[10_DevelopLanguage/012_Zookeeper/01_ZookeeperSubject/02、Leader 选举与集群管理]] | [[10_DevelopLanguage/012_Zookeeper/01_ZookeeperSubject/03、数据同步与一致性]]
> 
> 延伸阅读：[[33_DistributedKnowledge/02_分布式共识算法]]

---

## 一、ZAB 协议概述

**ZAB（ZooKeeper Atomic Broadcast）** 是 ZooKeeper 专用的分布式一致性协议，保证：
- **可靠提交**：事务在一台机器提交，最终所有机器都提交
- **全局有序**：所有机器以相同顺序应用所有事务

ZAB 分两个阶段交替运行：

```
集群启动 / Leader 宕机
        ↓
[崩溃恢复阶段]：Leader 选举 → 数据同步
        ↓ 超过半数节点同步完成
[消息广播阶段]：正常处理写请求
        ↓ Leader 失联
[崩溃恢复阶段]：重新开始
```

---

## 二、崩溃恢复阶段

### 2.1 Leader 选举（FastLeaderElection）

**选举规则（优先级从高到低）：**
1. epoch 更大 → 更新的 Leader 任期
2. zxid 更大 → 更新的数据
3. myid 更大 → 作为最后决胜条件

**初始化选举（全为0，按 myid 决胜）：**

```
5台服务器（myid=1,2,3,4,5），初始 zxid 全为0：
1→投给1, 2→投给2, 3→投给3, 4→投给4, 5→投给5
各节点不断更新投票（选 myid 最大的），最终 5 获全票当选
```

**崩溃恢复选举（zxid 决胜）：**
- 各 Follower 广播自己的 `(myid, zxid)`
- zxid 最大的节点优先当选（数据最新）
- 防止已提交数据丢失

### 2.2 数据同步（三种方式）

| 同步方式 | 触发条件 | 操作 |
|---------|---------|------|
| **DIFF 差量同步** | Follower zxid 在 Leader 日志范围内 | 只发送差异事务 |
| **TRUNC 截断** | Follower 有多余事务（zxid > Leader） | 通知 Follower 回滚 |
| **SNAP 全量快照** | Follower 太旧，超出日志范围 | 发送完整内存快照 |

**同步完成条件：** 超过半数节点完成同步，Leader 广播 UPTODATE，集群开始服务

---

## 三、消息广播阶段（ZAB 两阶段提交）

```
客户端写请求 → Leader
                 ↓
          ① 生成 ZXID，写 WAL
          ② 广播 Proposal 给所有 Follower（FIFO 队列，保序）
                 ↓
         Follower 写 WAL，发送 ACK
                 ↓
     ③ 收到 >N/2 个 ACK → 提交事务
          ④ 广播 COMMIT 给所有 Follower
                 ↓
         Follower 应用事务到内存
          ⑤ 响应客户端写成功
```

**与标准 2PC 的区别：**
- 标准 2PC：所有参与者确认才提交（一个失败则全部失败）
- ZAB：超过半数确认即提交，容忍少数节点失败

---

## 四、ZXID 结构

```
ZXID（64位）：
  高32位（epoch）：Leader 任期编号，每次新选举 +1
  低32位（counter）：当前 Leader 内的事务序号，单调递增
```

**epoch 的防脑裂作用：**
- 旧 Leader 复活后发现自己 epoch 小，主动降级为 Follower
- 所有节点拒绝接受 epoch 更小的 Leader 发出的 Proposal

---

## 五、集群角色与工作状态

| 角色 | 参与选举 | 参与 ACK 投票 | 处理读请求 | 处理写请求 |
|------|---------|-------------|---------|---------|
| **Leader** | 当选者 | ✅ | ✅ | ✅（直接处理） |
| **Follower** | ✅ | ✅ | ✅ | 转发给 Leader |
| **Observer** | ❌ | ❌ | ✅ | 转发给 Leader |

**节点工作状态（ServerState）：**
- `LOOKING`：选举中，不对外服务
- `FOLLOWING`：Follower，正常服务
- `LEADING`：Leader，正常服务
- `OBSERVING`：Observer，正常服务

---

## 六、集群可用性与奇数原则

**核心公式：** 存活节点数 > 总节点数 / 2

| 节点数 | 允许宕机 | 成本效益 |
|-------|---------|---------|
| 3 | 1 | ✅ 最小可用集群 |
| **4** | **1** | ❌ 和3台一样，多花一台 |
| 5 | 2 | ✅ 生产推荐 |
| **6** | **2** | ❌ 和5台一样，多花一台 |
| 7 | 3 | ✅ 高可用要求高时 |

**推荐：生产环境使用 3 台（小规模）或 5 台（大规模）。**

---

## 七、ZAB vs Raft vs Paxos

| 维度 | ZAB | Raft | Paxos |
|------|-----|------|-------|
| 设计目标 | ZooKeeper 专用 | 通用、易理解 | 通用理论算法 |
| Leader 选举 | epoch+zxid+myid | term+lastLogIndex | ballot number |
| 日志连续性 | 必须连续 | 必须连续 | 允许空洞 |
| 成员变更 | 静态为主（3.5+ 支持动态） | Joint Consensus | 复杂 |
| 主要用途 | ZooKeeper | etcd、TiKV、Consul | 理论参考 |

---

## 八、脑裂防护机制

两道保险：
1. **半数以上原则**：任意分区最多一侧能达到多数派，只有一侧能选出 Leader
2. **epoch 机制**：旧 Leader 复活后 epoch 更小，主动退位，所有节点拒绝旧 epoch 的请求

```
5台发生 3+2 分区：
  3台侧 → 可选出 Leader（3 > 2.5），正常服务
  2台侧 → 无法选 Leader（2 ≤ 2.5），进入 LOOKING，拒绝写请求
```
