###### 1. Redis 如何设置密码?
Redis 提供两种密码设置方式，分别适用于不同场景：
**配置文件设置（持久化生效）**
修改 `redis.conf`文件中的 `requirepass`参数：
```bash
# 编辑配置文件
vim /etc/redis/redis.conf

# 找到并修改以下行
requirepass YourStrongPassword123!

# 重启Redis使配置生效
systemctl restart redis-server
```
此方式需重启服务，密码会持久化保存。
**命令行临时设置（立即生效）**
通过 Redis 命令行临时设置密码：
```bash
redis-cli
127.0.0.1:6379> CONFIG SET requirepass "YourStrongPassword123!"
OK
```
这种方式无需重启，但重启后失效，适合临时调试。
**密码验证方式**
- 连接时验证：`redis-cli -a YourPassword`
- 连接后验证：连接后执行 `AUTH YourPassword`
**安全建议**
- 密码应包含大小写字母、数字和特殊字符，长度至少16位
- 定期更换密码，避免使用默认密码
- 主从架构中需在从库配置 `masterauth`参数同步主库密码
###### 2. Redis 如何禁用危险命令?
通过重命名或禁用高风险命令提升安全性，修改 `redis.conf`：
```bash
# 禁用FLUSHALL/FLUSHDB（清空数据库）
rename-command FLUSHALL ""
rename-command FLUSHDB ""

# 重命名KEYS命令（避免阻塞）
rename-command KEYS "REDIS_KEYS"

# 限制CONFIG命令（防止配置篡改）
rename-command CONFIG "REDIS_CONFIG"
```
**注意事项**：
- 修改后需重启Redis生效
- 禁用命令可能影响依赖这些命令的脚本或应用
- 可配置为复杂名称平衡安全性与便利性
###### 3. Redis 的备份和恢复策略?
**RDB快照备份**
- **原理**：定时生成内存数据的时间点快照
- **配置**（redis.conf）：
```bash
    save 900 1       # 900秒内至少1个键变更
    save 300 10      # 300秒内至少10个键变更
    save 60 10000    # 60秒内至少10000个键变更
    dbfilename dump.rdb
    dir /var/lib/redis
```
- **手动触发**：`SAVE`（阻塞）或 `BGSAVE`（后台异步）
**AOF日志备份**
- **原理**：记录每个写操作命令的日志
- **配置**：
```bash
    appendonly yes
    appendfilename "appendonly.aof"
    appendfsync everysec  # 同步策略（always/everysec/no）
```
**混合持久化**（Redis 4.0+）
结合RDB快照和增量AOF日志，兼顾恢复速度与数据安全。
**企业级备份方案**：
```bash
# 每小时备份脚本示例
#!/bin/sh
cur_date=`date +%Y%m%d%k`
mkdir -p /backup/redis/$cur_date
cp /var/lib/redis/dump.rdb /backup/redis/$cur_date/

# 清理48小时前备份
find /backup/redis -type d -mtime +2 -exec rm -rf {} \;
```
**恢复流程**
1. 停止Redis服务
2. 替换备份文件（RDB或AOF）
3. 重启服务并验证数据
###### 4. 如何监控 Redis 的运行状态?
**内置监控命令**
- **INFO**：查看全局状态
```bash
    redis-cli info memory      # 内存详情
    redis-cli info stats       # 命令统计
    redis-cli info replication # 主从状态
```
- **MONITOR**：实时命令监控（谨慎使用，影响性能）
- **SLOWLOG**：慢查询分析
**关键监控指标**
- **内存使用**：`used_memory`、`mem_fragmentation_ratio`（碎片率）
- **连接数**：`connected_clients`（当前连接数）
- **命中率**：`keyspace_hits/(keyspace_hits+keyspace_misses)`
**外部监控工具**
- **Prometheus + Redis-exporter**：提供可视化监控和报警
- **Grafana仪表盘**：展示历史趋势和关键指标
- **自定义脚本**（Python示例）：
```python
    import redis
    r = redis.Redis(host='localhost', port=6379)
    info = r.info()
    print(f"内存使用: {info['used_memory_human']}")
    print(f"连接数: {info['connected_clients']}")
```
###### 5. Redis 的内存碎片问题如何处理?
**内存碎片成因**
- 频繁修改键值对导致内存分配不一致
- 键过期删除后空间无法立即重用
**检测方法**
通过 `INFO memory`查看碎片率：
```bash
redis-cli info memory | grep mem_fragmentation_ratio
# 合理范围：1-1.5；>1.5需处理
```
**解决方案**
1. **自动碎片整理**（Redis 4.0+）：
```bash
    # 启用自动整理
    config set activedefrag yes
    # 设置触发阈值
    config set active-defrag-ignore-bytes 100mb
    config set active-defrag-threshold-lower 10
```
2. **手动清理**：
```bash
    redis-cli memory purge  # 需jemalloc支持
```
3. **重启服务**：强制内存重新分配（生产环境慎用）
**优化建议**
- 使用适当的数据结构减少频繁修改
- 避免大量键同时过期
- 监控碎片率并设置告警阈值（如>1.5）
###### 6. Redis 如何进行版本升级?
**升级前准备**
1. **兼容性检查**：
    - 确认客户端协议兼容性（如RESP2/RESP3）
    - 检查第三方模块（RedisJSON等）版本支持
