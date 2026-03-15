###### 1. Redis 如何设置密码?
Redis 提供两种密码设置方式，分别适用于不同场景：

**配置文件设置（持久化生效）**

修改 `redis.conf` 文件中的 `requirepass` 参数：

```bash
# 找到并修改以下行
requirepass YourStrongPassword123!

# 重启 Redis 使配置生效
systemctl restart redis-server
```

此方式需要重启服务，密码会持久化保存，是生产环境的标准做法。

**命令行临时设置（立即生效）**

```bash
redis-cli
127.0.0.1:6379> CONFIG SET requirepass "YourStrongPassword123!"
OK
```

这种方式无需重启，但服务重启后失效，适合临时调试场景。

**验证方式**

有两种方式进行认证：连接时直接带密码 `redis-cli -a YourPassword`；或者连接后执行 `AUTH YourPassword` 命令。

**安全建议**：密码应包含大小写字母、数字和特殊字符，长度至少 16 位。主从架构中，还需要在从库配置 `masterauth` 参数，与主库密码保持一致，否则主从复制会因认证失败而中断。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/02、安全与运维#1-密码认证-requirepass|知识库：Redis 密码认证与安全配置]]

---

###### 2. Redis 如何禁用危险命令?
通过 `rename-command` 重命名或禁用高风险命令是提升 Redis 安全性的重要手段。修改 `redis.conf`：

```bash
# 禁用 FLUSHALL/FLUSHDB（清空数据库）
rename-command FLUSHALL ""
rename-command FLUSHDB ""

# 重命名 KEYS 命令（避免全量扫描阻塞）
rename-command KEYS "REDIS_KEYS_INTERNAL"

# 限制 CONFIG 命令（防止运行时配置篡改）
rename-command CONFIG "REDIS_CONFIG_INTERNAL"
```

将命令重命名为空字符串 `""` 就是彻底禁用，重命名为复杂字符串则是"半禁用"——既保留了应急使用的能力，又防止了误操作或被攻击。

**注意**：修改后需重启 Redis 生效，并且要同步更新所有依赖这些命令的脚本和应用，避免因命令名变更导致功能异常。生产环境强烈建议同时配合 `bind` 参数限制访问 IP，双重保护。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/02、安全与运维#2-禁用危险命令-rename-command|知识库：Redis 危险命令禁用]]

---

###### 3. Redis 的备份和恢复策略?
Redis 的备份核心是 RDB 快照和 AOF 日志，两者可以配合使用。

**RDB 快照备份**

RDB 是定时生成内存数据的时间点快照，配置示例：

```bash
save 900 1       # 900 秒内至少 1 个键变更
save 300 10      # 300 秒内至少 10 个键变更
save 60 10000    # 60 秒内至少 10000 个键变更
dbfilename dump.rdb
dir /var/lib/redis
```

手动触发有两个命令：`SAVE` 会阻塞主线程，生产环境禁用；`BGSAVE` 会 fork 子进程后台异步执行，是正确姿势。

**AOF 日志备份**

AOF 记录每个写操作命令，配置如下：

```bash
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec  # 每秒同步，平衡性能与安全
```

**混合持久化**（Redis 4.0+）

混合持久化结合了 RDB 的快速恢复和 AOF 的高数据完整性——AOF 文件头部是 RDB 格式的快照，尾部是增量 AOF 日志。重启时先加载 RDB 快照，再重放少量 AOF 增量，重启速度远快于纯 AOF。

**企业级备份方案**

```bash
#!/bin/sh
cur_date=`date +%Y%m%d%k`
mkdir -p /backup/redis/$cur_date
cp /var/lib/redis/dump.rdb /backup/redis/$cur_date/

# 清理 48 小时前的备份
find /backup/redis -type d -mtime +2 -exec rm -rf {} \;
```

**恢复流程**：先停止 Redis 服务，替换备份文件（RDB 或 AOF），重启服务后验证数据完整性。恢复时如果同时存在 RDB 和 AOF，Redis 会优先使用 AOF（因为 AOF 数据更新）。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/02、安全与运维#3-备份与恢复策略|知识库：Redis 备份恢复策略]]

---

###### 4. 如何监控 Redis 的运行状态?
监控 Redis 分两层：内置命令和外部工具体系。

**内置监控命令**

`INFO` 是最常用的，可按分组查看详情：

```bash
redis-cli info memory      # 内存详情（used_memory、碎片率等）
redis-cli info stats       # 命令统计（ops/s、命中率等）
redis-cli info replication # 主从状态（角色、偏移量等）
```

`MONITOR` 可以实时打印所有命令，适合开发调试，但**生产环境慎用**，因为它会显著降低 Redis 性能。

`SLOWLOG` 用于分析慢查询，`redis-cli slowlog get` 可查看超过阈值的慢命令。

**关键监控指标**

- **内存使用**：`used_memory` 是实际使用内存，`mem_fragmentation_ratio`（碎片率）正常范围在 1.0~1.5，超过 1.5 说明碎片严重
- **连接数**：`connected_clients` 超过 `maxclients` 阈值时会拒绝新连接
- **命中率**：`keyspace_hits / (keyspace_hits + keyspace_misses)`，低于 90% 需排查缓存策略

**外部监控工具**

生产标配是 **Prometheus + Redis Exporter + Grafana** 三件套：Redis Exporter 抓取 `INFO` 数据暴露给 Prometheus，Grafana 展示历史趋势和告警。也可以用 Python 脚本做轻量监控：

```python
import redis
r = redis.Redis(host='localhost', port=6379)
info = r.info()
print(f"内存使用: {info['used_memory_human']}")
print(f"连接数: {info['connected_clients']}")
print(f"命中率: {info['keyspace_hits'] / (info['keyspace_hits'] + info['keyspace_misses'] + 1):.2%}")
```

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/02、安全与运维#4-监控与可观测性|知识库：Redis 监控体系]]

