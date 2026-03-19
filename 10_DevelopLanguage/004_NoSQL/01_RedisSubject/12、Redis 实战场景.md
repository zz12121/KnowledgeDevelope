###### 1. 你项目中Redis主要用来做什么？——高频面试引导问题

面试官问这个问题，其实是想验证你是否有**真实的Redis项目经验**，以及你是否能合理地评估Redis的适用场景。回答时要结合项目实际情况，选择几个核心场景展开说。

**常见Redis使用场景**：

| 场景分类 | 具体用途 | 典型业务 |
|---------|---------|---------|
| **缓存** | 热点数据缓存 | 商品详情、用户信息、配置数据 |
| **分布式锁** | 库存扣减、抢票、防止重复提交 | 秒杀、订单创建、接口防重 |
| **Token存储** | 用户登录态、Session | JWT黑名单、Token续期 |
| **计数器** | 接口限流、访问统计 | 日活、PV/UV、点赞数 |
| **全局唯一ID** | 订单号、流水号 | 分布式ID生成 |
| **排行榜** | 实时榜单、热度排序 | 销售榜、积分榜、游戏排名 |
| **延迟队列** | 订单超时取消、消息推送 | 定时任务、异步通知 |
| **分布式Session** | 多节点 session 共享 | Spring Session + Redis |
| **发布订阅** | 实时消息、事件通知 | 订单状态变更通知 |
| **点赞/关注** | 社交功能 | 微博点赞、粉丝数 |

**回答示例**：

> 我们项目里Redis主要用在几个地方：
> 1. **缓存热点数据**：商品详情、用户信息这些查询频繁但不常修改的数据放Redis，数据库压力降低70%以上
> 2. **分布式锁**：库存扣减、订单创建这些需要保证原子性的操作，用Redisson实现可重入锁
> 3. **Token存储**：用户登录态用Redis存，支持集群环境下Session共享，还有过期自动清理
> 4. **延迟队列**：订单创建后放入延迟队列，30分钟未支付自动取消
>
> 选Redis主要是考虑高性能+丰富的数据结构，业务场景匹配度高。

**面试加分项**：
- 能说出具体的数据结构选择（String/Hash/ZSet等）
- 能估算缓存命中率、内存使用量
- 能提到遇到的坑和解决方案（缓存穿透/击穿/雪崩）

---

###### 2. 如何使用 Redis 实现限流?
Redis 限流通过控制单位时间内的请求量来保护系统，主要有四种算法实现：

**固定窗口计数器**

用 `INCR` 统计固定时间窗口内的请求数，超过阈值则拒绝。实现最简单，但有个经典的窗口临界点问题——在两个窗口的边界时刻可能瞬间通过 2 倍阈值的请求。

```java
public boolean isAllowed(String key, int limit, int window) {
    String luaScript = 
        "local current = redis.call('GET', KEYS[1]) or 0\n" +
        "if tonumber(current) >= tonumber(ARGV[1]) then\n" +
        "    return 0\n" +
        "else\n" +
        "    if tonumber(current) == 0 then\n" +
        "        redis.call('SETEX', KEYS[1], ARGV[2], 1)\n" +
        "    else\n" +
        "        redis.call('INCR', KEYS[1])\n" +
        "    end\n" +
        "    return 1\n" +
        "end";
    
    Object result = jedis.eval(luaScript, 1, key, 
        String.valueOf(limit), String.valueOf(window));
    return "1".equals(result.toString());
}
```

**滑动窗口日志**

用有序集合存储请求时间戳，每次请求先移除窗口外的旧数据，再统计窗口内请求数。解决了临界点问题，但每个请求都要存一条记录，内存消耗较大。

```java
public boolean slidingWindow(String key, int limit, int windowSec) {
    long now = System.currentTimeMillis();
    long windowStart = now - windowSec * 1000L;
    
    String luaScript = 
        "redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1])\n" +
        "local count = redis.call('ZCARD', KEYS[1])\n" +
        "if count >= tonumber(ARGV[3]) then\n" +
        "    return 0\n" +
        "else\n" +
        "    redis.call('ZADD', KEYS[1], ARGV[2], ARGV[4])\n" +
        "    redis.call('EXPIRE', KEYS[1], ARGV[5])\n" +
        "    return 1\n" +
        "end";
    
    return jedis.eval(luaScript, 1, key, 
        String.valueOf(windowStart), String.valueOf(now),
        String.valueOf(limit), UUID.randomUUID().toString(),
        String.valueOf(windowSec + 1)) != null;
}
```

**令牌桶算法**

以恒定速率向桶中添加令牌，请求消耗令牌。允许突发流量（桶满时积累了令牌），适合需要应对流量波动的场景。

**漏桶算法**

以固定速率处理请求，超出容量的请求被丢弃或排队，流量绝对平滑，但不允许突发。

生产环境推荐直接使用 **Redisson 的 `RRateLimiter`**，分布式友好，基于滑动窗口：

```java
RRateLimiter limiter = redisson.getRateLimiter("myLimiter");
limiter.trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.MINUTES);
if (limiter.tryAcquire(1)) {
    // 请求通过
}
```

