###### 1. Redis 事务的原理是什么?
Redis事务的核心原理是通过**命令队列+原子执行**的机制实现的，包含三个阶段：**事务开始、命令入队、事务执行**。
**事务流程与核心源码机制：**
- **MULTI命令**：标记事务开始，将客户端状态设置为`CLIENT_MULTI`。在`multiCommand(client *c)`函数中，检查若已处于事务状态则报错"MULTI calls can not be nested"。
- **命令入队**：非MULTI/EXEC/DISCARD/WATCH的命令会被暂存到事务队列。在`processCommand()`函数中，若客户端标志包含`CLIENT_MULTI`，则调用`queueMultiCommand()`将命令封装为`multiCmd`结构体存入`client->mstate.commands`数组（一个FIFO队列）。
- **EXEC命令**：触发事务执行。在`execCommand()`函数中，服务器遍历事务队列，依次执行所有命令，并将结果一次性返回给客户端。执行过程中不会被其他客户端命令打断。
**关键数据结构**（定义在`redis.h`）：
```c
typedef struct multiState {
    multiCmd *commands;  // 事务命令队列
    int count;           // 命令计数
} multiState;

typedef struct multiCmd {
    robj **argv;        // 命令参数
    int argc;           // 参数个数
    struct redisCommand *cmd;  // 命令结构
} multiCmd;
```
###### 2. Redis 事务支持回滚吗?
**Redis事务不支持回滚机制**。
**错误处理策略分为两种情况：**
- **EXEC执行前错误**（如命令语法错误）：事务会被拒绝执行，所有命令都不会执行。
- **EXEC执行后错误**（如命令执行错误，例如对字符串值调用哈希表操作命令）：错误命令会报错，但**事务会继续执行后续命令，且已执行成功的命令不会被撤销**。
**设计哲学**：Redis认为事务错误通常是编程错误，应在开发环境发现并修复，而非依赖运行时的回滚。这种设计简化了Redis内核，提升了性能。
###### 3. WATCH 命令的作用是什么?
`WATCH`命令实现了一种**乐观锁**机制，用于在事务执行前检查监控的键是否被修改，以保证数据一致性。
**源码实现原理：**
1. **注册监控**：当客户端执行`WATCH key`时，`watchCommand()`会调用`watchForKey()`，将客户端添加到Redis数据库的`db->watched_keys`字典中该key对应的监控客户端列表里。
2. **标记脏键**：任何修改数据库键的命令（如SET、LPUSH）成功后，都会调用`signalModifiedKey()`函数，进而触发`touchWatchedKey()`。该函数会遍历`db->watched_keys`中找到的所有监控此键的客户端，并将它们的`flags`标记为`CLIENT_DIRTY_CAS`。
3. **执行检查**：当客户端执行`EXEC`时，服务器会检查客户端的`CLIENT_DIRTY_CAS`标志是否被置位。如果已置位，说明有被`WATCH`的键被修改过，则拒绝执行事务，返回`nil`。
**应用示例**（Java Jedis）：
```java
jedis.watch("mykey"); // 开始监控key
Transaction tx = jedis.multi();
tx.set("mykey", "new value");
if (tx.exec() == null) {
    // 如果exec返回null，表示事务执行失败（键被其他客户端修改）
    // 应用层可决定重试或放弃
}
```
###### 4. Redis 如何实现分布式锁?
基于Redis实现分布式锁的核心是**确保在分布式环境下，对一个共享资源的操作是互斥的**。其演进过程如下：
1. **基础版：SETNX + EXPIRE**
```java
    // 问题：SETNX和EXPIRE非原子操作，若设置锁后程序崩溃，会导致死锁
    Boolean lock = jedis.setnx("lock_key", "random_value");
    if (lock) {
        jedis.expire("lock_key", 30); // 非原子操作
    }
```
2. **优化版：原子SET命令**
 ```java
    // 使用SET命令的NX和PX选项，保证原子性
    String result = jedis.set("lock_key", "random_value", "NX", "PX", 30000);
    if ("OK".equals(result)) {
        // 获取锁成功
    }
```
3. **安全版：验证锁值 + Lua脚本原子解锁**
 ```java
    // 加锁
    String uniqueId = UUID.randomUUID().toString();
    String result = jedis.set("lock_key", uniqueId, "NX", "PX", 30000);
    
    // 解锁（Lua脚本保证原子性）
    String luaScript = 
        "if redis.call('get', KEYS[1]) == ARGV[1] then " +
        "   return redis.call('del', KEYS[1]) " +
        "else " +
        "   return 0 " +
        "end";
    Object unlockResult = jedis.eval(luaScript, 
        Collections.singletonList("lock_key"), 
        Collections.singletonList(uniqueId));
```
此方案通过唯一值（如UUID）避免误删其他客户端的锁，并使用Lua脚本保证判断和删除的原子性。
###### 5. 分布式锁需要注意哪些问题?
实现分布式锁时需特别注意以下问题及其解决方案：

