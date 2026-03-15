# 持久化机制

## 1. RDB 持久化

RDB（Redis Database）是基于**内存快照**的持久化机制，将某个时间点的内存数据保存为二进制文件（默认 `dump.rdb`）。

### 触发方式

**手动触发**：
- `SAVE`：阻塞主线程，不推荐在生产环境使用
- `BGSAVE`：fork 子进程在后台生成 RDB，主线程继续服务

**自动触发**（配置文件中 `save` 指令）：
```properties
save 900 1      # 900秒内至少1个key变化
save 300 10     # 300秒内至少10个key变化
save 60 10000   # 60秒内至少10000个key变化
```

**其他触发**：执行 `SHUTDOWN`、`FLUSHALL` 或主从全量复制时。

### BGSAVE 核心机制

```c
int rdbSaveBackground(char *filename) {
    pid_t childpid;
    if ((childpid = fork()) == 0) {
        // 子进程：执行实际 RDB 文件生成
        int retval = rdbSave(filename);
        exit((retval == C_OK) ? 0 : 1);
    } else {
        // 父进程：记录 fork 时间，继续处理请求
        server.stat_fork_time = ustime() - start;
        return C_OK;
    }
}
```

**写时复制（Copy-on-Write）**：fork 后父子进程共享物理内存页。只有父进程修改某页时，操作系统才为子进程创建该页的副本。极大减少内存开销，子进程看到的是 fork 时刻的内存快照。

### RDB 文件结构

```
REDIS (魔数) | RDB版本号 | 辅助字段 | 数据库数据 | EOF标志 | CRC64校验和
```

数据库数据包含：选择DB命令 + 键值对（含过期时间）。

### 优缺点

**优点**：
- 文件紧凑，加载速度快（直接反序列化二进制数据）
- 对主线程影响小（子进程执行）
- 适合全量备份和灾难恢复

**缺点**：
- 数据安全性低，两次快照之间宕机会丢失数据
- fork 时有短暂阻塞，内存越大阻塞越长
- RDB 不同版本间可能存在兼容性问题

---

## 2. AOF 持久化

AOF（Append Only File）通过**记录写操作命令**来保证持久性，每次写命令追加到 AOF 文件。

### 工作流程

1. **命令传播**：客户端写命令执行后，命令追加到 `aof_buf` 缓冲区
2. **文件同步**：根据 `appendfsync` 配置将缓冲区内容刷入磁盘
3. **文件重写**：定期对 AOF 文件重写，消除冗余命令

### AOF 文件格式（RESP 协议）

```
*3          # 命令有3个参数
$3          # 第1个参数长度3
SET
$5          # 第2个参数长度5
mykey
$7          # 第3个参数长度7
myvalue
```

### appendfsync 三种策略

| 策略 | 机制 | 数据安全性 | 性能 | 适用场景 |
|------|------|-----------|------|---------|
| **always** | 每命令都调用 `fsync` | 最高（最多丢失1条命令） | 最低 | 金融交易 |
| **everysec** | 后台线程每秒 `fsync` 一次 | 较高（最多丢失1秒数据） | 较高 | **大多数场景推荐** |
| **no** | 不主动 `fsync`，由 OS 决定 | 较低 | 最高 | 可容忍数据丢失的缓存 |

### AOF 重写机制

AOF 重写不是分析旧文件，而是**遍历当前数据库状态**，为每个键生成最小命令集：

```
# 原始AOF可能包含：
SET counter 1
INCR counter
INCR counter
INCR counter

# 重写后只需：
SET counter 4
```

**重写过程（BGREWRITEAOF）**：
1. 主进程 fork 子进程执行重写
2. 子进程遍历数据库，生成新 AOF 文件
3. 主进程同时将新写命令写入 **AOF 缓冲区** 和 **AOF 重写缓冲区**
4. 子进程完成后，主进程将重写缓冲区内容追加到新文件
5. 原子性地用新文件替换旧文件

**自动触发条件**：
```properties
auto-aof-rewrite-percentage 100   # AOF文件增长超过上次重写后大小的100%
auto-aof-rewrite-min-size 64mb   # AOF文件最小64MB才触发
```

---

## 3. 混合持久化（Redis 4.0+）

混合持久化结合了 RDB 的快速恢复和 AOF 的数据安全。

**启用配置**：
```properties
appendonly yes
aof-use-rdb-preamble yes
```

**文件结构**：AOF 文件头部是 RDB 格式的全量快照，后续是 AOF 格式的增量命令。
```
[RDB全量数据] + [增量AOF命令]
```

**优势**：
- 恢复时先加载 RDB 部分（快速），再重放少量 AOF 命令（精确）
- 兼具 RDB 的快速恢复和 AOF 的数据安全

---

## 4. RDB vs AOF 对比

| 特性 | RDB | AOF |
|------|-----|-----|
| **数据安全** | 可能丢失最后一次快照后的所有数据 | 最多丢失1秒（everysec 策略） |
| **恢复速度** | 快（直接加载二进制） | 慢（需逐条重放命令） |
| **磁盘占用** | 小（二进制压缩） | 大（文本格式，需重写优化） |
| **性能影响** | fork 时短暂阻塞 | 持续写入，fsync 策略影响性能 |
| **可读性** | 差（二进制） | 好（文本，可手工修改修复） |

**选择策略**：
- **纯缓存场景**：仅 RDB，追求最高性能
- **生产业务**：RDB + AOF（everysec），平衡性能与安全
- **金融交易**：AOF（always），数据绝对不能丢失
- **Redis 4.0+**：推荐混合持久化

---

## 5. 数据安全最佳实践

- **多机房备份**：定期将 RDB/AOF 文件备份到异地存储
- **监控持久化状态**：`INFO persistence` 监控 `rdb_last_bgsave_status`、`aof_last_bgrewrite_status`
- **控制实例内存**：单实例建议不超过 20GB，减少 fork 耗时（可通过 `INFO stats` 的 `latest_fork_usec` 监控）
- **故障演练**：定期测试从 RDB/AOF 恢复数据的流程

---

**相关面试题** → [[../../10_DevelopLanguage/004_NOSQL/01_RedisSubject/03、Redis 持久化机制|03、Redis持久化机制]]