---

###### 5. Redis 的内存碎片问题如何处理?
内存碎片是个常见但容易被忽视的问题。

**成因**：频繁修改键值对时，内存分配器（jemalloc）可能分配比实际需要更大的内存块；键过期删除后留下的空隙无法被后续分配立即填充，久而久之碎片率就上来了。

**检测方式**

```bash
redis-cli info memory | grep mem_fragmentation_ratio
# 输出示例：mem_fragmentation_ratio:1.89
# 合理范围：1.0~1.5；>1.5 需处理；<1.0 说明内存不足，Redis 在使用 swap
```

**解决方案**

Redis 4.0+ 提供了**自动碎片整理**（`activedefrag`），这是生产环境首选：

```bash
config set activedefrag yes
config set active-defrag-ignore-bytes 100mb   # 碎片超过 100MB 才触发
config set active-defrag-threshold-lower 10   # 碎片率超过 10% 才触发
```

`activedefrag` 在后台以渐进式方式整理内存，不会明显影响服务性能。

也可以执行 `redis-cli memory purge` 主动释放（需要 jemalloc 支持），或者直接重启服务强制内存重新分配——但重启在生产环境是最后手段。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/02、安全与运维#5-内存碎片整理|知识库：Redis 内存碎片整理]]

---

###### 6. Redis 如何进行版本升级?
版本升级的核心原则是：备份先行、兼容性确认、滚动升级。

**升级前准备**

首先确认客户端协议兼容性（如是否需要 RESP3），以及第三方模块（RedisJSON、RediSearch 等）的版本支持情况。然后执行 `BGSAVE` 确保有最新的 RDB 备份。

**单机升级步骤**

```bash
# 1. 下载并编译新版本
wget http://download.redis.io/releases/redis-7.2.tar.gz
tar xzf redis-7.2.tar.gz && cd redis-7.2
make && make install

# 2. 平滑重启（会加载现有 RDB/AOF）
redis-cli shutdown
redis-server /path/to/redis.conf
```

**集群滚动升级方案**

集群环境推荐**滚动升级**：先逐个升级从节点，再对每个主节点执行 `CLUSTER FAILOVER` 将其切换为从节点后升级，最后再 failover 回主节点。整个过程服务不中断，风险可控。

**验证与回滚**：升级完成后检查命令兼容性和关键性能指标。升级前保留旧版本二进制文件，一旦出现异常可以快速回滚——只需用旧二进制重启即可，数据文件格式向后兼容。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/02、安全与运维#6-版本升级方案|知识库：Redis 版本升级]]

---

###### 7. Redis 的配置文件有哪些重要参数?
`redis.conf` 中几类关键参数值得重点关注：

**安全相关**

```bash
requirepass "密码"           # 访问密码，生产必须设置
rename-command FLUSHALL ""   # 禁用危险命令
bind 127.0.0.1               # 限制监听 IP，避免公网暴露
```

**性能与内存**

```bash
maxmemory 4gb                    # 最大内存限制
maxmemory-policy allkeys-lru     # 内存淘汰策略（超出 maxmemory 后的行为）
hz 10                            # 事件循环频率，影响过期键扫描精度
```

**持久化**

```bash
save 900 1                       # RDB 触发条件
appendonly yes                   # 开启 AOF
appendfsync everysec             # AOF 同步策略
aof-use-rdb-preamble yes         # 开启混合持久化（4.0+）
```

**内存碎片整理**

```bash
activedefrag yes                   # 启用自动碎片整理（4.0+）
active-defrag-threshold-lower 10   # 碎片整理触发阈值（%）
hash-max-ziplist-entries 128       # Hash 小数据结构编码阈值
zset-max-ziplist-entries 128       # ZSet 小数据结构编码阈值
```

这些参数要结合业务场景来调整，没有通用最优值。比如 `maxmemory-policy` 的选择取决于业务能否容忍缓存被淘汰；`appendfsync` 的策略取决于对数据安全 vs 写性能的取舍。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/02、安全与运维#7-重要配置参数|知识库：Redis 重要配置参数]]

---

###### 8. 如何排查 Redis 的故障?
排查 Redis 故障要按"资源 → 配置 → 日志 → 性能"的顺序系统来。

**常见故障类型**

- **内存不足**：`used_memory` 接近 `maxmemory` 时会触发淘汰，检查淘汰策略和 key 的生命周期
- **连接数超标**：`connected_clients` 超限后会拒绝新连接，排查是否有连接泄漏（长连接未释放）
- **持久化失败**：通常是磁盘空间不足或权限问题，`BGSAVE` 返回 Background saving failed 时立即告警
- **主从同步异常**：`INFO replication` 查看 `master_link_status` 是否为 `up`，排查网络和密码配置

**排查工具**

```bash
# 查看实时统计（命中率、ops/s 等）
redis-cli --stat

# 检查网络延迟
redis-cli --latency

# 查看慢查询（执行时间超过 slowlog-log-slower-than 阈值的命令）
redis-cli slowlog get 10

# 内存诊断报告
redis-cli memory doctor

# 查看 latency 历史
redis-cli latency history event
```

**系统性排查流程**：先看资源水位（CPU、内存、磁盘、网络带宽）；再验证配置参数是否合理（`CONFIG GET *` 对比预期值）；然后分析日志中的 ERROR 和 WARNING；最后做性能 profiling 定位瓶颈点——比如通过 `slowlog` 找出耗时命令，通过 `--latency` 排查网络抖动。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/02、安全与运维#8-故障排查流程|知识库：Redis 故障排查]]
