# 高可用方案（MHA / MGR）

> 主从复制提供了数据冗余，但主库故障时仍需**自动检测 + 自动切换**才能真正做到高可用。MHA 是业界成熟的传统方案，MGR 是 MySQL 官方推出的原生高可用解决方案，两者各有适用场景。

---

## 1. 高可用核心指标

| 指标 | 说明 | 目标 |
|------|------|------|
| **RTO**（Recovery Time Objective） | 允许的最长宕机恢复时间 | 通常要求 < 30 秒 |
| **RPO**（Recovery Point Objective） | 允许的最大数据丢失量 | 核心业务要求 RPO=0（零丢失） |
| **自动故障转移** | 无需人工介入自动完成主从切换 | 必备能力 |

---

## 2. 传统主从 + VIP 的局限

```
传统架构：
  应用 ──▶ VIP ──▶ MySQL 主库
                        │
                   Binlog 复制
                        │
                   MySQL 从库

问题：主库故障后，需要人工：
1. 确认主库故障（排除网络闪断等误判）
2. 选择一个数据最新的从库提升为主库
3. 修改 VIP 指向或更新配置中心
4. 让其他从库指向新主库
5. 通知应用重建连接

手工操作 = 分钟级以上的停机时间
```

---

## 3. MHA（Master High Availability）

### 3.1 架构组成

```
┌─────────────────────────────────────────┐
│             MHA Manager（管理节点）        │
│  - 周期性探测主库状态                      │
│  - 触发故障切换流程                        │
│  - 可部署在独立服务器上                    │
└─────────────────────────────────────────┘
          │监控           │切换指令
          ▼               ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ MHA Node     │   │ MHA Node     │   │ MHA Node     │
│ MySQL 主库   │   │ MySQL 从库1  │   │ MySQL 从库2  │
│（候选新主库） │   │              │   │              │
└──────────────┘   └──────────────┘   └──────────────┘
```

**组件说明**：
- **MHA Manager**：独立部署，负责监控主库、协调切换。生产中通常配合 **Keepalived** 实现 Manager 自身的高可用
- **MHA Node**：部署在每台 MySQL 服务器上，负责拷贝 Binlog、切换主从关系

### 3.2 故障切换流程

```
1. Manager 检测到主库不可达（默认重试3次，约15-30秒）

2. 从各从库中选出数据最新的候选主库（比较 binlog position 或 GTID）

3. 补偿差异 Binlog：
   - 从主库（如果未彻底崩溃）SSH 拷贝未同步的 Binlog
   - 或从其他从库获取差异数据
   - 应用差异到候选主库，确保数据最新

4. 候选主库提升为新主库（RESET SLAVE ALL，RESET MASTER）

5. 其他从库指向新主库（CHANGE MASTER TO）

6. 更新 VIP 或向配置中心/注册中心通知新主库地址

整个过程：20-60 秒（依赖网络状况和数据差距）
```

### 3.3 配置要点

```ini
# manager.conf
[server default]
manager_workdir=/var/log/masterha
manager_log=/var/log/masterha/manager.log
master_binlog_dir=/var/log/mysql
master_ip_failover_script=/etc/mha/master_ip_failover   # VIP 切换脚本
ssh_user=mysql
repl_user=repl
repl_password=replpass
ping_interval=1

[server1]
hostname=192.168.1.10
candidate_master=1    # 优先作为新主库
port=3306

[server2]
hostname=192.168.1.11
port=3306

[server3]
hostname=192.168.1.12
no_master=1           # 不参与主库竞选（纯读库）
port=3306
```

```bash
# 检查 MHA 配置
masterha_check_ssh --conf=/etc/mha/manager.conf
masterha_check_repl --conf=/etc/mha/manager.conf

# 启动 Manager（后台运行）
nohup masterha_manager --conf=/etc/mha/manager.conf &

# 查看状态
masterha_check_status --conf=/etc/mha/manager.conf
```

### 3.4 MHA 的优缺点

