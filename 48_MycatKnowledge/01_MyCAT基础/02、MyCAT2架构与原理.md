# MyCAT2 架构与原理

## 2.1 MyCAT2 核心概念

### 逻辑库（Logical Database）

在 MyCAT2 中，逻辑库是一个抽象的概念，它并不是真实存在的数据库，而是一个对多个实际数据库的映射。当客户端连接到 MyCAT2 时，会选择一个逻辑库进行操作，MyCAT2 会根据配置将操作分发到对应的实际数据库上执行。

### 逻辑表（Logical Table）

同样，逻辑表也是一个抽象的概念，它并不是真实存在的表，而是一个对多个实际表的映射。

### 物理库（Physical Database）

MySQL 中真实存在的数据库。

### 物理表（Physical Table）

MySQL 中真实存在的表。

### 拆分键（Shard Key）

描述拆分逻辑表的数据规则的字段。

### 表类型

| 类型 | 说明 |
|------|------|
| **单表** | 没有分片，没有数据冗余的表 |
| **全局表/广播表** | 每个数据库实例都冗余全量数据的逻辑表，通过数据冗余使分片表的分区与该表的数据在同一个数据库实例里，达到 join 运算能够直接在该数据库实例里执行 |
| **ER表** | 狭义指父子表中的子表，它的分片键指向父表的分片键，而且两表的分片算法相同；广义指具有相同数据分布的一组表 |

### 集群（Cluster）

多个数据节点组成的逻辑节点。在 MyCAT2 里，它是把对多个数据源地址视为一个数据源地址（名称），并提供自动故障恢复、转移，即实现高可用、负载均衡的组件。

### 数据源（Datasource）

连接后端数据库的组件，它是数据库代理中连接后端数据库的客户端。

### Schema

在 MyCAT2 中配置表逻辑、视图等的配置。

---

## 2.2 配置文件结构

### 整体目录

```
mycat配置文件夹/
├── clusters/                    # 集群配置
│   ├── prototype.cluster.json   # 无集群时自动创建
│   ├── c0.cluster.json
│   └── c1.cluster.json
├── datasources/                # 数据源配置
│   ├── prototypeDs.datasource.json  # 无数据源时自动创建
│   ├── dr0.datasource.json
│   └── dw0.datasource.json
├── schemas/                    # 逻辑库表配置
│   ├── db1.schema.json
│   └── mysql.schema.json
├── sequences/                  # 序列配置
│   └── db1_schema.sequence.json
├── server.json                 # 服务器配置
└── state.json                  # 运行状态
```

### 数据源配置（datasource）

```json
{
    "dbType": "mysql",                    // 数据库类型
    "idleTimeout": 60000,                  // 空闲连接超时
    "initSqls": [],                        // 初始化SQL
    "initSqlsGetConnection": true,         // 每次获取连接是否执行initSql
    "instanceType": "READ_WRITE",          // 实例类型：READ_WRITE/READ/WRITE
    "maxCon": 1000,                        // 最大连接数
    "maxConnectTimeout": 3000,              // 最大连接超时
    "maxRetryCount": 5,                     // 最大重试次数
    "minCon": 1,                           // 最小连接数
    "name": "prototypeDs",                 // 数据源名称
    "password": "root",                    // 密码
    "type": "JDBC",                        // 数据源类型
    "url": "jdbc:mysql://localhost:3306/xxx",  // JDBC连接地址
    "user": "root",                        // 用户名
    "weight": 0                            // 负载均衡权重
}
```

### 集群配置（cluster）

```json
{
    "clusterType": "MASTER_SLAVE",         // 集群类型
    "heartbeat": {                         // 心跳配置
        "heartbeatTimeout": 1000,          // 心跳超时
        "maxRetry": 3,                     // 最大重试
        "minSwitchTimeInterval": 300,       // 最小切换间隔
        "slaveThreshold": 0                // 从节点延迟阈值
    },
    "masters": ["dw0"],                    // 主节点列表
    "maxCon": 200,                         // 最大连接数
    "name": "prototype",                   // 集群名称
    "readBalanceType": "BALANCE_ALL",     // 负载均衡策略
    "switchType": "SWITCH"                 // 切换类型
}
```

**集群类型**：
- `SINGLE_NODE`：单一节点
- `MASTER_SLAVE`：普通主从
- `GARELA_CLUSTER`：Garela Cluster / PXC 集群
- `MHA`：MHA 集群（自动化主从切换和故障恢复）
- `MGR`：MGR 集群

**负载均衡策略**：
- `BALANCE_ALL`：获取集群中所有数据源
- `BALANCE_ALL_READ`：获取集群中允许读的数据源
- `BALANCE_READ_WRITE`：获取集群中允许读写的数据源，但允许读的数据源优先
- `BALANCE_NONE`：获取集群中允许写数据源，即主节点中选择

### 用户配置（user）

```json
{
    "dialect": "mysql",                     // 使用语言
    "ip": null,                            // 客户端访问IP限制
    "password": "123456",                  // 密码
    "transactionType": "proxy",            // 事务类型：proxy/xa
    "username": "root"                     // 用户名
}
```

**事务类型**：
- `proxy`：本地事务，涉及大于1个数据库的事务在commit阶段失败会导致不一致，但兼容性最好
- `xa`：XA事务，需要确认存储节点集群类型是否支持XA

> 注意：不涉及跨库事务请把事务类型改成proxy，不要使用XA

### 服务配置（server.json）

```json
{
    "loadBalance": {
        "defaultLoadBalance": "BalanceRandom",
        "loadBalances": []
    },
    "mode": "local",
    "server": {
        "bufferPool": {},
        "idleTimer": {
            "initialDelay": 3,
            "period": 60000,
            "timeUnit": "SECONDS"
        },
        "ip": "0.0.0.0",
        "mycatId": 1,
        "port": 8066,                      // MyCAT2 端口
        "reactorNumber": 8,                // 反应器线程数
        "workerPool": {                    // 工作线程池
            "corePoolSize": 1,
            "maxPoolSize": 1024,
            "keepAliveTime": 1,
            "taskTimeout": 5,
            "timeUnit": "MINUTES"
        }
    }
}
```

---

## 2.3 MyCAT2 原理详解

### SQL 处理流程

```
客户端请求
    ↓
SQL 拦截（MyCAT 接收 SQL）
    ↓
SQL 解析（Parser 解析 SQL）
    ↓
路由计算（根据分片规则计算目标数据源）
    ↓
SQL 转发（向目标数据库发送 SQL）
    ↓
结果归并（合并多个结果集返回客户端）
```

### 负载均衡策略

| 策略 | 说明 |
|------|------|
| BalanceLeastActive | 最少正在使用的连接数的 MySQL 数据源被选中 |
| BalanceRandom | 随机算法产生随机数，从活跃的数据源中选取 |
| BalanceRoundRobin | 加权轮询算法 |
| BalanceRunOnReplica | 把请求尽量发往从节点，不会发到不可读与不可用的从节点 |

---

⬆️ [[索引]]
