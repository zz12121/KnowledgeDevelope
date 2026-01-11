###### 1. 如何使用 Redis 实现限流?
Redis限流通过控制单位时间内的请求量来保护系统，主要采用四种算法实现：
**固定窗口计数器（计数器算法）**
- **原理**：使用`INCR`命令统计固定时间窗口内的请求数，超过阈值则限流
- **Java实现**：
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
- **缺点**：存在窗口临界点问题，可能瞬间通过2倍阈值请求
**滑动窗口日志**
- **原理**：使用有序集合存储请求时间戳，移除窗口外数据后统计数量
- **实现**：
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
- **原理**：以恒定速率向桶中添加令牌，请求获取令牌后才被处理
- **优势**：允许突发流量，适合需要应对流量波动的场景
**漏桶算法**​
- **原理**：以固定速率处理请求，超出速率的请求被丢弃或排队
- **特点**：严格限制处理速率，保证流量平滑
生产环境推荐使用**Redisson**的`RRateLimiter`，它基于滑动窗口实现，支持分布式环境：
```java
RRateLimiter limiter = redisson.getRateLimiter("myLimiter");
limiter.trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.MINUTES);
if (limiter.tryAcquire(1)) {
    // 请求通过
}
```
###### 2. 如何使用 Redis 实现排行榜?
Redis的**有序集合(ZSET)是实现排行榜的理想选择，支持自动排序和高效范围查询
核心命令与应用**：
- `ZADD key score member`：添加或更新成员分数
- `ZREVRANGE key start stop WITHSCORES`：获取排名前N的成员
- `ZREVRANK key member`：获取成员排名（从高到低）
- `ZINCRBY key increment member`：增加成员分数
**Java完整实现**：
```java
public class LeaderboardService {
    private Jedis jedis;
    private static final String LEADERBOARD_KEY = "game:leaderboard";
    
    // 更新用户分数
    public void updateScore(String userId, double score) {
        jedis.zadd(LEADERBOARD_KEY, score, userId);
    }
    
    // 获取前N名排行榜
    public List<Player> getTopN(int n) {
        Set<Tuple> results = jedis.zrevrangeWithScores(LEADERBOARD_KEY, 0, n-1);
        return results.stream().map(tuple -> 
            new Player(tuple.getElement(), tuple.getScore())).collect(Collectors.toList());
    }
    
    // 获取用户排名
    public Long getUserRank(String userId) {
        Long rank = jedis.zrevrank(LEADERBOARD_KEY, userId);
        return rank != null ? rank + 1 : null; // 转换为1-based排名
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
**高级特性**：
- **多维度排行榜**：使用不同key区分（如日榜、周榜、总榜）
- **分数相同处理**：默认按成员字典序排序，可通过`ZREVRANGEBYSCORE`精细控制
- **过期策略**：为临时排行榜设置TTL，自动清理过期数据
###### 3. 如何使用 Redis 实现签到功能?
签到功能需要记录每日签到状态并统计连续签到天数，**位图(Bitmap)是最佳选择
位图签到实现**：
```java
public class SignInService {
    private static final String SIGN_KEY_PREFIX = "sign:";
    
    // 用户签到
    public boolean signIn(String userId, LocalDate date) {
        String key = getKey(userId, date.getYear(), date.getMonthValue());
        int offset = date.getDayOfMonth() - 1; // 日期作为偏移量
        
        return jedis.setbit(key, offset, true);
    }
    
    // 统计当月签到天数
    public long getSignCount(String userId, int year, int month) {
        String key = getKey(userId, year, month);
        return jedis.bitcount(key);
    }
    