| 优点 | 缺点 |
|------|------|
| 故障切换速度快（通常 30 秒内） | Manager 单点，需 Keepalived 冗余 |
| 成熟稳定，社区案例丰富 | 需要 SSH 免密互通，安全配置复杂 |
| 对现有主从拓扑侵入小 | 切换后需手动重新搭建原主库为从库 |
| 支持 Binlog 差异补偿，数据丢失风险低 | 不能完全保证 RPO=0（异步复制下） |

**推荐配套**：
- 半同步复制（降低数据丢失风险）
- GTID 复制（简化切换后从库重定向）
- Keepalived（Manager 高可用）
- VIP 漂移脚本（应用无感知切换）

---

## 4. MGR（MySQL Group Replication）

### 4.1 核心原理

MGR 是 MySQL 官方（5.7.17+）基于 **Paxos 分布式一致性协议** 实现的原生高可用方案。

```
MGR 复制组（3节点示例）：

  ┌──────────┐    组通信协议    ┌──────────┐
  │  Node1   │◀─────────────▶│  Node2   │
  │（主节点）  │                │（从节点）  │
  └──────────┘                └──────────┘
       ▲                           ▲
       │         ┌──────────┐      │
       └────────▶│  Node3   │◀─────┘
                 │（从节点）  │
                 └──────────┘

事务提交流程：
1. Node1 收到写请求，将事务 Write Set 广播给所有节点
2. 组内超过半数节点（≥ N/2+1）确认认证通过
3. 所有节点按相同顺序提交事务
4. 少数节点（如 Node3）未确认，事务回滚
```

**Write Set 认证机制**：
- 每个事务包含其修改的**行级 Write Set**（主键集合）
- 组内每个节点检查自己的 Write Set 队列，判断是否有冲突
- 无冲突 → 认证通过，可提交；有冲突 → 回滚冲突事务（后提交的失败）

### 4.2 单主模式 vs 多主模式

| 维度 | **单主模式（Single-Primary）** | **多主模式（Multi-Primary）** |
|------|-------------------------------|------------------------------|
| 写入节点 | 仅1个主节点可写，其余只读 | 所有节点均可写 |
| 冲突概率 | 低（无并发写冲突） | 高（需 Write Set 冲突检测） |
| 自动切换 | 主节点故障，从组内自动选举新主 | 任一节点故障，其余节点继续服务 |
| 适用场景 | 绝大多数 OLTP 场景（**推荐**） | 多活写入场景（如多机房） |
| 局限 | 写请求仍是单点瓶颈 | 跨节点写同一行会产生冲突，吞吐受限 |

### 4.3 MGR 配置（单主模式）

```ini
# my.cnf（三个节点各自配置，server_id 不同）
[mysqld]
# 基础复制配置
server_id = 1              # 每节点唯一
gtid_mode = ON
enforce_gtid_consistency = ON
log_bin = ON
binlog_format = ROW
log_replica_updates = ON   # 从库也记录 Binlog（MGR 必须）

# MGR 插件配置
plugin_load_add = 'group_replication.so'
group_replication_group_name = "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"  # UUID，所有节点相同
group_replication_start_on_boot = OFF
group_replication_local_address = "192.168.1.10:33061"   # 本节点 MGR 通信地址
group_replication_group_seeds = "192.168.1.10:33061,192.168.1.11:33061,192.168.1.12:33061"
group_replication_bootstrap_group = OFF  # 仅第一次启动第一个节点时设为 ON
```

```sql
-- 安装 MGR 插件
INSTALL PLUGIN group_replication SONAME 'group_replication.so';

-- 创建复制用户（三节点都执行）
CREATE USER 'repl'@'%' IDENTIFIED BY 'replpass';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

CHANGE MASTER TO MASTER_USER='repl', MASTER_PASSWORD='replpass' 
  FOR CHANNEL 'group_replication_recovery';

-- 第一个节点引导启动
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;

-- 其他节点加入
START GROUP_REPLICATION;

-- 查看组成员状态
SELECT * FROM performance_schema.replication_group_members;
```

### 4.4 MGR 的优缺点