|问题|描述|解决方案|
|---|---|---|
|**死锁**​|客户端获取锁后崩溃，锁无法释放|设置合理的过期时间（PX选项）|
|**误删锁**​|客户端A操作超时，锁过期释放后，客户端B获锁，A完成操作后误删B的锁|锁值加入唯一标识（如UUID），删除前进行验证|
|**原子性**​|非原子操作（如SETNX+EXPIRE）可能导致死锁|使用原子命令（如SET NX PX）或Lua脚本|
|**锁续期**​|业务操作时间可能超过锁过期时间|使用后台线程（看门狗）定期续期（如Redisson）|
|**集群故障**​|主从切换可能导致锁丢失|使用RedLock算法（有争议）|
###### 6. Redlock 算法是什么?
Redlock算法是Redis作者Antirez提出的**用于在Redis集群环境下实现分布式锁的算法**，旨在解决主从架构中主节点宕机可能导致锁失效的问题。
**算法步骤**：
1. 获取当前精确时间（毫秒）。
2. 客户端依次向**N个独立的Redis节点**（通常为5个）请求获取锁，使用相同的key和随机值。每个请求设置一个远小于锁总有效时间的超时时间（例如锁有效期为10秒，则超时时间设为5-50毫秒）。
3. 客户端计算获取锁总共花费的时间（当前时间减去步骤1的时间）。只有当客户端从**超过半数（N/2 + 1）的节点上成功获取锁**，且总耗时小于锁的有效期，才认为获取锁成功。
4. 如果获取锁成功，锁的有效期等于初始有效期减去获取锁总耗时。
5. 如果获取锁失败（未达到半数或超时），客户端会向所有Redis节点发起释放锁的请求（即使该节点并未成功设置锁）。
**争议点**：该算法依赖多个节点的系统时钟大致同步，且对网络延迟敏感，在某些极端场景下（如进程Pause）仍可能存在问题，社区对此有讨论。
###### 7. 如何避免分布式锁的死锁?
避免死锁的核心策略是**设置锁的过期时间**，并确保在极端情况下锁最终能被释放。
1. **设置过期时间**：这是最基本的要求。
```java
    jedis.set("lock_key", "value", "NX", "PX", 30000); // 30秒后自动过期
```
2. **锁续期（Watch Dog）**：对于可能长时间持有锁的业务，需实现锁续期机制。例如Redisson框架的看门狗机制：
    - 获取锁成功后，会启动一个后台定时任务，定期（默认每10秒）检查客户端是否还持有锁，如果是则延长锁的过期时间。
    - 这样只要客户端实例正常运行，锁就不会因过期而释放，直到客户端主动释放锁或实例宕机（此时看门狗停止，锁最终会过期）。
3. **避免非原子操作**：确保设置锁和设置过期时间是原子操作。
###### 8. Redis 分布式锁和 Zookeeper 分布式锁的区别?

|特性|Redis 分布式锁|Zookeeper 分布式锁|
|---|---|---|
|**实现原理**​|基于 `SET NX PX`命令或 Redlock 算法|基于临时顺序节点 (Ephemeral Sequential Znodes)|
|**一致性模型**​|**AP**：优先保证可用性，异步复制下可能丢锁|**CP**：优先保证一致性，强一致性|
|**性能**​|**高**：基于内存操作|**较低**：需要保持节点间状态一致|
|**锁释放机制**​|依赖过期时间，非自然释放|会话结束（Session）或连接断开时，临时节点自动删除，自然释放|
|**等待锁机制**​|客户端需自旋重试|可基于 Watch 机制监听前序节点，实现顺序排队和事件通知|
|**复杂性**​|相对简单，但处理锁续期、集群故障较复杂|相对复杂，但原生支持临时节点和事件监听，避免了自旋|
**选择建议**：
- 追求**高性能、高并发**，且能容忍极端情况下极低概率的锁失效，可选**Redis**。
- 要求**高可靠性、强一致性**，业务场景对锁的绝对安全要求高，可选**Zookeeper**。
###### 9. 如何实现可重入的分布式锁?
可重入锁是指**同一个线程或客户端可以多次获取同一把锁而不会发生死锁**。实现要点是记录锁的持有者（owner）和重入次数（count）。
**Redis实现方案（使用Hash结构+Lua脚本）**：
```java
// 加锁Lua脚本
String lockScript = 
    "if (redis.call('exists', KEYS[1]) == 0) then " + // 锁不存在
    "   redis.call('hset', KEYS[1], ARGV[1], 1); " +  // 设置持有者，重入次数为1
    "   redis.call('pexpire', KEYS[1], ARGV[2]); " +   // 设置过期时间
    "   return 1; " +                                  // 返回成功
    "end; " +
    "if (redis.call('hexists', KEYS[1], ARGV[1]) == 1) then " + // 锁存在且是当前持有者
    "   redis.call('hincrby', KEYS[1], ARGV[1], 1); " + // 重入次数+1
    "   redis.call('pexpire', KEYS[1], ARGV[2]); " +    // 刷新过期时间
    "   return 1; " +
    "end; " +
    "return 0;" // 获取锁失败

// 解锁Lua脚本
String unlockScript = 
    "if (redis.call('hexists', KEYS[1], ARGV[1]) == 0) then " +
    "   return 0; " + // 锁不属于当前请求者
    "end; " +
    "local counter = redis.call('hincrby', KEYS[1], ARGV[1], -1); " + // 重入次数-1
    "if (counter > 0) then " +
    "   redis.call('pexpire', KEYS[1], ARGV[2]); " + // 重入次数>0，刷新过期时间
    "   return 1; " +
    "else " +
    "   redis.call('del', KEYS[1]); " + // 重入次数=0，删除锁
    "   return 1; " +
    "end; " +
    "return 0;"
```
**实现关键点**：
- **锁信息存储**：使用Redis Hash结构，field存储客户端唯一标识（如UUID+线程ID），value存储重入次数。
- **原子操作**：使用Lua脚本确保判断重入次数和修改操作的原子性。
- **Redisson框架**：Java生态中Redisson客户端提供了开箱即用的可重入锁`RLock`，其实现原理与上述方案类似。