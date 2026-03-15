###### 1. Redis 的过期键删除策略有哪些?
Redis 采用**惰性删除**和**定期删除**相结合的策略，两者协同工作，在 CPU 利用率和内存利用率之间取得平衡。

理论上还有一种"定时删除"（设置过期时间时创建定时器，到期立即触发删除），它对内存最友好，但会给 CPU 造成很大压力，Redis **没有采用**这种方式。

Redis 选择惰性 + 定期组合的原因是：惰性删除是 CPU 友好型（只在访问时检查，不额外占用 CPU），定期删除是内存友好型（主动清理，防止大量过期 key 长期占着内存），两者互补。

> 📖 [[../../../23_RedisKnowledge/04_过期与内存/01、过期策略与内存淘汰#1-过期键删除策略|知识库：过期键删除策略]]

###### 2. 什么是惰性删除?
惰性删除是**被动清理**策略：只有当客户端访问某个 key 时，Redis 才顺带检查它是否过期。如果过期就删掉并返回空值，如果没过期就正常返回。

**源码实现**：核心逻辑在 `db.c` 的 `expireIfNeeded` 函数里。所有读写数据库的命令（`GET`、`HGET`、`SET` 等）在执行前都会先调用这个函数检查 key 是否过期。

```c
int expireIfNeeded(redisDb *db, robj *key) {
    if (!keyIsExpired(db, key)) return 0;  // 未过期，直接返回
    
    server.stat_expiredkeys++;  // 过期 key 计数 +1
    // 根据 lazyfree-lazy-expire 配置决定同步还是异步删除
    return server.lazyfree_lazy_expire ?
        dbAsyncDelete(db, key) : dbSyncDelete(db, key);
}
```

**优点**：对 CPU 极友好，不创建额外的定时任务，删除操作只在必要时触发。

**缺点**：对内存不友好。如果一个 key 过期后永远不再被访问，它会一直占着内存，相当于内存泄漏。这就是为什么光靠惰性删除不够，还需要定期删除来兜底。

> 📖 [[../../../23_RedisKnowledge/04_过期与内存/01、过期策略与内存淘汰#2-惰性删除|知识库：惰性删除原理]]

###### 3. 什么是定期删除?
定期删除是**主动清理**策略：Redis 周期性地执行 `activeExpireCycle` 函数，随机抽查一部分 key 并删除其中已过期的。

**工作流程**（源码 `expire.c`）：
1. 每次最多遍历 16 个数据库
2. 从每个库的过期字典（`expires`）里随机抽取 20 个 key
3. 删除所有已过期的 key
4. **自适应调整**：如果抽中的过期 key 比例超过 25%，说明这个库里过期 key 很多，继续重复抽样，直到过期比例低于 25% 或达到时间上限

**执行时间控制**：为防止定期删除占用过多 CPU，有严格的时间限制：慢模式下每次不超过 25ms，快模式下更短。通过 `hz` 参数（默认 10）控制每秒执行频率。

**25% 阈值的意义**：自适应策略让定期删除在过期 key 密集时更积极，在过期 key 少时节约 CPU，非常聪明的设计。

> 📖 [[../../../23_RedisKnowledge/04_过期与内存/01、过期策略与内存淘汰#3-定期删除|知识库：定期删除原理]]

###### 4. Redis 的内存淘汰策略有哪些?
当 Redis 内存用量达到 `maxmemory` 上限时，再有写入命令就会触发内存淘汰。Redis 提供 8 种淘汰策略，可以分三个维度来记：

**范围维度**：`allkeys-` 前缀的对所有 key 操作，`volatile-` 前缀的只对设置了过期时间的 key 操作。

**算法维度**：
- **LRU**（最近最久未使用）：淘汰最长时间没被访问的 key
- **LFU**（最不经常使用）：淘汰访问频率最低的 key
- **random**：随机淘汰
- **ttl**：淘汰剩余过期时间最短的 key（只有 volatile 版本）

**特殊策略**：`noeviction` 是默认策略，内存满了就报错，不淘汰任何 key，保证数据完整性。

总结成八个：`noeviction`、`allkeys-lru`、`volatile-lru`、`allkeys-random`、`volatile-random`、`volatile-ttl`、`allkeys-lfu`、`volatile-lfu`。

**配置方法**：

```bash
# redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru

# 运行时修改
CONFIG SET maxmemory-policy allkeys-lfu
```

**选择建议**：通用缓存且无法预测访问模式就用 `allkeys-lru`；有明显热点数据就用 `allkeys-lfu`；需要保留某些永久 key 就用 `volatile-` 系列。

> 📖 [[../../../23_RedisKnowledge/04_过期与内存/01、过期策略与内存淘汰#4-8-种内存淘汰策略|知识库：8 种内存淘汰策略详解]]

###### 5. LRU 和 LFU 算法的区别是什么?
**LRU（Least Recently Used）** 关注的是**访问时间**：淘汰最久没有被访问的 key。如果一个 key 在很久以前被频繁访问，但最近一段时间没有访问，LRU 会把它淘汰掉。

**LFU（Least Frequently Used）** 关注的是**访问频率**：淘汰访问次数最少的 key。即使某个 key 最近刚被访问过，但历史上总共只访问了几次，LFU 也可能淘汰它。

**一个直观例子**：key A 在一小时前被访问了 1000 次，最近 5 分钟没访问；key B 最近 5 分钟刚被访问了 3 次。LRU 会淘汰 A（最久未访问），LFU 会淘汰 B（访问频率低）。哪个策略更合理取决于业务场景。

**Redis 的近似实现**：两种算法都用了**采样近似**而非精确实现（精确 LRU 需要维护全局链表，开销太大）。每个 `redisObject` 里有一个 24 位的 `lru` 字段，LRU 模式下记录最近访问时间戳，LFU 模式下高 16 位记录最近衰减时间、低 8 位记录访问频率计数（最大 255）。淘汰时随机采样（默认 5 个 key），从中选最优的淘汰。

**LFU 的时间衰减**：LFU 计数器会随时间自动衰减（避免历史高频 key 永远不被淘汰），衰减速率由 `lfu-decay-time` 控制，默认每分钟计数器减半。

> 📖 [[../../../23_RedisKnowledge/04_过期与内存/01、过期策略与内存淘汰#5-lru-vs-lfu|知识库：LRU vs LFU 算法对比]]

###### 6. Redis 如何设置过期时间?
Redis 提供多种命令设置 TTL，常用的有：

**`EXPIRE key seconds`**：设置 key 的过期时间，单位秒。例如 `EXPIRE user:123 3600` 表示 1 小时后过期。

**`PEXPIRE key milliseconds`**：毫秒精度的过期时间，适合需要精细控制的场景。

**`EXPIREAT key timestamp`**：设置 key 在某个具体的 UNIX 时间戳（秒）过期，适合定时任务场景。

**`PEXPIREAT key timestamp`**：毫秒级 UNIX 时间戳。

**`SETEX key seconds value`**：原子操作，同时设置值和过期时间（秒）。等价于 `SET` + `EXPIRE`，但保证原子性。

**`SET key value EX seconds NX`**：设置值的同时指定过期时间，支持 `NX`（不存在才设置）等选项，是分布式锁的标准写法。

**查看和清除过期时间**：
- `TTL key`：查看剩余秒数，返回 -1 表示没有过期时间，-2 表示 key 不存在
- `PTTL key`：毫秒精度的剩余时间
- `PERSIST key`：移除过期时间，让 key 变为永久 key

> 📖 [[../../../23_RedisKnowledge/04_过期与内存/01、过期策略与内存淘汰#6-过期时间命令|知识库：过期时间命令汇总]]

###### 7. Redis 内存满了会怎样?
当 Redis 内存达到 `maxmemory` 限制，行为由 `maxmemory-policy` 决定。

**触发时机**：内存淘汰是**被动触发**的，只有当客户端执行可能增加内存的写命令（`SET`、`LPUSH` 等）时，Redis 才会检查内存并触发淘汰流程，而不是后台主动清理。

**处理过程**：Redis 按配置的策略选择要淘汰的 key 并删除，循环执行直到释放出足够的内存来完成这次写操作。

**报错场景**：如果策略是 `noeviction`，或者在 `volatile-*` 策略下根本没有设置过期时间的 key，Redis 会直接对写操作返回错误：`OOM command not allowed when used memory > 'maxmemory'`。注意已有的读操作和删除操作不受影响。

**防患于未然**：不要等到内存真的满了才处理，应该提前设置告警（比如内存使用率超过 80% 就告警），配合合理的 TTL 和淘汰策略，避免 OOM 错误影响业务。

> 📖 [[../../../23_RedisKnowledge/04_过期与内存/01、过期策略与内存淘汰#4-内存淘汰触发流程|知识库：内存淘汰触发流程]]

###### 8. 如何查看 Redis 的内存使用情况?
**`INFO memory`** 是最全面的命令，常用字段：

- `used_memory_human`：Redis 实际分配的数据内存，如 `1.23G`
- `used_memory_rss`：操作系统层面 Redis 进程占用的内存（包含碎片）
- `mem_fragmentation_ratio`：内存碎片率 = `used_memory_rss / used_memory`，正常范围 1.0-1.5，超过 1.5 说明碎片严重
- `used_memory_peak_human`：历史内存使用峰值
- `maxmemory_human`：`maxmemory` 配置值

```bash
redis-cli info memory | grep -E 'used_memory_human|mem_fragmentation_ratio|maxmemory'
```

**`MEMORY USAGE key`**（Redis 4.0+）：查看某个具体 key 占用的内存字节数，排查大 key 很有用。

**`MEMORY DOCTOR`**（Redis 4.0+）：给出内存健康状况的诊断建议，比如碎片率过高、RSS 过大等，输出是人类可读的文字，很实用。

**`redis-cli --stat`**：实时输出内存使用等统计数据，每秒刷新一次，适合做实时监控。

**`CONFIG GET maxmemory`**：查看当前设置的内存上限。

> 📖 [[../../../23_RedisKnowledge/04_过期与内存/01、过期策略与内存淘汰#7-内存监控命令|知识库：内存监控命令]]