> 📖 [[../../../23_RedisKnowledge/11_实战场景/01、常见实战场景实现#1-限流|知识库：Redis 限流实现]]

---

###### 2. 如何使用 Redis 实现排行榜?
Redis 的**有序集合（ZSet）**是实现排行榜的理想选择，自动按 score 排序，支持高效的范围查询。

**核心命令**

- `ZADD key score member`：添加或更新成员分数
- `ZREVRANGE key start stop WITHSCORES`：按分数从高到低获取前 N 名
- `ZREVRANK key member`：获取成员排名（从 0 开始，0 = 第 1 名）
- `ZINCRBY key increment member`：为成员加分

**Java 完整实现**

```java
public class LeaderboardService {
    private Jedis jedis;
    private static final String LEADERBOARD_KEY = "game:leaderboard";
    
    // 更新用户分数
    public void updateScore(String userId, double score) {
        jedis.zadd(LEADERBOARD_KEY, score, userId);
    }
    
    // 获取前 N 名排行榜
    public List<Player> getTopN(int n) {
        Set<Tuple> results = jedis.zrevrangeWithScores(LEADERBOARD_KEY, 0, n-1);
        return results.stream().map(tuple -> 
            new Player(tuple.getElement(), tuple.getScore())).collect(Collectors.toList());
    }
    
    // 获取用户排名（1-based）
    public Long getUserRank(String userId) {
        Long rank = jedis.zrevrank(LEADERBOARD_KEY, userId);
        return rank != null ? rank + 1 : null;
    }
    
    // 分页查询
    public List<Player> getByPage(int page, int size) {
        int start = (page - 1) * size;
        int end = start + size - 1;
        Set<Tuple> results = jedis.zrevrangeWithScores(LEADERBOARD_KEY, start, end);
        return results.stream().map(tuple -> 
            new Player(tuple.getElement(), tuple.getScore())).collect(Collectors.toList());
    }
}
```

**高级特性**：多维度排行榜用不同 key 区分（如日榜 `leaderboard:daily:20260315`、总榜 `leaderboard:total`）；临时排行榜设置 TTL 自动过期；分数相同时 ZSet 默认按 member 字典序排序，可以设计组合 score（分数左移 + 时间戳）实现相同分数按时间先后排序。

> 📖 [[../../../23_RedisKnowledge/11_实战场景/01、常见实战场景实现#2-排行榜|知识库：Redis 排行榜实现]]

---

###### 3. 如何使用 Redis 实现签到功能?
签到功能需要记录每日状态并统计连续天数，**Bitmap（位图）**是最佳选择——每个用户每月仅需约 4 字节的存储空间。

```java
public class SignInService {
    private static final String SIGN_KEY_PREFIX = "sign:";
    
    // 用户签到：以日期在月份中的天数减 1 作为 offset
    public boolean signIn(String userId, LocalDate date) {
        String key = getKey(userId, date.getYear(), date.getMonthValue());
        int offset = date.getDayOfMonth() - 1;
        return jedis.setbit(key, offset, true);
    }
    
    // 统计当月签到天数（BITCOUNT 一次搞定）
    public long getSignCount(String userId, int year, int month) {
        String key = getKey(userId, year, month);
        return jedis.bitcount(key);
    }
    
    // 计算连续签到天数（从今天向前遍历）
    public int getContinuousDays(String userId) {
        LocalDate today = LocalDate.now();
        int continuous = 0;
        
        for (int i = 0; i < 30; i++) {
            LocalDate date = today.minusDays(i);
            String key = getKey(userId, date.getYear(), date.getMonthValue());
            int offset = date.getDayOfMonth() - 1;
            
            if (jedis.getbit(key, offset)) {
                continuous++;
            } else if (i > 0) {  // 今天未签到不算断签，从昨天开始断才算
                break;
            }
        }
        return continuous;
    }
    
    private String getKey(String userId, int year, int month) {
        return SIGN_KEY_PREFIX + userId + ":" + year + "-" + month;
    }
}
```

**位图优势**：极致存储效率，1 亿用户一天的签到数据只需约 12MB；`BITCOUNT` 统计签到天数是 O(N)（N 是字节数），月签到约 4 字节，接近 O(1)；多用户签到数据可以用 `BITOP AND` 求交集（同时签到的用户）、`BITOP OR` 求并集。

> 📖 [[../../../23_RedisKnowledge/11_实战场景/01、常见实战场景实现#3-签到统计|知识库：Redis 签到功能实现]]

---

###### 4. 如何使用 Redis 实现点赞功能?
点赞功能要处理三件事：状态记录、计数、防重复。采用 **Hash + Set 组合方案**：Hash 存储点赞计数，Set 记录用户是否已点赞（防重）。

```java
public class LikeService {
    private static final String LIKE_KEY_PREFIX = "like:";
    private static final String USER_LIKED_PREFIX = "user_liked:";
    
    // 点赞 / 取消点赞（幂等操作）
    public boolean toggleLike(String userId, String targetType, String targetId) {
        String likeKey = LIKE_KEY_PREFIX + targetType + ":" + targetId;
        String userLikedKey = USER_LIKED_PREFIX + userId;
        String field = targetType + ":" + targetId;
        
        Transaction tx = jedis.multi();
        if (jedis.sismember(userLikedKey, field)) {
            // 已点赞 → 取消
            tx.hincrBy(likeKey, "count", -1);
            tx.srem(userLikedKey, field);
        } else {
            // 未点赞 → 点赞
            tx.hincrBy(likeKey, "count", 1);
            tx.sadd(userLikedKey, field);
            tx.hset(likeKey, "last_liked", String.valueOf(System.currentTimeMillis()));
        }
        tx.exec();
        return true;
    }
    
    // 获取点赞数
    public long getLikeCount(String targetType, String targetId) {
        String likeKey = LIKE_KEY_PREFIX + targetType + ":" + targetId;
        String count = jedis.hget(likeKey, "count");
        return count != null ? Long.parseLong(count) : 0;
    }
    
    // 检查用户是否已点赞
    public boolean hasLiked(String userId, String targetType, String targetId) {
        String userLikedKey = USER_LIKED_PREFIX + userId;
        return jedis.sismember(userLikedKey, targetType + ":" + targetId);
    }
}
```

**数据结构设计**：`like:post:123` → Hash，存 `{count: 45, last_liked: 时间戳}`；`user_liked:user456` → Set，存该用户点过赞的所有内容 ID（`["post:123", "comment:789"]`）。热点内容的点赞数据定期同步到数据库，冷数据可以设置 TTL 淘汰。

> 📖 [[../../../23_RedisKnowledge/11_实战场景/01、常见实战场景实现#4-点赞系统|知识库：Redis 点赞功能实现]]

---

###### 5. 如何使用 Redis 实现共同好友?
共同好友基于**集合运算**，Redis 的 `SINTER`（交集）命令天然支持这个场景。

```java
public class FriendService {
    private static final String FRIEND_KEY_PREFIX = "friends:";
    
    // 添加好友关系（双向）
    public void addFriend(String userId, String friendId) {
        Transaction tx = jedis.multi();
        tx.sadd(FRIEND_KEY_PREFIX + userId, friendId);
        tx.sadd(FRIEND_KEY_PREFIX + friendId, userId);
        tx.exec();
    }
    
    // 获取共同好友
    public Set<String> getCommonFriends(String userId1, String userId2) {
        return jedis.sinter(FRIEND_KEY_PREFIX + userId1, FRIEND_KEY_PREFIX + userId2);
    }
    
    // 好友推荐（好友的好友）
    public Set<String> getFriendRecommendations(String userId, int maxResults) {
        String userKey = FRIEND_KEY_PREFIX + userId;
        Set<String> friends = jedis.smembers(userKey);
        
        Pipeline pipeline = jedis.pipelined();
        for (String friend : friends) {
            pipeline.smembers(FRIEND_KEY_PREFIX + friend);
        }
        List<Object> results = pipeline.syncAndReturnAll();
        
        Set<String> recommendations = new HashSet<>();
        for (Object result : results) {
            @SuppressWarnings("unchecked")
            Set<String> friendFriends = (Set<String>) result;
            recommendations.addAll(friendFriends);
        }
        
        recommendations.removeAll(friends);  // 排除已有好友
        recommendations.remove(userId);      // 排除自己
        
        return recommendations.stream().limit(maxResults).collect(Collectors.toSet());
    }
}
```

**性能优化**：获取好友的好友列表时用 Pipeline 批量获取，减少网络往返；大集合运算避免在主线程直接 `SINTER`，可以用 `SINTERSTORE` 存入临时 key 后异步处理；好友推荐结果可以短期缓存，避免重复计算。

> 📖 [[../../../23_RedisKnowledge/11_实战场景/01、常见实战场景实现#5-共同好友|知识库：Redis 共同好友实现]]

---

###### 6. 如何使用 Redis 实现附近的人?
Redis 的 **GEO 地理位置**功能基于有序集合和 Geohash 算法，支持半径查询和距离计算，天然适合"附近的人"场景。

```java
public class NearbyService {
    private static final String LOCATION_KEY = "user:locations";
    
    // 更新用户位置
    public void updateLocation(String userId, double longitude, double latitude) {
        jedis.geoadd(LOCATION_KEY, longitude, latitude, userId);
    }
    
    // 查找附近的人（指定经纬度和半径）
    public List<GeoRadiusResponse> findNearbyUsers(double longitude, double latitude,
                                                   double radius, GeoUnit unit) {
        return jedis.georadius(LOCATION_KEY, longitude, latitude, radius, unit, 
            GeoRadiusParam.geoRadiusParam().withDist().withCoord().sortAscending());
    }
    
    // 获取两人之间的距离
    public Double getDistance(String userId1, String userId2, GeoUnit unit) {
        return jedis.geodist(LOCATION_KEY, userId1, userId2, unit);
    }
    
    // 分页查询附近的人
    public List<GeoRadiusResponse> findNearbyUsersPaged(double longitude, double latitude,
                                                       double radius, GeoUnit unit,
                                                       int page, int size) {
        int offset = (page - 1) * size;
        return jedis.georadius(LOCATION_KEY, longitude, latitude, radius, unit,
            GeoRadiusParam.geoRadiusParam()
                .withDist().withCoord().sortAscending().limit(offset + size)
        ).subList(offset, offset + size);
    }
}
```

**GEO 底层原理**：Geohash 把二维经纬度转换为一维整数，空间相近的点 Geohash 值相近，因此用 Sorted Set 按 Geohash 值排序后，范围查询就变成了一维区间查询，效率很高。注意 Redis 7.0+ 推荐使用 `GEOSEARCH` 命令（支持圆形和矩形两种范围），`GEORADIUS` 已被标记为废弃。

> 📖 [[../../../23_RedisKnowledge/11_实战场景/01、常见实战场景实现#6-附近的人|知识库：Redis 附近的人实现]]

---

###### 7. 如何使用 Redis 实现购物车?
购物车需要支持商品增删改查，结构是"用户 → 多个商品 → 各自数量"，**Hash 结构**最合适：key 是 `cart:userId`，field 是商品 ID，value 是数量。

```java
public class ShoppingCartService {
    private static final String CART_KEY_PREFIX = "cart:";
    private static final int CART_EXPIRE_DAYS = 30;
    
    // 添加/更新商品数量
    public void addItem(String userId, String productId, int quantity) {
        String cartKey = CART_KEY_PREFIX + userId;
        if (quantity <= 0) {
            jedis.hdel(cartKey, productId);  // 数量为 0 直接移除
        } else {
            jedis.hset(cartKey, productId, String.valueOf(quantity));
        }
        jedis.expire(cartKey, CART_EXPIRE_DAYS * 86400);  // 刷新过期时间
    }
    
    // 获取购物车商品总数量
    public int getCartItemCount(String userId) {
        return jedis.hgetAll(CART_KEY_PREFIX + userId)
            .values().stream().mapToInt(Integer::parseInt).sum();
    }
    
    // 获取购物车所有商品
    public Map<String, Integer> getCartItems(String userId) {
        return jedis.hgetAll(CART_KEY_PREFIX + userId).entrySet().stream()
            .collect(Collectors.toMap(Map.Entry::getKey, 
                e -> Integer.parseInt(e.getValue())));
    }
    
    // 清空购物车
    public void clearCart(String userId) {
        jedis.del(CART_KEY_PREFIX + userId);
    }
    
    // 合并购物车（用户登录时将临时购物车合并到账号购物车）
    public void mergeCarts(String userId, Map<String, Integer> tempCart) {
        String cartKey = CART_KEY_PREFIX + userId;
        Pipeline pipeline = jedis.pipelined();
        for (Map.Entry<String, Integer> entry : tempCart.entrySet()) {
            pipeline.hincrBy(cartKey, entry.getKey(), entry.getValue());
        }
        pipeline.expire(cartKey, CART_EXPIRE_DAYS * 86400);
        pipeline.sync();
    }
}
```

**设计注意点**：购物车是高频读写数据，适合全量存 Redis；下单时再查数据库校验库存和最新价格，不要把价格存购物车里（价格可能变）；合并购物车时用 `HINCRBY` 累加数量而不是简单覆盖。

> 📖 [[../../../23_RedisKnowledge/11_实战场景/01、常见实战场景实现#7-购物车|知识库：Redis 购物车实现]]

---

###### 8. 如何使用 Redis 实现秒杀系统?
秒杀的核心问题是**防超卖**和**高并发**，需要把"检查库存 + 扣减库存 + 记录用户"这三步做成原子操作，Lua 脚本是标准答案。

```java
public class SecKillService {
    private static final String STOCK_KEY_PREFIX = "seckill:stock:";
    private static final String USER_KEY_PREFIX = "seckill:users:";
    
    // 初始化秒杀库存
    public void initStock(String productId, int stock) {
        jedis.set(STOCK_KEY_PREFIX + productId, String.valueOf(stock));
    }
    
    // 秒杀核心逻辑（Lua 脚本保证原子性）
    public boolean secKill(String userId, String productId) {
        String luaScript = 
            "local stockKey = KEYS[1]\n" +
            "local userKey = KEYS[2]\n" +
            "local userId = ARGV[1]\n" +
            "\n" +
            "-- 防重：检查用户是否已参与\n" +
            "if redis.call('SISMEMBER', userKey, userId) == 1 then\n" +
            "    return 0\n" +
            "end\n" +
            "\n" +
            "-- 检查库存\n" +
            "local stock = tonumber(redis.call('GET', stockKey))\n" +
            "if not stock or stock <= 0 then\n" +
            "    return 0\n" +
            "end\n" +
            "\n" +
            "-- 原子扣减库存 + 记录用户\n" +
            "redis.call('DECR', stockKey)\n" +
            "redis.call('SADD', userKey, userId)\n" +
            "return 1";
        
        Object result = jedis.eval(luaScript, 2, 
            STOCK_KEY_PREFIX + productId, 
            USER_KEY_PREFIX + productId,
            userId);
        return "1".equals(result.toString());
    }
    
    public int getRemainingStock(String productId) {
        String stock = jedis.get(STOCK_KEY_PREFIX + productId);
        return stock != null ? Integer.parseInt(stock) : 0;
    }
}
```

**秒杀优化策略**：活动开始前提前**库存预热**，将库存数据加载到 Redis；入口处加**限流**（如令牌桶），拦截大量无效请求；秒杀成功后**异步处理**订单（发消息队列），避免同步调用数据库导致 Redis 等待；还可以用消息队列做**流量削峰**，把秒杀请求排队处理。

> 📖 [[../../../23_RedisKnowledge/11_实战场景/01、常见实战场景实现#8-秒杀系统|知识库：Redis 秒杀系统实现]]

---

###### 9. 如何使用 Redis 实现分布式 Session?
分布式 Session 解决多实例间的用户状态共享问题，Redis 用 Hash 结构存储 Session 数据，UUID 作为 Session ID。

```java
public class RedisSessionManager {
    private static final String SESSION_KEY_PREFIX = "session:";
    private static final int SESSION_TIMEOUT = 30 * 60;  // 30 分钟
    
    // 创建 Session
    public String createSession(User user) {
        String sessionId = UUID.randomUUID().toString();
        String sessionKey = SESSION_KEY_PREFIX + sessionId;
        
        Map<String, String> sessionData = new HashMap<>();
        sessionData.put("userId", user.getId());
        sessionData.put("username", user.getUsername());
        sessionData.put("loginTime", String.valueOf(System.currentTimeMillis()));
        
        jedis.hset(sessionKey, sessionData);
        jedis.expire(sessionKey, SESSION_TIMEOUT);
        return sessionId;
    }
    
    // 获取 Session（同时刷新过期时间）
    public Session getSession(String sessionId) {
        String sessionKey = SESSION_KEY_PREFIX + sessionId;
        if (!jedis.exists(sessionKey)) {
            return null;
        }
        jedis.expire(sessionKey, SESSION_TIMEOUT);  // 滑动过期
        
        Map<String, String> data = jedis.hgetAll(sessionKey);
        Session session = new Session();
        session.setId(sessionId);
        session.setUserId(data.get("userId"));
        session.setUsername(data.get("username"));
        session.setLoginTime(Long.parseLong(data.get("loginTime")));
        return session;
    }
    
    // 更新 Session 属性
    public void updateSessionAttribute(String sessionId, String key, String value) {
        String sessionKey = SESSION_KEY_PREFIX + sessionId;
        jedis.hset(sessionKey, key, value);
        jedis.expire(sessionKey, SESSION_TIMEOUT);
    }
    
    // 销毁 Session（登出）
    public void invalidateSession(String sessionId) {
        jedis.del(SESSION_KEY_PREFIX + sessionId);
    }
}
```

**最佳实践**：每次访问都刷新过期时间（滑动过期），保持活跃用户的登录状态；Session ID 用 UUID 随机生成，防止暴力猜测；敏感操作（如支付）前建议二次验证，不要完全依赖 Session 状态；生产环境更推荐 Spring Session + Redis 的集成方案，自动拦截 HTTP 请求，对应用代码透明。

> 📖 [[../../../23_RedisKnowledge/11_实战场景/01、常见实战场景实现#9-分布式-session|知识库：Redis 分布式 Session 实现]]

---

###### 10. 如何使用 Redis 实现唯一 ID 生成?
分布式 ID 需要保证全局唯一和趋势递增，Redis 的原子 `INCR` 命令是核心。

```java
public class IdGeneratorService {
    private static final String ID_KEY_PREFIX = "id:generator:";
    
    // 基于时间戳 + 序列号的 ID 生成
    public long generateTimestampId(String businessType) {
        String key = ID_KEY_PREFIX + businessType;
        long timestamp = System.currentTimeMillis();
        long sequence = jedis.incr(key);
        jedis.expire(key, 1);  // 每秒重置序列号
        
        // 时间戳 17 位 + 序列号 4 位
        return Long.parseLong(timestamp + String.format("%04d", sequence % 10000));
    }
    
    // 号段模式（减少 Redis 访问次数）
    public class SegmentIdGenerator {
        private long current = 0;
        private long max = 0;
        private final int step = 1000;  // 每次预申请 1000 个 ID
        
        public synchronized long nextId(String businessType) {
            if (current >= max) {
                // 号段用完，批量申请下一段
                String luaScript = 
                    "local key = KEYS[1]\n" +
                    "local step = tonumber(ARGV[1])\n" +
                    "local max = redis.call('INCRBY', key, step)\n" +
                    "return {max - step + 1, max}";
                
                @SuppressWarnings("unchecked")
                List<Long> result = (List<Long>) jedis.eval(luaScript, 
                    1, ID_KEY_PREFIX + businessType, String.valueOf(step));
                
                current = result.get(0);
                max = result.get(1);
            }
            return current++;
        }
    }
}
```

**方案对比**：直接 `INCR` 简单可靠但每次都要访问 Redis，高并发下有瓶颈；**号段模式**每次批量申请一段 ID 缓存在本地，减少 Redis 访问次数，性能大幅提升，是美团 Leaf 等主流方案的核心思路；如果需要更高可用性（Redis 挂了也能生成 ID），可以用 Snowflake 算法（基于机器 ID + 时间戳 + 序列号），不依赖外部存储。

> 📖 [[../../../23_RedisKnowledge/11_实战场景/01、常见实战场景实现#10-唯一-id-生成|知识库：Redis 唯一 ID 生成]]

---

###### 11. 如何使用 Redis 实现推荐系统?
推荐系统利用 Redis 的**集合运算**和**有序集合**实现用户行为记录和物品相似度计算。

```java
public class RecommendationService {
    private static final String USER_VIEW_PREFIX = "user:view:";
    private static final String ITEM_SIM_PREFIX = "item:sim:";
    
    // 记录用户行为（ZSet，按时间戳排序，保留最近 1000 条）
    public void recordUserAction(String userId, String itemId, String actionType, double weight) {
        String userActionKey = USER_VIEW_PREFIX + userId;
        jedis.zadd(userActionKey, System.currentTimeMillis(), 
            itemId + ":" + actionType + ":" + weight);
        jedis.zremrangeByRank(userActionKey, 0, -1001);  // 只保留最新 1000 条
    }
    
    // 基于物品的协同过滤推荐
    public List<String> getItemBasedRecommendations(String userId, int maxResults) {
        Set<String> userActions = jedis.zrange(USER_VIEW_PREFIX + userId, -10, -1);
        
        Pipeline pipeline = jedis.pipelined();
        for (String action : userActions) {
            String itemId = action.split(":")[0];
            pipeline.zrevrange(ITEM_SIM_PREFIX + itemId, 0, maxResults - 1);
        }
        
        List<Object> results = pipeline.syncAndReturnAll();
        Set<String> recommendations = new HashSet<>();
        for (Object result : results) {
            @SuppressWarnings("unchecked")
            recommendations.addAll((Set<String>) result);
        }
        
        return new ArrayList<>(recommendations).subList(0, 
            Math.min(maxResults, recommendations.size()));
    }
    
    // 计算物品相似度（Jaccard 相似度）
    public void calculateItemSimilarity(String itemId1, String itemId2) {
        String viewers1 = USER_VIEW_PREFIX + "items:" + itemId1;
        String viewers2 = USER_VIEW_PREFIX + "items:" + itemId2;
        
        Long commonUsers = jedis.sinterstore("temp:common", viewers1, viewers2);
        Long unionUsers = jedis.sunionstore("temp:union", viewers1, viewers2);
        
        double similarity = commonUsers.doubleValue() / unionUsers.doubleValue();
        jedis.zadd(ITEM_SIM_PREFIX + itemId1, similarity, itemId2);
        jedis.zadd(ITEM_SIM_PREFIX + itemId2, similarity, itemId1);
        
        jedis.del("temp:common", "temp:union");
    }
}
```

**扩展方向**：用时间衰减函数对历史行为降权（近期行为权重更高）；用 Hash 存储用户标签画像，支持基于内容的过滤；离线计算物品相似度矩阵（可以用 Spark），结果存到 Redis ZSet 里，在线推荐只做查询。Redis 在推荐系统中主要承担**特征存储**和**在线计算结果缓存**的角色。

> 📖 [[../../../23_RedisKnowledge/11_实战场景/01、常见实战场景实现#11-推荐系统|知识库：Redis 推荐系统实现]]

---

###### 12. 如何使用 Redis 实现搜索自动补全?
搜索补全基于**前缀匹配**，Redis 的有序集合 + `ZRANGEBYLEX` 命令支持字典序范围查询，非常适合这个场景。

```java
public class AutoCompleteService {
    private static final String SUGGESTION_KEY = "search:suggestions";
    private static final String SCORE_KEY = "search:score";
    
    // 添加搜索词并建立前缀索引
    public void addSearchTerm(String term) {
        // 将词条的所有前缀都存入 SUGGESTION_KEY（score 统一为 0，靠字典序排）
        for (int i = 1; i <= term.length(); i++) {
            jedis.zadd(SUGGESTION_KEY, 0, term.substring(0, i));
        }
        // 记录完整词条的搜索热度
        jedis.zincrby(SCORE_KEY, 1, term);
    }
    
    // 获取补全建议（按热度排序）
    public List<String> getSuggestions(String prefix, int maxResults) {
        // ZRANGEBYLEX 获取以 prefix 开头的所有词条
        Set<String> matched = jedis.zrangeByLex(SUGGESTION_KEY, 
            "[" + prefix, "[" + prefix + "\xff");
        
        // 从 SCORE_KEY 中按热度排序，过滤出匹配的词条
        return jedis.zrevrangeWithScores(SCORE_KEY, 0, -1).stream()
            .map(Tuple::getElement)
            .filter(term -> term.startsWith(prefix))
            .limit(maxResults)
            .collect(Collectors.toList());
    }
    
    // 热度+1
    public void incrementScore(String term) {
        jedis.zincrby(SCORE_KEY, 1, term);
    }
}
```

**注意**：`ZRANGEBYLEX` 要求所有元素的 score 相同（通常设为 0），才能按字典序排序，否则排序结果不对。`"\xff"` 是 ASCII 最大字符，用作范围上界，表示"以 prefix 开头的所有字符串"。

**性能优化**：限制前缀索引的最小长度（比如只存长度 ≥ 2 的前缀），减少存储；定期对热度分数做衰减，让结果反映最新趋势；中文场景需要先分词再建索引，比直接按字符前缀更准确。

> 📖 [[../../../23_RedisKnowledge/11_实战场景/01、常见实战场景实现#12-搜索自动补全|知识库：Redis 搜索自动补全实现]]

---

###### 13. 如何使用 Redis 实现社交关系链?
社交关系链包括关注、粉丝、共同关注等，用**有序集合（ZSet）**存储，以关注时间戳作为 score，支持按时间排序分页。

```java
public class SocialGraphService {
    private static final String FOLLOWING_PREFIX = "following:";
    private static final String FOLLOWERS_PREFIX = "followers:";
    
    // 关注用户（双向写入）
    public void follow(String userId, String targetUserId) {
        Transaction tx = jedis.multi();
        tx.zadd(FOLLOWING_PREFIX + userId, System.currentTimeMillis(), targetUserId);
        tx.zadd(FOLLOWERS_PREFIX + targetUserId, System.currentTimeMillis(), userId);
        tx.exec();
    }
    
    // 获取关注列表（按时间倒序分页）
    public List<String> getFollowing(String userId, int page, int size) {
        int start = (page - 1) * size;
        return new ArrayList<>(jedis.zrevrange(FOLLOWING_PREFIX + userId, start, start + size - 1));
    }
    
    // 获取共同关注（ZINTERSTORE 求交集）
    public List<String> getCommonFollowing(String userId1, String userId2) {
        String tempKey = "temp:common:following:" + userId1 + ":" + userId2;
        jedis.zinterstore(tempKey, FOLLOWING_PREFIX + userId1, FOLLOWING_PREFIX + userId2);
        List<String> common = new ArrayList<>(jedis.zrange(tempKey, 0, -1));
        jedis.del(tempKey);
        return common;
    }
    
    // 获取二度人脉（关注的人关注了谁）
    public List<String> getSecondDegreeConnections(String userId) {
        Set<String> directFollowing = jedis.zrange(FOLLOWING_PREFIX + userId, 0, -1);
        
        Pipeline pipeline = jedis.pipelined();
        for (String followedUser : directFollowing) {
            pipeline.zrange(FOLLOWING_PREFIX + followedUser, 0, -1);
        }
        
        List<Object> results = pipeline.syncAndReturnAll();
        Set<String> secondDegree = new HashSet<>();
        for (Object result : results) {
            @SuppressWarnings("unchecked")
            secondDegree.addAll((Set<String>) result);
        }
        
        secondDegree.removeAll(directFollowing);  // 排除已关注的人
        secondDegree.remove(userId);              // 排除自己
        return new ArrayList<>(secondDegree);
    }
}
```

**关系链设计模式**：写操作用事务保证双向关系的一致性（关注同时写 following 和 followers）；读操作用 Pipeline 批量获取减少网络往返；关系数据同时写 Redis 和数据库，Redis 提供读取性能，数据库保证持久化；热点用户（大 V）的粉丝列表可能非常大，查询时需要限制范围，避免 `ZRANGE` 返回几百万条数据。

> 📖 [[../../../23_RedisKnowledge/11_实战场景/01、常见实战场景实现#13-社交关系链|知识库：Redis 社交关系链实现]]

---

###### 14. Redis雪崩、击穿、穿透分别是什么？怎么解决？——高频面试引导问题

这是Redis最常考的三个问题，属于连环炮类型。

**概念区分**：

| 问题 | 原因 | 现象 | 解决思路 |
|------|------|------|---------|
| **雪崩** | 大量key同时过期 | 同一时刻大量请求打到DB | 随机过期时间 + 限流 + 熔断 |
| **击穿** | 热点key过期 | 热点key失效瞬间大量请求打到DB | 永不过期 + 互斥锁 |
| **穿透** | 查询不存在的数据 | 请求直接打到DB | 布隆过滤器 + 空值缓存 |

**回答示例**：

> 面试官好，这三个问题我都遇到过：
>
> 1. **雪崩**：我们做法是给key的过期时间加随机值，比如基础过期时间是1小时，随机偏移 ±10分钟，避免同时失效。同时做了限流，接口层限制同一秒内同一用户只能请求10次。
>
> 2. **击穿**：我们用的是"永不过期"方案 - 不设置TTL，而是用逻辑过期（在value里存过期时间戳），发现过期后用后台异步线程去重建缓存，同时加互斥锁防止并发重建。
>
> 3. **穿透**：我们用的是布隆过滤器 + 空值缓存。布隆过滤器存所有存在的key，不存在的直接过滤掉；对于查到的空结果，也缓存起来（但过期时间短一点，5分钟），避免反复查数据库。
>
> 追问：布隆过滤器误判怎么办？
> 布隆过滤器有误判率（假阳性），但不会漏判（假阴性）。我们通过业务上允许小概率误判来接受这个问题，或者用白名单机制兜底。

---

###### 15. Redis的数据类型有哪些？各自的应用场景？——高频面试引导问题

**八大数据类型**：

| 类型 | 数据结构 | 典型场景 | 命令示例 |
|------|---------|---------|---------|
| String | 简单字符串/数字/JSON | 缓存、计数器、分布式锁 | SET/GET/INCR |
| List | 双向链表 | 消息队列、排行榜、关注列表 | LPUSH/RPOP/LRANGE |
| Hash | 哈希表 | 对象存储、购物车 | HSET/HGET/HGETALL |
| Set | 无序集合 | 标签、好友、共同关注 | SADD/SMEMBERS/SINTER |
| ZSet | 有序集合 | 排行榜、延迟队列 | ZADD/ZRANGE/ZRANK |
| Bitmaps | 位图 | 用户签到、活跃用户统计 | SETBIT/GETBIT/BITCOUNT |
| HyperLogLog | 基数统计 | UV统计、DAU统计 | PFADD/PFCOUNT |
| Geospatial | 地理位置 | 附近的人、门店定位 | GEOADD/GEORADIUS |
| Stream | 消息队列 | 消息消费、事件流 | XADD/XREAD |

**回答示例**：

> 我们项目里主要用了这些：
> - **String**：最常用，存用户信息的JSON序列化、验证码、token
> - **Hash**：存购物车（userId -> {商品ID:数量}），比String好的是可以单独更新某个字段
> - **ZSet**：做排行榜，按分数排序查top N很方便
> - **List**：之前用过List做简单消息队列，后来换成Kafka了
> - **Bitmaps**：做用户签到统计，存一年才365位，比String省空间
>
> 追问：String的实现原理？
> Redis String底层是SDS（简单动态字符串），有预分配空间、惰性释放、O(1)获取长度，比C字符串更安全高效。

---

###### 16. Redis的持久化机制了解吗？RDB和AOF有什么区别？——高频面试引导问题

**两种持久化方式对比**：

| 特性 | RDB | AOF |
|------|-----|-----|
| 原理 | 定时快照，生成数据文件 | 记录所有写操作命令 |
| 文件大小 | 小（数据完整） | 大（累积命令） |
| 恢复速度 | 快（直接加载） | 慢（重放命令） |
| 数据完整性 | 可能丢数据 | 最多丢1秒 |
| 性能影响 | 阻塞（fork子进程） | 后台异步， |

**回答示例**：

> 我们生产环境用的是混合持久化（RDB + AOF），具体配置是：
> - 每1分钟key变化超过1万次时触发RDB快照
> - AOF用always模式（每条命令都刷盘），保证不丢数据
> - 同时开启两种，RDB做全量备份，AOF做增量
>
> 追问：RDB fork子进程会阻塞吗？
> 会阻塞，但Linux用了copy-on-write技术，子进程fork后共享父进程内存页，只有修改时才复制，所以正常情况阻塞时间很短。但如果父进程修改频繁，可能导致内存翻倍，有OOM风险。我们当时是限制RDB频率 + 半夜定时做来规避。

---

###### 17. Redis主从复制原理了解吗？——高频面试引导问题

**复制流程**：

1. **全量复制**：主fork子进程生成RDB → 子进程写磁盘 → 主同步给从 → 从加载
2. **增量复制**：主维护复制缓冲区 → 从断线重连 → 主从偏移量对比 → 发送增量命令

**回答示例**：

> 我们Redis Cluster用的是哨兵模式，一主二从三哨兵：
> - 主节点负责读写
> - 从节点做热备和读分流
> - 哨兵负责故障检测和自动切换
>
> 追问：主从同步延迟怎么办？
> 我们业务上做了妥协：要求强一致的数据不走Redis（比如支付相关），走数据库；Redis只用来缓存读多写少的数据，这样即使有点延迟也不影响业务。

---

###### 18. Redis Cluster是怎么工作的？——高频面试引导问题

**集群原理**：

- **槽位分配**：16384个槽，均匀分配到多个主节点
- **客户端重定向**：MOVED错误指引客户端到正确节点
- **故障转移**：主节点宕机后从节点自动选举

**回答示例**：

> 我们用的是Redis Cluster，6个节点（3主3从）：
> - 数据按CRC16(key) % 16384自动分片
> - 每个主节点有一个从节点，做故障自动切换
> - 官方客户端支持自动重定向，用起来和单机版差不多
>
> 追问：支持跨槽事务吗？
> 不支持，必须用Hash Tag强制路由到同一个槽。业务设计时要把会一起操作的数据用同一个key前缀。

---

###### 19. Redis和MySQL数据如何保持一致？——高频面试引导问题

这是Redis最常问的连环炮问题。

**常见方案**：

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| Cache Aside | 先删缓存，再更新DB | 简单 | 并发时可能脏读 |
| Read Through | 读时自动更新缓存 | 代码简单 | 首次并发可能击穿 |
| Write Through | 写时同步更新缓存 | 一致性好 | 性能差 |
| 延迟双删 | 删缓存 → 更新DB → 延迟再删 | 解决并发 | 增加延迟 |

**回答示例**：

> 我们项目用的是Cache Aside + 延迟双删：
> 1. 写操作：先删缓存，再更新DB，500ms后再删一次缓存
> 2. 读操作：先查缓存，没有则查DB并回填缓存
>
> 为什么不用Write Through？因为我们读多写少，写的时候不关心实时一致性，延迟双删已经足够。
>
> 追问：为什么先删缓存再更新DB？
> 如果先更新DB再删缓存，并发读可能把旧数据回填到缓存。而先删缓存的话，下一个读会直接查DB拿到新数据。

---

###### 20. Redis做分布式锁有什么问题？如何解决？——高频面试引导问题

**分布式锁要点**：

| 问题 | 解决方案 |
|------|---------|
| 单点故障 | RedLock（多节点） |
| TTL设置 | 看门狗自动续期 |
| Lua脚本 | 原子性释放 |
| 可重入 | ThreadLocal记录重入次数 |

**回答示例**：

> 我们用Redis做分布式锁，但和网上说的不太一样：
> 1. **锁key**：用`lock:业务唯一标识`（比如orderId）
> 2. **值**：用UUID + 线程ID，释放时比对，防止误删
> 3. **原子性**：用Lua脚本保证判断和删除原子
> 4. **看门狗**：Redisson框架自动续期，默认30秒过期，10秒续一次
>
> 追问：为什么不用ZooKeeper做分布式锁？
> Redis性能更好，但CAP里只能保证AP；ZooKeeper是CP，更适合金融场景。我们业务对性能要求高，用Redis就够了。

---

###### 21. Redis的内存淘汰策略有哪些？——高频面试引导问题

**淘汰策略分类**：

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| noeviction | 不淘汰，返回错误 | 数据必须完整 |
| volatile-lru | 只淘汰设置了TTL的，用LRU | 缓存场景 |
| allkeys-lru | 所有key用LRU淘汰 | 通用缓存 |
| volatile-random | 随机淘汰有TTL的 | - |
| allkeys-random | 随机淘汰所有 | - |
| volatile-ttl | 优先淘汰TTL短的 | 热点数据 |
| volatile-lfu | 用LFU算法淘汰 | 访问频率 |

**回答示例**：

> 我们用的是volatile-lru，因为：
> - 缓存数据都有TTL，到期自动淘汰
> - 不用allkeys-lru是为了保护持久化数据不被误删
> - 用LRU而不是LFU是因为我们访问模式不固定，LRU更通用
>
> 追问：LRU算法了解吗？
> Redis用的是近似LRU（采样淘汰），不是精确LRU。采样数默认16，值越大越精确但消耗也越大。我们设置的是`maxmemory-samples 10`来平衡。

---

###### 22. Redis的BigKey问题了解吗？——高频面试引导问题

**BigKey危害**：

- 内存不均匀，影响集群稳定性
- 阻塞主线程（慢查询）
- 网络带宽占用高

**回答示例**：

> 我们遇到过BigKey问题血的教训：
> - 一个Hash存了用户所有订单详情，几MB大小
> - 查一次就把Redis主线程阻塞了
>
> 解决方式：
> 1. **拆分**：大Hash按用户ID哈希取模拆成多个小Hash
> 2. **压缩**：序列化方式从JSON改成Protobuf，减少体积
> 3. **定期清理**：用SCAN代替KEYS，渐进式删除
> 4. **监控**：用`redis-cli --bigkeys`定期扫描，发现就告警

---

###### 23. Redis Pipeline有什么作用？——高频面试引导问题

**Pipeline vs 普通命令**：

| 方式 | 网络往返 | 原子性 |
|------|---------|-------|
| 普通 | 每个命令一次RTT | 每个命令原子 |
| Pipeline | 批量一次RTT | 非原子，Redis没报错就执行 |

**回答示例**：

> 我们在批量操作时用了Pipeline：
> - 批量查询用户信息：查100个用户，1次RTT vs 100次RTT
> - 批量更新库存：批量扣减库存
>
> 但要注意：
> 1. Pipeline不保证原子性，部分失败需要补偿
> 2. 命令太多要分批（比如1万条拆成10批）
> 3. MGET/MSET能搞定的场景优先用原生批量命令
