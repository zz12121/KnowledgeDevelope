###### 1. Redis 事务的原理是什么?
Redis 事务通过**命令队列 + 原子执行**实现，整个过程分三个阶段：**事务开始、命令入队、事务执行**。

**MULTI 命令**：标记事务开始，将客户端状态设置为 `CLIENT_MULTI`。如果已经在事务状态中再次执行 MULTI，会报错 "MULTI calls can not be nested"。

**命令入队**：处于 `CLIENT_MULTI` 状态的客户端，发送的非 MULTI/EXEC/DISCARD/WATCH 命令不会立即执行，而是被 `queueMultiCommand()` 封装成 `multiCmd` 结构体，放入 `client->mstate.commands` 这个 FIFO 队列里。

**EXEC 命令**：触发执行，服务器遍历事务队列，依次执行所有命令，把所有结果打包成一个响应返回给客户端。执行过程中不会被其他客户端命令插入，这保证了事务的原子性。

**关键数据结构**（`redis.h`）：

```c
typedef struct multiState {
    multiCmd *commands;  // 事务命令队列（FIFO）
    int count;           // 队列中的命令数
} multiState;
```

> 📖 [[../../../23_RedisKnowledge/07_事务与锁/01、事务与分布式锁#1-redis-事务原理|知识库：Redis 事务原理]]

###### 2. Redis 事务支持回滚吗?
**Redis 事务不支持回滚**，这是 Redis 的刻意设计选择，不是局限。

**错误处理分两种情况**：
- **EXEC 前的错误**（入队时命令语法就错了，比如 `MULTI` → `WRONG_CMD` → `EXEC`）：事务会被拒绝执行，所有命令都不执行。
- **EXEC 后的错误**（命令语法对但操作类型错误，比如对 String 执行 LPUSH）：只有错误的那条命令报错，**其他命令正常执行且不会回滚**。

**为什么不支持回滚**：Redis 认为事务执行错误基本都是编程 bug（用错了命令类型），这种问题应该在开发测试阶段发现，而不是靠运行时回滚来兜底。不支持回滚让 Redis 内核更简单，性能更高。这和关系型数据库的事务设计哲学完全不同。

> 📖 [[../../../23_RedisKnowledge/07_事务与锁/01、事务与分布式锁#2-不支持回滚的设计哲学|知识库：Redis 事务不支持回滚]]

###### 3. WATCH 命令的作用是什么?
`WATCH` 实现了一种**乐观锁**机制，用于在事务执行前检测被监控的 key 是否被其他客户端修改，以保证 CAS（Compare-And-Swap）语义。

**源码实现原理**：
1. 执行 `WATCH key` 时，把当前客户端注册到 `db->watched_keys[key]` 的客户端列表里
2. 任何修改数据库 key 的命令执行后，都会调用 `signalModifiedKey()` → `touchWatchedKey()`，将监控该 key 的所有客户端标记为 `CLIENT_DIRTY_CAS`
3. 客户端执行 `EXEC` 时，检查 `CLIENT_DIRTY_CAS` 标志，如果被置位，说明监控的 key 已被修改，**拒绝执行事务，返回 nil**

**应用示例（Java Jedis）**：

```java
jedis.watch("account:balance");
String balance = jedis.get("account:balance");

Transaction tx = jedis.multi();
tx.set("account:balance", String.valueOf(Integer.parseInt(balance) - 100));
List<Object> result = tx.exec();

if (result == null) {
    // 事务失败，说明 balance 在 WATCH 后被其他客户端修改了
    // 可以重试整个操作
    retry();
}
```

`WATCH` 配合 `MULTI/EXEC` 实现了"如果期间没人改这个 key，我的操作才执行"的语义，是 Redis 实现乐观并发控制的标准方式。

> 📖 [[../../../23_RedisKnowledge/07_事务与锁/01、事务与分布式锁#3-watch-乐观锁|知识库：WATCH 乐观锁机制]]

###### 4. Redis 如何实现分布式锁?
分布式锁的核心需求是**互斥**：同一时刻只有一个客户端能持有锁。Redis 分布式锁的实现有一个清晰的演进过程：

**版本一：SETNX + EXPIRE（有问题）**

```java
Boolean locked = jedis.setnx("lock", "1");  // 设置锁
if (locked) {
    jedis.expire("lock", 30);  // 设置过期时间（非原子！）
}
```

问题：`SETNX` 和 `EXPIRE` 是两步操作，如果 `SETNX` 成功后应用崩溃，锁永远不会过期，造成死锁。

**版本二：SET NX PX（原子设置）**

```java
String result = jedis.set("lock", "1", "NX", "PX", 30000);
if ("OK".equals(result)) {
    // 获锁成功，超时时间 30 秒
}
```

原子操作，解决了死锁问题。但解锁时直接 `DEL` 有问题：如果 A 的锁超时了，B 获得了锁，A 完成操作后会把 B 的锁删掉。

**版本三：UUID + Lua 脚本（生产可用）**

```java
// 加锁：value 存唯一标识
String uniqueId = UUID.randomUUID().toString();
jedis.set("lock", uniqueId, "NX", "PX", 30000);

// 解锁：先验证是不是自己的锁，再删，用 Lua 保证原子性
String luaScript =
    "if redis.call('get', KEYS[1]) == ARGV[1] then " +
    "    return redis.call('del', KEYS[1]) " +
    "else " +
    "    return 0 " +
    "end";
jedis.eval(luaScript, Collections.singletonList("lock"), Collections.singletonList(uniqueId));
```

这版本解决了误删别人锁的问题，是目前推荐的基础实现。

> 📖 [[../../../23_RedisKnowledge/07_事务与锁/01、事务与分布式锁#4-分布式锁演进|知识库：分布式锁演进]]

###### 5. 分布式锁需要注意哪些问题?
分布式锁有五个核心问题需要关注：

**死锁**：客户端获取锁后崩溃，锁无法释放。解决：用 `PX` 选项设置合理的超时时间，即使客户端宕机，锁也会自动过期。

**误删锁**：A 的锁超时了，B 获得锁，A 完成操作后删了 B 的锁。解决：锁的 value 设为客户端唯一标识（UUID），解锁时先验证 value 是不是自己的，再用 Lua 脚本原子删除。

**原子性**：`SETNX` + `EXPIRE` 不是原子的可能死锁，`GET` + `DEL` 不是原子的可能误删。解决：加锁用 `SET NX PX` 原子命令，解锁用 Lua 脚本。

**锁续期**：业务操作时间可能超过锁的过期时间，锁自动过期后其他客户端可能抢到锁，导致并发问题。解决：Redisson 的看门狗机制，获锁后启动后台线程定期续期，直到业务完成主动释放。

**集群故障**：主节点加锁成功后宕机，从节点还没同步到这个锁，故障转移后新主节点上没有这把锁，其他客户端可以再加锁。解决：RedLock 算法（有争议）；或者业务层做幂等性保证，接受极低概率的重复执行。

> 📖 [[../../../23_RedisKnowledge/07_事务与锁/01、事务与分布式锁#5-分布式锁五大核心问题|知识库：分布式锁五大问题]]

###### 6. Redlock 算法是什么?
Redlock 是 Redis 作者提出的**在多个独立 Redis 节点上实现分布式锁的算法**，解决单节点/主从架构下主节点宕机导致锁失效的问题。

**算法步骤**（以 5 个节点为例）：
1. 获取当前时间戳 T1（毫秒）
2. 依次向 5 个独立 Redis 节点（不是主从，是物理独立的）请求加锁，每个请求设置一个短超时（比如 5ms）
3. 计算获锁总耗时 T2 - T1。只有**超过 3 个（N/2+1）节点成功加锁**，且总耗时 < 锁有效期，才算获锁成功
4. 获锁成功后，实际有效期 = 初始有效期 - 耗时
5. 若获锁失败，向所有节点发送解锁请求

**争议**：Martin Kleppmann（分布式系统专家）认为 Redlock 在某些极端场景下（进程 GC pause、时钟跳跃）仍然不安全，并不是真正意义上的强一致性分布式锁。Redis 作者 Antirez 对此有反驳。这场争论在社区广为人知，实践中大多数人选择接受 Redlock 的局限性，配合业务幂等性一起使用。

> 📖 [[../../../23_RedisKnowledge/07_事务与锁/01、事务与分布式锁#7-redlock-算法|知识库：RedLock 算法与争议]]

###### 7. 如何避免分布式锁的死锁?
死锁的核心原因是锁的持有者无法释放锁（宕机、网络问题等），防死锁也围绕这一点展开。

**最基本：设置超时时间**。锁必须有过期时间，即使持有者挂了，锁也会自动释放：

```java
jedis.set("lock", uniqueId, "NX", "PX", 30000);  // 30 秒超时
```

**进阶：看门狗机制（Redisson）**。如果业务执行时间不确定，固定超时容易提前过期；太长又影响可用性。Redisson 的解决方案是：获锁成功后启动一个后台定时线程，每隔 10 秒检查客户端是否还持有锁，如果是就把过期时间续期到 30 秒。只要客户端正常运行，锁就不会过期；客户端宕机后，看门狗停止，锁在 30 秒内自然过期。

```java
RLock lock = redisson.getLock("myLock");
lock.lock();  // 默认 30 秒看门狗超时，自动续期
try {
    // 业务逻辑
} finally {
    lock.unlock();  // 主动释放
}
```

**另外**：确保加锁是原子操作（`SET NX PX`），避免"加锁成功但设置超时失败"的问题。

> 📖 [[../../../23_RedisKnowledge/07_事务与锁/01、事务与分布式锁#6-死锁防范与看门狗|知识库：分布式锁死锁防范]]

###### 8. Redis 分布式锁和 Zookeeper 分布式锁的区别?
两者都能实现分布式锁，本质区别是一致性模型不同：Redis 是 **AP** 系统，ZooKeeper 是 **CP** 系统。

**实现原理不同**：Redis 锁基于 `SET NX PX` 命令（内存操作），ZooKeeper 锁基于临时顺序节点（持久化到磁盘，所有节点强一致）。

**锁的释放机制不同**：Redis 依赖过期时间，客户端挂了要等 TTL 到期才释放，有丢锁风险（主从同步延迟时主节点宕机）；ZooKeeper 使用会话机制，客户端连接断开，临时节点自动删除，锁立即释放，更可靠。

**等待锁的方式不同**：Redis 客户端需要自旋轮询（busy-wait）；ZooKeeper 通过 Watch 机制监听前序节点，排队等待，不浪费 CPU，还天然实现了公平锁。

**性能不同**：Redis 内存操作，吞吐量极高；ZooKeeper 要保证多节点一致性，吞吐量相对较低。

**选择建议**：追求高性能高并发、能接受极端情况下极低概率锁失效，选 Redis；要求高可靠性强一致性、对锁安全要求苛刻，选 ZooKeeper。

> 📖 [[../../../23_RedisKnowledge/07_事务与锁/01、事务与分布式锁#8-redis-锁-vs-zookeeper-锁|知识库：Redis 锁 vs ZooKeeper 锁]]

###### 9. 如何实现可重入的分布式锁?
可重入锁要求**同一个线程可以多次获取同一把锁而不死锁**，解锁时需要和加锁次数对应才能真正释放。

**实现方案：Hash 结构 + Lua 脚本**

用 Redis Hash 存储锁信息：field 是"客户端 ID + 线程 ID"，value 是重入次数。

```lua
-- 加锁脚本
if (redis.call('exists', KEYS[1]) == 0) then
    -- 锁不存在，首次加锁
    redis.call('hset', KEYS[1], ARGV[1], 1)
    redis.call('pexpire', KEYS[1], ARGV[2])
    return 1
end
if (redis.call('hexists', KEYS[1], ARGV[1]) == 1) then
    -- 是自己的锁，重入次数 +1
    redis.call('hincrby', KEYS[1], ARGV[1], 1)
    redis.call('pexpire', KEYS[1], ARGV[2])
    return 1
end
return 0  -- 锁被别人持有，加锁失败

-- 解锁脚本
if (redis.call('hexists', KEYS[1], ARGV[1]) == 0) then
    return 0  -- 不是自己的锁
end
local counter = redis.call('hincrby', KEYS[1], ARGV[1], -1)
if (counter > 0) then
    redis.call('pexpire', KEYS[1], ARGV[2])  -- 还有重入，刷新过期时间
    return 1
else
    redis.call('del', KEYS[1])  -- 最后一次解锁，删掉
    return 1
end
```

这正是 Redisson 的 `RLock` 底层实现原理，生产环境直接用 Redisson 即可，不需要自己写。

> 📖 [[../../../23_RedisKnowledge/07_事务与锁/01、事务与分布式锁#6-可重入锁实现|知识库：可重入分布式锁实现]]