| 优点 | 缺点 |
|------|------|
| **原生高可用**，故障自动检测和选举 | **对网络要求极高**：延迟 > 5ms 会明显影响性能 |
| **数据强一致**（Paxos 协议保证） | 组内节点数建议 3/5/7 奇数，最多9个 |
| 内置冲突检测，无需额外工具 | 多主模式下冲突处理复杂，不适合大量并发写同一行 |
| 自动成员管理（加入/退出自动重新配置） | 大事务会增加组通信压力，可能导致节点驱逐 |
| 支持 MySQL Router 实现应用透明访问 | 不支持外键级联操作（多主模式） |

### 4.5 MGR 生产注意事项

```sql
-- 监控组成员状态
SELECT MEMBER_ID, MEMBER_HOST, MEMBER_PORT, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
-- MEMBER_STATE: ONLINE（正常）/ RECOVERING（同步中）/ UNREACHABLE（不可达）/ ERROR

-- 查看组复制延迟（各节点 Apply Queue 积压）
SELECT * FROM performance_schema.replication_group_member_stats\G

-- 查看冲突事务数
SELECT COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE,
       COUNT_CONFLICTS_DETECTED
FROM performance_schema.replication_group_member_stats;
```

**大事务处理**（超过 `group_replication_transaction_size_limit` 会被驱逐）：
```sql
-- 查看并调整大事务限制（默认 150MB）
SHOW VARIABLES LIKE 'group_replication_transaction_size_limit';
-- 业务层需将大批量操作分批执行
```

---

## 5. MHA vs MGR 选型对比

| 维度 | MHA | MGR |
|------|-----|-----|
| **数据一致性** | 依赖半同步复制，RPO 近似 0 | Paxos 保证强一致，RPO=0 |
| **故障切换** | 20-60 秒，需 VIP 脚本 | 自动选举，通常 < 10 秒 |
| **运维复杂度** | 中（需要 SSH 免密、VIP 脚本等） | 中（网络和节点数要求严格） |
| **成熟度** | 高（生产验证多年） | 中（5.7.17+，8.0 更稳定） |
| **MySQL 版本要求** | 5.5+（建议 5.6+） | 5.7.17+（建议 8.0） |
| **多写支持** | 不支持 | 支持（多主模式） |
| **推荐场景** | 传统架构，追求稳定可控 | 新项目，MySQL 8.0，追求原生 HA |

---

## 6. 其他高可用方案简述

### 6.1 Orchestrator

- GitHub 开源，Go 语言实现
- 自动发现和可视化 MySQL 复制拓扑
- 支持自动故障转移，内置 Raft 选举保证 Orchestrator 自身高可用
- 与 MHA 类似，但更现代化，支持更复杂拓扑

### 6.2 MySQL InnoDB Cluster

- MGR + MySQL Router + MySQL Shell 的完整套件
- MySQL Router 提供应用侧的透明读写路由
- MySQL Shell 提供 AdminAPI，简化集群管理操作
- 推荐用于新建的 MySQL 8.0 生产环境

```bash
# MySQL Shell 创建 InnoDB Cluster（示例）
mysqlsh root@192.168.1.10

\js
var cluster = dba.createCluster('myCluster');
cluster.addInstance('root@192.168.1.11');
cluster.addInstance('root@192.168.1.12');
cluster.status();  # 查看集群状态
```

### 6.3 PXC（Percona XtraDB Cluster）

- 基于 Galera 协议实现的同步多主集群
- 写操作在所有节点同步执行（强一致性）
- 适合要求真正多活写入的场景
- 注意：写入吞吐受限于最慢节点

---

## 7. 高可用方案选型建议

```
新项目（MySQL 8.0）：
  └─ 首选 InnoDB Cluster（MGR + MySQL Router）
  └─ 简单场景可用 MGR 单主 + Keepalived + MySQL Router

老项目（MySQL 5.6/5.7）：
  └─ 首选 MHA + 半同步复制 + GTID + Keepalived

多活写入场景：
  └─ PXC 或 MGR 多主模式（需谨慎处理冲突）

云原生场景：
  └─ 使用云厂商托管 RDS（如腾讯云 TDSQL），内置高可用，无需自建
```

---

**相关面试题** → [[../../10_Developlanguage/002_SQL/01_MySQLSubject/10、高可用与主从复制|📖]]

**相关知识点** → [[02、主从复制原理|主从复制原理]] | [[../05_架构与运维/01、分库分表|分库分表]]