2. **数据备份**：执行 `BGSAVE`确保数据安全
**升级流程**
3. **单机升级步骤**：
```bash
    # 下载新版本并编译
    wget http://download.redis.io/releases/redis-7.2.tar.gz
    tar xzf redis-7.2.tar.gz && cd redis-7.2
    make && make install
    
    # 平滑重启
    redis-cli shutdown
    redis-server /path/to/redis.conf
```
4. **集群升级方案**：
    - 滚动升级：逐个节点升级，避免服务中断
    - 主从切换：先升级从节点，然后执行故障转移
**验证与回滚**
- **功能验证**：检查命令兼容性和性能表现
- **回滚方案**：备份旧版本二进制文件，必要时快速还原
###### 7. Redis 的配置文件有哪些重要参数?
**安全相关参数**
```bash
requirepass "密码"          # 访问密码[1](@ref)
rename-command FLUSHALL ""  # 禁用危险命令[6](@ref)
bind 127.0.0.1              # 限制访问IP[1](@ref)
```
**性能优化参数**
```bash
maxmemory 4gb                    # 最大内存限制
maxmemory-policy allkeys-lru     # 内存淘汰策略
save 900 1                       # RDB触发条件[9](@ref)
appendonly yes                   # 开启AOF持久化[9](@ref)
```
**内存管理参数**
```bash
activedefrag yes                  # 启用碎片整理[15,17](@ref)
active-defrag-threshold-lower 10  # 碎片整理触发阈值[17](@ref)
hash-max-ziplist-entries 512      # 小数据结构优化[17](@ref)
```
###### 8. 如何排查 Redis 的故障?
**常见故障分类**
1. **内存不足**：监控 `used_memory`，优化淘汰策略
2. **连接数超标**：调整 `maxclients`，检查客户端行为
3. **持久化失败**：检查磁盘空间和权限
4. **主从同步异常**：验证网络和密码配置
**排查工具与命令**
```bash
# 检查服务状态
systemctl status redis-server

# 分析日志
tail -f /var/log/redis/redis-server.log

# 性能诊断
redis-cli --latency              # 网络延迟
redis-cli --stat                 # 实时统计
redis-cli slowlog get            # 慢查询分析
```
**系统性排查流程**
1. **资源检查**：CPU、内存、磁盘、网络
2. **配置验证**：参数合理性及冲突检测
3. **日志分析**：错误信息和警告事件
4. **性能 profiling**：识别瓶颈点