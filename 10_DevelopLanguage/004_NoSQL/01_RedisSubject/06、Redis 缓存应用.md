###### 1. 什么是缓存穿透?如何解决?
**缓存穿透**是指查询一个**缓存和数据库中都不存在**的数据，导致每次请求都会直接访问数据库，无法利用缓存优势。这通常由恶意攻击者构造大量不存在key的请求引发。
**解决方案：**
1. **布隆过滤器（Bloom Filter）**：在缓存前加一层布隆过滤器，将所有可能存在的数据哈希到一个足够大的BitSet中。请求来时先经过布隆过滤器判断，若确定不存在则直接返回。
```java
    // Guava布隆过滤器示例
    BloomFilter<String> bloomFilter = BloomFilter.create(
        Funnels.stringFunnel(Charset.defaultCharset()), 
        1000000,  // 预期元素数量
        0.01      // 误判率
    );
    // 系统启动时预热合法key
    bloomFilter.put("validKey1");
    if (!bloomFilter.mightContain(key)) return null; // 直接拦截[3,13](@ref)
```
2. **缓存空对象**：当数据库查询为空时，仍将空结果（如null）进行缓存，并设置较短过期时间（如5分钟）。
```java
    if (data == null) {
        stringRedisTemplate.opsForValue().set(key, "", Duration.ofMinutes(5));
        return null;
    }[5](@ref)
```
3. **接口层校验**：对请求参数进行合法性校验（如ID是否为负数），拦截非法请求。
###### 2. 什么是缓存击穿?如何解决?
**缓存击穿**是指一个**热点Key突然失效**时，大量并发请求同时访问这个Key，请求直接穿透到数据库，导致数据库压力激增。
**解决方案：**
1. **互斥锁（Mutex Lock）**：当缓存失效时，只允许一个线程查询数据库并重建缓存，其他线程等待或重试。
```java
    // 基于Redis SETNX实现分布式锁
    private boolean tryLock(String key) {
        return Boolean.TRUE.equals(
            stringRedisTemplate.opsForValue()
                .setIfAbsent(key, "1", Duration.ofSeconds(10)) // 锁超时防死锁
        );
    }
    // 查询流程
    if (cacheValue == null) {
        if (tryLock(lockKey)) {
            try {
                // 双重检查，防止其他线程已重建
                cacheValue = getFromCache(key);
                if (cacheValue == null) {
                    // 查询数据库并重建缓存
                }
            } finally {
                releaseLock(lockKey);
            }
        } else {
            Thread.sleep(50); // 未获锁，稍后重试
            return queryById(id); // 递归重试
        }
    }[4,5](@ref)
```
2. **逻辑过期**：不设置物理过期时间，而是在Value中存储逻辑过期时间。当发现数据逻辑过期时，异步刷新缓存。
3. **热点数据永不过期**：对极热点数据设置为永不过期，通过后台任务异步更新。
###### 3. 什么是缓存雪崩?如何解决?
**缓存雪崩**是指**大量缓存数据在同一时间点大面积失效**，或Redis服务宕机，导致所有请求直接访问数据库，造成数据库压力过大甚至宕机。
**解决方案：**
1. **设置随机过期时间**：避免大量Key同时失效，在基础过期时间上增加随机值。
```java
    int expireTime = 30 + new Random().nextInt(10); // 30-40分钟随机
    redisTemplate.opsForValue().set(key, value, expireTime, TimeUnit.MINUTES);[7](@ref)
```
2. **多级缓存架构**：采用本地缓存（如Caffeine） + Redis分布式缓存，避免Redis单点故障。
3. **Redis高可用**：通过Redis集群或主从复制提升服务可用性。
4. **熔断降级**：引入Hystrix或Sentinel等组件，在数据库压力过大时快速失败或返回兜底数据。
###### 4. 什么是热点数据?如何处理?
**热点数据**是指被**极高频率访问**的数据。处理策略包括：
1. **缓存预热**：系统启动或低峰期提前加载热点数据到缓存。
2. **多级缓存**：本地缓存 + 分布式缓存，减少网络开销。
3. **热点探测与隔离**：实时监控Key访问频率，对热点Key进行特殊处理（如单独Redis实例）。
4. **缓存永不过期**+异步更新：对核心热点数据设置永不过期，通过后台任务更新。
###### 5. 布隆过滤器是什么?如何使用?
**布隆过滤器**是一种**空间效率高的概率型数据结构**，用于判断一个元素是否存在于集合中。它可能误判（假阳性），但不会漏判（假阴性）。
**原理**：使用一个比特数组和多个哈希函数。添加元素时，通过哈希函数将多个位置置1；查询时，若所有对应位都是1则认为元素可能存在。
**Java实现：**
```java
// Guava实现
BloomFilter<String> bloomFilter = BloomFilter.create(
    Funnels.stringFunnel(Charset.defaultCharset()),
    1000000,  // 预期元素数量
    0.01      // 可接受的误判率
);
// 添加元素
bloomFilter.put("key1");
// 判断存在性
if (bloomFilter.mightContain("key1")) {
    // 可能存在，继续后续查询
}[3,13](@ref)
```
**Redis集成**：可使用RedisBloom模块，在分布式环境中部署布隆过滤器。
###### 6. 缓存预热是什么?如何实现?
**缓存预热**指在**系统启动或低峰期**提前将热点数据加载到缓存，避免冷启动时大量请求直接访问数据库。
**实现方式：**
1. **启动时加载**：应用启动时执行初始化任务。
```java
    @Component
    public class CacheInit implements CommandLineRunner {
        @Override
        public void run(String... args) {
            List<HotData> hotDataList = fetchHotData();
            hotDataList.forEach(data -> 
                redisTemplate.opsForValue().set(
                    "hotkey:" + data.getId(), 
                    data, 
                    Duration.ofHours(1)
                )
            );
        }
    }[3](@ref)
```
2. **定时任务刷新**：通过定时任务在低峰期更新缓存。
3. **手动触发**：提供管理接口手动触发预热。
###### 7. 缓存更新策略有哪些?
常见缓存更新策略包括：
1. **Cache Aside Pattern**：先更新数据库，再删除缓存。
2. **Read/Write Through**：应用直接操作缓存，由缓存组件负责同步更新数据库。
3. **Write Behind Caching**：先更新缓存，异步批量更新数据库。
###### 8. Cache Aside Pattern 是什么?
这是**最常用的缓存模式**，流程如下：
- **读操作**：先读缓存，命中则返回；未命中则读数据库，写入缓存。
- **写操作**：先更新数据库，再**删除缓存**（非更新缓存）。
**优点**：简单有效，避免并发写导致的缓存数据不一致。
###### 9. Read/Write Through Pattern 是什么?
应用将**缓存作为主要数据源**，所有读写操作都先经过缓存：
- **读穿透**：缓存命中直接返回；未命中由缓存组件从数据库加载并写入缓存。
- **写穿透**：应用更新缓存，缓存组件同步更新数据库。
**优点**：代码简洁，缓存组件封装了复杂性。
###### 10. Write Behind Pattern 是什么?
**写操作**先更新缓存，然后**异步批量**更新数据库。
- **优点**：写入性能极高，适合写多读少场景。
- **缺点**：数据一致性较弱，可能丢失数据。
###### 11. 如何保证缓存和数据库的数据一致性?
保证一致性是复杂问题，常用方案：
1. **延迟双删**：更新数据库后，删除缓存，延迟几百毫秒再次删除。
2. **基于binlog的同步**：通过MySQL binlog订阅（如Canal）异步更新缓存。
3. **设置合理过期时间**：即使出现不一致，数据最终也会通过过期机制保持一致。
###### 12. 双写一致性问题如何解决?
**双写一致性**指同时更新缓存和数据库时如何保证一致性：
1. **先更新数据库，后删除缓存**：这是Cache Aside模式推荐的做法。
2. **使用分布式事务**：通过消息队列确保两个操作同时成功或失败，但性能开销大。
3. **版本号或时间戳**：在缓存数据中增加版本号，只有新版本数据才能覆盖旧版本。