    // 检查连续签到天数（改进版）
    public int getContinuousDays(String userId) {
        LocalDate today = LocalDate.now();
        int continuous = 0;
        
        for (int i = 0; i < 30; i++) { // 检查最近30天
            LocalDate date = today.minusDays(i);
            String key = getKey(userId, date.getYear(), date.getMonthValue());
            int offset = date.getDayOfMonth() - 1;
            
            if (jedis.getbit(key, offset)) {
                continuous++;
            } else if (i > 0) { // 今天没签到不算断签
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
**位图优势**：
- **极致存储效率**：每个用户每月仅需约4KB存储空间
- **快速统计**：`BITCOUNT`命令O(1)复杂度统计签到天数
- **内存优化**：稀疏位图自动压缩，节省内存
###### 4. 如何使用 Redis 实现点赞功能?
点赞功能需要处理点赞状态、计数和防重复，采用**哈希(Hash)和**集合(Set)**组合方案
```java
public class LikeService {
    private static final String LIKE_KEY_PREFIX = "like:";
    private static final String USER_LIKED_PREFIX = "user_liked:";
    
    // 点赞/取消点赞
    public boolean toggleLike(String userId, String targetType, String targetId) {
        String likeKey = LIKE_KEY_PREFIX + targetType + ":" + targetId;
        String userLikedKey = USER_LIKED_PREFIX + userId;
        String field = targetType + ":" + targetId;
        
        // 使用事务保证原子性
        Transaction tx = jedis.multi();
        
        if (jedis.sismember(userLikedKey, field)) {
            // 取消点赞
            tx.hincrBy(likeKey, "count", -1);
            tx.srem(userLikedKey, field);
        } else {
            // 点赞
            tx.hincrBy(likeKey, "count", 1);
            tx.sadd(userLikedKey, field);
            tx.hset(likeKey, "last_liked", System.currentTimeMillis());
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
    
    // 用户是否已点赞
    public boolean hasLiked(String userId, String targetType, String targetId) {
        String userLikedKey = USER_LIKED_PREFIX + userId;
        String field = targetType + ":" + targetId;
        return jedis.sismember(userLikedKey, field);
    }
}
```
**数据结构设计要点**：
- **哈希存储计数**：`like:post:123`→ `{count: 45, last_liked: 1640995200}`
- **集合记录用户行为**：`user_liked:user456`→ `["post:123", "comment:789"]`
- **过期策略**：为热点数据设置TTL，冷数据持久化到数据库
###### 5. 如何使用 Redis 实现共同好友?
共同好友基于**集合运算**，Redis提供高效的`SINTER`、`SUNION`、`SDIFF`命令
```java
public class FriendService {
    private static final String FRIEND_KEY_PREFIX = "friends:";
    
    // 添加好友关系
    public void addFriend(String userId, String friendId) {
        String userKey = FRIEND_KEY_PREFIX + userId;
        String friendKey = FRIEND_KEY_PREFIX + friendId;
        
        Transaction tx = jedis.multi();
        tx.sadd(userKey, friendId);
        tx.sadd(friendKey, userId); // 双向关系
        tx.exec();
    }
    
    // 获取共同好友
    public Set<String> getCommonFriends(String userId1, String userId2) {
        String key1 = FRIEND_KEY_PREFIX + userId1;
        String key2 = FRIEND_KEY_PREFIX + userId2;
        
        return jedis.sinter(key1, key2);
    }
    
    // 推荐好友（好友的好友）
    public Set<String> getFriendRecommendations(String userId, int maxResults) {
        String userKey = FRIEND_KEY_PREFIX + userId;
        Set<String> friends = jedis.smembers(userKey);
        
        // 获取所有好友的好友集合
        Pipeline pipeline = jedis.pipelined();
        for (String friend : friends) {
            pipeline.smembers(FRIEND_KEY_PREFIX + friend);
        }
        List<Object> results = pipeline.syncAndReturnAll();
        
        // 合并并排除已有好友
        Set<String> recommendations = new HashSet<>();
        for (Object result : results) {
            @SuppressWarnings("unchecked")
            Set<String> friendFriends = (Set<String>) result;
            recommendations.addAll(friendFriends);
        }
        
        recommendations.removeAll(friends);
        recommendations.remove(userId); // 排除自己
        
        return recommendations.stream().limit(maxResults).collect(Collectors.toSet());
    }
}
```
**性能优化**：
- **管道批量操作**：减少网络往返次数
- **集合运算优化**：避免大集合运算，可分批处理
- **缓存策略**：对结果进行短期缓存
###### 6. 如何使用 Redis 实现附近的人?
Redis的**GEO地理位置**功能基于有序集合实现，支持半径查询和距离计算
```java
public class NearbyService {
    private static final String LOCATION_KEY = "user:locations";
    
    // 更新用户位置
    public void updateLocation(String userId, double longitude, double latitude) {
        jedis.geoadd(LOCATION_KEY, longitude, latitude, userId);
    }
    
    // 查找附近的人（半径范围内）
    public List<GeoRadiusResponse> findNearbyUsers(double longitude, double latitude, 
                                                   double radius, GeoUnit unit) {
        return jedis.georadius(LOCATION_KEY, longitude, latitude, radius, unit, 
            GeoRadiusParam.geoRadiusParam().withDist().withCoord().sortAscending());
    }
    
    // 获取两人距离
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
                .withDist()
                .withCoord()
                .sortAscending()
                .limit(offset + size)
        ).subList(offset, offset + size);
    }
}
```
**GEO底层原理**：
- **Geohash编码**：将二维坐标转换为一维字符串，保留空间关系
- **有序集合存储**：Geohash作为score，实现快速范围查询
- **精度控制**：根据业务需求选择合适的Geohash精度级别
###### 7. 如何使用 Redis 实现购物车?
购物车需要支持商品增删改查和过期清理，**哈希(Hash)**结构最合适
```java
public class ShoppingCartService {
    private static final String CART_KEY_PREFIX = "cart:";
    private static final int CART_EXPIRE_DAYS = 30;
    
    // 添加商品到购物车
    public void addItem(String userId, String productId, int quantity) {
        String cartKey = CART_KEY_PREFIX + userId;
        
        if (quantity <= 0) {
            // 数量为0或负数时移除商品
            jedis.hdel(cartKey, productId);
        } else {
            jedis.hset(cartKey, productId, String.valueOf(quantity));
        }
        
        // 刷新过期时间
        jedis.expire(cartKey, CART_EXPIRE_DAYS * 24 * 60 * 60);
    }
    
    // 获取购物车商品数量
    public int getCartItemCount(String userId) {
        String cartKey = CART_KEY_PREFIX + userId;
        Map<String, String> cart = jedis.hgetAll(cartKey);
        return cart.values().stream().mapToInt(Integer::parseInt).sum();
    }
    
    // 获取购物车所有商品
    public Map<String, Integer> getCartItems(String userId) {
        String cartKey = CART_KEY_PREFIX + userId;
        Map<String, String> cart = jedis.hgetAll(cartKey);
        
        return cart.entrySet().stream()
            .collect(Collectors.toMap(Map.Entry::getKey, 
                e -> Integer.parseInt(e.getValue())));
    }
    
    // 清空购物车
    public void clearCart(String userId) {
        String cartKey = CART_KEY_PREFIX + userId;
        jedis.del(cartKey);
    }
    
    // 合并购物车（用户登录时合并临时购物车）
    public void mergeCarts(String userId, Map<String, Integer> tempCart) {
        String cartKey = CART_KEY_PREFIX + userId;
        
        Pipeline pipeline = jedis.pipelined();
        for (Map.Entry<String, Integer> entry : tempCart.entrySet()) {
            pipeline.hincrBy(cartKey, entry.getKey(), entry.getValue());
        }
        pipeline.expire(cartKey, CART_EXPIRE_DAYS * 24 * 60 * 60);
        pipeline.sync();
    }
}
```
**购物车设计考虑**：
- **数据持久化**：定期将购物车数据同步到数据库
- **库存校验**：下单前验证商品库存是否充足
- **价格同步**：商品价格变化时需重新计算
###### 8. 如何使用 Redis 实现秒杀系统?
秒杀系统核心是**库存原子操作**和**防超卖**，利用Redis的原子性命令
```java
public class SecKillService {
    private static final String STOCK_KEY_PREFIX = "seckill:stock:";
    private static final String USER_KEY_PREFIX = "seckill:users:";
    
    // 初始化秒杀库存
    public void initStock(String productId, int stock) {
        String stockKey = STOCK_KEY_PREFIX + productId;
        jedis.set(stockKey, String.valueOf(stock));
    }
    
    // 秒杀核心逻辑（Lua脚本保证原子性）
    public boolean secKill(String userId, String productId) {
        String luaScript = 
            "local stockKey = KEYS[1]\n" +
            "local userKey = KEYS[2]\n" +
            "local userId = ARGV[1]\n" +
            "local productId = ARGV[2]\n" +
            "\n" +
            "-- 检查用户是否已参与\n" +
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
            "-- 扣减库存并记录用户\n" +
            "redis.call('DECR', stockKey)\n" +
            "redis.call('SADD', userKey, userId)\n" +
            "return 1";
        
        Object result = jedis.eval(luaScript, 2, 
            STOCK_KEY_PREFIX + productId, 
            USER_KEY_PREFIX + productId,
            userId, productId);
        
        return "1".equals(result.toString());
    }
    
    // 获取剩余库存
    public int getRemainingStock(String productId) {
        String stockKey = STOCK_KEY_PREFIX + productId;
        String stock = jedis.get(stockKey);
        return stock != null ? Integer.parseInt(stock) : 0;
    }
}
```
**秒杀优化策略**：
- **库存预热**：活动开始前将库存加载到Redis
- **限流降级**：结合限流算法保护系统
- **异步处理**：秒杀成功后异步处理订单逻辑
- **缓存预热**：提前加载商品信息到缓存
###### 9. 如何使用 Redis 实现分布式 Session?
分布式Session解决多实例间的状态共享问题，Redis提供高性能存储方案
```java
public class RedisSessionManager {
    private static final String SESSION_KEY_PREFIX = "session:";
    private static final int SESSION_TIMEOUT = 30 * 60; // 30分钟
    
    // 创建Session
    public String createSession(User user) {
        String sessionId = UUID.randomUUID().toString();
        String sessionKey = SESSION_KEY_PREFIX + sessionId;
        
        Map<String, String> sessionData = new HashMap<>();
        sessionData.put("userId", user.getId());
        sessionData.put("username", user.getUsername());
        sessionData.put("loginTime", String.valueOf(System.currentTimeMillis()));
        sessionData.put("lastAccessTime", String.valueOf(System.currentTimeMillis()));
        
        jedis.hset(sessionKey, sessionData);
        jedis.expire(sessionKey, SESSION_TIMEOUT);
        
        return sessionId;
    }
    
    // 获取Session
    public Session getSession(String sessionId) {
        String sessionKey = SESSION_KEY_PREFIX + sessionId;
        
        // 刷新过期时间
        if (!jedis.exists(sessionKey)) {
            return null;
        }
        
        jedis.expire(sessionKey, SESSION_TIMEOUT);
        Map<String, String> sessionData = jedis.hgetAll(sessionKey);
        
        Session session = new Session();
        session.setId(sessionId);
        session.setUserId(sessionData.get("userId"));
        session.setUsername(sessionData.get("username"));
        session.setLoginTime(Long.parseLong(sessionData.get("loginTime")));
        session.setLastAccessTime(Long.parseLong(sessionData.get("lastAccessTime")));
        
        return session;
    }
    
    // 更新Session
    public void updateSessionAttribute(String sessionId, String key, String value) {
        String sessionKey = SESSION_KEY_PREFIX + sessionId;
        jedis.hset(sessionKey, key, value);
        jedis.expire(sessionKey, SESSION_TIMEOUT);
    }
    
    // 销毁Session
    public void invalidateSession(String sessionId) {
        String sessionKey = SESSION_KEY_PREFIX + sessionId;
        jedis.del(sessionKey);
    }
}
```
**Session管理最佳实践**：
- **合理超时**：平衡安全性和用户体验设置Session超时
- **数据序列化**：使用JSON或高效二进制序列化
- **安全考虑**：Session ID随机生成，防止猜测攻击
###### 10. 如何使用 Redis 实现唯一 ID 生成?
分布式ID生成需要保证全局唯一和趋势递增，Redis原子操作非常适合
```java
public class IdGeneratorService {
    private static final String ID_KEY_PREFIX = "id:generator:";
    
    // 基于时间戳的ID生成（Twitter Snowflake风格）
    public long generateTimestampId(String businessType) {
        String key = ID_KEY_PREFIX + businessType;
        long timestamp = System.currentTimeMillis();
        
        // 日期格式：yyyyMMddHHmmssSSS + 序列号
        String timestampStr = String.format("%d", timestamp);
        long sequence = jedis.incr(key);
        
        // 重置序列号（每秒重置）
        jedis.expire(key, 1);
        
        return Long.parseLong(timestampStr + String.format("%04d", sequence % 10000));
    }
    
    // 分段ID生成（预分配号段）
    public class SegmentIdGenerator {
        private String key;
        private long current = 0;
        private long max = 0;
        private final int step = 1000; // 每次获取1000个ID
        
        public synchronized long nextId(String businessType) {
            if (current >= max) {
                // 申请新号段
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
**ID生成方案对比**：
- **自增ID**：简单但暴露业务量信息
- **时间戳ID**：包含时间信息，适合时序查询
- **号段模式**：减少Redis访问，性能更高
###### 11. 如何使用 Redis 实现推荐系统?
推荐系统利用Redis的**集合运算**和**有序集合**实现用户行为分析和相似度计算
```java
public class RecommendationService {
    private static final String USER_VIEW_PREFIX = "user:view:";
    private static final String ITEM_SIM_PREFIX = "item:sim:";
    
    // 记录用户行为
    public void recordUserAction(String userId, String itemId, String actionType, double weight) {
        String userActionKey = USER_VIEW_PREFIX + userId;
        jedis.zadd(userActionKey, System.currentTimeMillis(), 
            itemId + ":" + actionType + ":" + weight);
        
        // 保留最近1000个行为
        jedis.zremrangeByRank(userActionKey, 0, -1001);
    }
    
    // 基于物品的协同过滤
    public List<String> getItemBasedRecommendations(String userId, int maxResults) {
        String userActionKey = USER_VIEW_PREFIX + userId;
        Set<String> userActions = jedis.zrange(userActionKey, -10, -1); // 最近10个行为
        
        // 获取相似物品
        Pipeline pipeline = jedis.pipelined();
        for (String action : userActions) {
            String itemId = action.split(":")[0];
            pipeline.zrevrange(ITEM_SIM_PREFIX + itemId, 0, maxResults - 1);
        }
        
        List<Object> results = pipeline.syncAndReturnAll();
        Set<String> recommendations = new HashSet<>();
        
        for (Object result : results) {
            @SuppressWarnings("unchecked")
            Set<String> similarItems = (Set<String>) result;
            recommendations.addAll(similarItems);
        }
        
        return new ArrayList<>(recommendations).subList(0, 
            Math.min(maxResults, recommendations.size()));
    }
    
    // 计算物品相似度
    public void calculateItemSimilarity(String itemId1, String itemId2) {
        String viewers1 = USER_VIEW_PREFIX + "items:" + itemId1;
        String viewers2 = USER_VIEW_PREFIX + "items:" + itemId2;
        
        // 计算Jaccard相似度
        Long commonUsers = jedis.sinterstore("temp:common", viewers1, viewers2);
        Long unionUsers = jedis.sunionstore("temp:union", viewers1, viewers2);
        
        double similarity = commonUsers.doubleValue() / unionUsers.doubleValue();
        jedis.zadd(ITEM_SIM_PREFIX + itemId1, similarity, itemId2);
        jedis.zadd(ITEM_SIM_PREFIX + itemId2, similarity, itemId1);
        
        jedis.del("temp:common", "temp:union");
    }
}
```
**推荐算法扩展**：
- **实时热度榜**：基于时间衰减的分数计算
- **用户画像**：使用哈希存储用户标签权重
- **深度学习**：Redis作为特征存储，支持在线推理
###### 12. 如何使用 Redis 实现搜索自动补全?
搜索补全基于**前缀匹配**，Redis的**有序集合**和**ZRANGEBYLEX**命令非常适合
```java
public class AutoCompleteService {
    private static final String SUGGESTION_KEY = "search:suggestions";
    private static final String SCORE_KEY = "search:score";
    
    // 添加搜索词
    public void addSearchTerm(String term) {
        // 使用分词生成前缀
        for (int i = 1; i <= term.length(); i++) {
            String prefix = term.substring(0, i);
            jedis.zadd(SUGGESTION_KEY, 0, prefix);
        }
        
        // 存储完整词条及其分数（热度）
        jedis.zincrby(SCORE_KEY, 1, term);
    }
    
    // 获取补全建议
    public List<String> getSuggestions(String prefix, int maxResults) {
        String start = prefix;
        String end = prefix + "\xff"; // 超出ASCII范围的字符
        
        // 获取匹配前缀的词条
        Set<String> prefixes = jedis.zrangeByLex(SUGGESTION_KEY, 
            "[" + start, "[" + end);
        
        // 根据热度排序返回建议
        Set<Tuple> suggestions = jedis.zrevrangeWithScores(SCORE_KEY, 0, -1);
        
        return suggestions.stream()
            .map(Tuple::getElement)
            .filter(term -> term.startsWith(prefix))
            .limit(maxResults)
            .collect(Collectors.toList());
    }
    
    // 增量更新词条热度
    public void incrementScore(String term) {
        jedis.zincrby(SCORE_KEY, 1, term);
    }
}
```
**性能优化技巧**：
- **前缀最小长度**：限制前缀索引的最小长度，减少存储
- **热度衰减**：定期对分数进行衰减，反映最新趋势
- **内存控制**：设置最大内存限制，避免无限增长
###### 13. 如何使用 Redis 实现社交关系链?
社交关系链包括关注、粉丝、好友等关系，**集合**和**有序集合**是核心数据结构
```java
public class SocialGraphService {
    private static final String FOLLOWING_PREFIX = "following:";
    private static final String FOLLOWERS_PREFIX = "followers:";
    private static final String FRIENDS_PREFIX = "friends:";
    
    // 关注用户
    public void follow(String userId, String targetUserId) {
        String followingKey = FOLLOWING_PREFIX + userId;
        String followersKey = FOLLOWERS_PREFIX + targetUserId;
        
        Transaction tx = jedis.multi();
        tx.zadd(followingKey, System.currentTimeMillis(), targetUserId);
        tx.zadd(followersKey, System.currentTimeMillis(), userId);
        tx.exec();
    }
    
    // 获取关注列表（带分页）
    public List<String> getFollowing(String userId, int page, int size) {
        String followingKey = FOLLOWING_PREFIX + userId;
        int start = (page - 1) * size;
        int end = start + size - 1;
        
        Set<String> following = jedis.zrevrange(followingKey, start, end);
        return new ArrayList<>(following);
    }
    
    // 获取共同关注
    public List<String> getCommonFollowing(String userId1, String userId2) {
        String key1 = FOLLOWING_PREFIX + userId1;
        String key2 = FOLLOWING_PREFIX + userId2;
        
        // 使用临时键存储交集
        String tempKey = "temp:common:following:" + userId1 + ":" + userId2;
        jedis.zinterstore(tempKey, key1, key2);
        
        Set<String> common = jedis.zrange(tempKey, 0, -1);
        jedis.del(tempKey);
        
        return new ArrayList<>(common);
    }
    
    // 获取二度人脉（朋友的朋友）
    public List<String> getSecondDegreeConnections(String userId) {
        String followingKey = FOLLOWING_PREFIX + userId;
        Set<String> directFollowing = jedis.zrange(followingKey, 0, -1);
        
        // 获取所有直接关注用户的关注列表
        Pipeline pipeline = jedis.pipelined();
        for (String followedUser : directFollowing) {
            pipeline.zrange(FOLLOWING_PREFIX + followedUser, 0, -1);
        }
        
        List<Object> results = pipeline.syncAndReturnAll();
        Set<String> secondDegree = new HashSet<>();
        
        for (Object result : results) {
            @SuppressWarnings("unchecked")
            Set<String> friendsOfFriends = (Set<String>) result;
            secondDegree.addAll(friendsOfFriends);
        }
        
        // 排除直接关注和自己
        secondDegree.removeAll(directFollowing);
        secondDegree.remove(userId);
        
        return new ArrayList<>(secondDegree);
    }
}
```
**关系链设计模式**：
- **读写分离**：写操作使用事务，读操作使用管道
- **数据同步**：定期将重要关系同步到数据库
- **缓存策略**：对热点用户的关系进行缓存