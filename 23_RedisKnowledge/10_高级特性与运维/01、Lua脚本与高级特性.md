# Lua 脚本与高级特性

## 1. Lua 脚本

### 1.1 为什么需要 Lua 脚本

Redis 本身支持 `MULTI/EXEC` 事务，但事务有一个明显的局限：它不支持条件判断，你没法在事务里根据某个 key 的值决定后续操作。Lua 脚本解决了这个问题，它让你可以在服务端执行**包含条件分支、循环的复杂多步操作**，并且整个脚本天然是原子的。

**核心价值三点**：
1. **原子性保证**：Lua 脚本在执行期间，Redis 的单线程模型保证不会有其他命令插进来，整个脚本作为一个不可分割的任务单元执行。
2. **减少网络往返**：多条命令打包成一个脚本，一次网络传输搞定，在高延迟网络下效果特别显著。
3. **复杂逻辑封装**：可以写条件判断、循环，实现分布式锁、限流、秒杀等场景，这些在纯命令层面根本无法原子实现。

### 1.2 源码视角：scriptingInit

Redis 启动时，`server.c` 的 `initServer` 函数会调用 `scriptingInit` 初始化 Lua 环境。这个函数主要做三件事：
1. 创建 Lua 虚拟机（`lua_State`）
2. 加载标准库（string/table/math 等）
3. 向 Lua 环境注入 `redis.call()` 和 `redis.pcall()` 两个桥接函数

有了这两个桥接函数，Lua 脚本才能调用 Redis 命令。

### 1.3 EVAL 命令

```redis
EVAL script numkeys key [key ...] arg [arg ...]
```

- `script`：Lua 脚本字符串
- `numkeys`：说明后面有几个是 key
- `key ...`：通过 `KEYS[1]`、`KEYS[2]` 访问
- `arg ...`：通过 `ARGV[1]`、`ARGV[2]` 访问

**执行流程**：
1. 收到 `EVAL` 命令，计算脚本的 SHA1 值
2. 查脚本缓存，命中则直接复用已编译版本
3. 未命中则编译脚本后执行
4. 将 Lua 返回值转为 RESP 格式返回给客户端

**脚本缓存**：用 `EVALSHA sha1 numkeys key... arg...` 可以直接用 SHA1 执行已缓存的脚本，避免每次传输脚本体，节省带宽。`SCRIPT LOAD script` 用于预加载脚本。

### 1.4 redis.call 与 redis.pcall 的区别

这两个函数都用于在 Lua 脚本里调用 Redis 命令，区别在于**错误处理方式**：

**`redis.call()`**：如果命令执行出错，直接抛出 Lua 错误，脚本立即终止，返回错误给客户端。

**`redis.pcall()`**：出错时不终止脚本，而是返回一个包含错误信息的 Lua 表，让脚本自己决定后续逻辑。

一般原则：对于必须成功的关键操作用 `redis.call()`（例如扣减库存），需要容错时用 `redis.pcall()`。

### 1.5 典型示例：Lua 实现限流

```lua
-- 固定窗口限流：每分钟最多 limit 次
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local current = tonumber(redis.call('GET', key) or "0")
if current + 1 > limit then
    return 0  -- 超过限制，拒绝
else
    redis.call('INCR', key)
    redis.call('EXPIRE', key, 60)
    return 1  -- 允许通过
end
```

```java
// Java 调用
String luaScript = "...";
Object result = jedis.eval(luaScript, 1, "rate:user:123", "10");
boolean allowed = Long.valueOf(1L).equals(result);
```

---

## 2. HyperLogLog

### 2.1 是什么，解决什么问题

HyperLogLog 是一种专门用于**基数估算**的概率数据结构，它回答的问题是："这个集合里有多少个不重复的元素？"

传统方案用 Set 存所有元素，元素越多内存占用越大。HyperLogLog 的神奇之处在于：**无论集合有多少元素，每个 HyperLogLog 键只占约 12KB 固定内存**，代价是有约 0.81% 的误差率。

### 2.2 核心命令

```redis
PFADD key element [element ...]      # 添加元素
PFCOUNT key [key ...]                 # 估算基数
PFMERGE destkey sourcekey [...]       # 合并多个 HyperLogLog
```

**示例：统计 UV**

```redis
# 每次用户访问，将用户 ID 加入当天的 HyperLogLog
PFADD uv:20260315 user_001 user_002 user_003

# 获取当天的不重复访客数
PFCOUNT uv:20260315

# 合并一周 UV
PFMERGE uv:week uv:20260309 uv:20260310 ... uv:20260315
PFCOUNT uv:week
```

### 2.3 适用场景

- **网站 UV 统计**：每天独立访客数，不需要精确到个位
- **搜索词去重统计**：统计每天有多少种不同的搜索词
- **大数据近似去重**：数亿条数据去重，允许千分之八的误差

**不适用场景**：需要精确计数、需要知道哪些元素存在（只能估算数量，不能查询元素）。

---

## 3. GEO 地理位置

### 3.1 底层原理

GEO 功能底层完全依托 **Sorted Set** 实现，没有独立的新数据结构。地理坐标通过 **Geohash 算法**编码成一个 52 位的整数作为 score，地点名称作为 member，存进 ZSet。

Geohash 的核心思想是把二维的经纬度空间递归地二分，将一个地理位置映射到一维的编码字符串，**空间上相邻的位置，Geohash 值也相近**，这样范围查询就能转化为 ZSet 的分数范围查询。

### 3.2 核心命令

```redis
GEOADD key longitude latitude member [...]    # 添加位置
GEODIST key member1 member2 [m|km|mi|ft]      # 计算距离
GEOPOS key member [...]                        # 获取坐标
GEOSEARCH key FROMMEMBER member BYRADIUS radius km ASC  # Redis 6.2+ 推荐
GEORADIUS key lon lat radius km WITHCOORD WITHDIST ASC  # 旧版命令
```

**示例：附近的商家**

```redis
# 添加商家位置
GEOADD stores 116.4053 39.9049 "麦当劳中关村店"
GEOADD stores 116.3950 39.9100 "星巴克中关村店"

# 查找某坐标 2km 内的商家，按距离升序
GEORADIUS stores 116.40 39.90 2 km WITHDIST ASC
```

### 3.3 注意事项

Geohash 编码的**精度是有限的**，边界附近的查询可能遗漏少量数据，所以实际使用时通常将搜索半径稍微扩大一点，再在应用层做精确过滤。另外，GEO 数据和普通 ZSet 数据不能存在同一个 key 下混用。

---

## 4. Bitmap 位图

### 4.1 本质

Bitmap 不是一种独立的数据结构，它本质上是对 **String 类型按位操作**的命令集合。Redis 的 String 最大 512MB，也就是说一个 Bitmap 最多可以表示 2^32（约 43 亿）个位。

### 4.2 核心命令

```redis
SETBIT key offset value      # 设置第 offset 位为 0 或 1
GETBIT key offset            # 获取第 offset 位的值
BITCOUNT key [start end]     # 统计值为 1 的位数（start/end 是字节范围）
BITOP AND/OR/XOR/NOT destkey key [key ...]  # 位运算
BITPOS key bit [start [end]] # 查找第一个 0 或 1 的位置
```

### 4.3 经典场景：签到统计

```java
// 每个用户每月一个 key，offset 为日期（0-30）
String key = "sign:" + userId + ":2026-03";
jedis.setbit(key, dayOfMonth - 1, true);  // 签到

long signCount = jedis.bitcount(key);     // 当月签到总天数
boolean signedToday = jedis.getbit(key, dayOfMonth - 1);  // 今天是否签到
```

一个用户一个月的签到数据，只需要 31 位 = 不到 4 字节。1000 万用户也才 38MB，内存效率极高。

### 4.4 其他应用场景

- **活跃用户分析**：`sign:all:20260315` 中每一位代表一个用户 ID，当天活跃则置 1，然后用 `BITOP AND` 计算连续多天都活跃的用户
- **在线状态**：用一个 Bitmap 表示所有用户的在线状态，`GETBIT` 判断单用户，`BITCOUNT` 统计在线人数

---

## 5. Redis 模块系统

### 5.1 是什么

Redis 4.0 引入的模块系统允许开发者用 **C 语言编写动态链接库**（`.so` 文件），在运行时加载到 Redis 进程，添加全新的数据类型和命令。

模块和 Redis 运行在**同一进程**里，通过内存直接交互，性能损耗极低。

**加载方式**：
```bash
# 配置文件加载
loadmodule /path/to/module.so [args]

# 命令动态加载（不重启）
MODULE LOAD /path/to/module.so
MODULE LIST        # 查看已加载模块
MODULE UNLOAD name # 卸载模块
```

**入口函数**：每个模块必须实现 `RedisModule_OnLoad(RedisModuleCtx *ctx, ...)` 函数，在这里注册新命令和新数据类型。

### 5.2 RedisJSON

RedisJSON 让 Redis 原生支持 JSON 文档的存储和操作，不再需要把 JSON 序列化成字符串存进 String。

**核心优势**：可以用 JSONPath 语法**直接操作文档内部的字段**，不需要取出整个 JSON、修改后再整体写回。

```redis
# 存储 JSON 文档
JSON.SET user:1000 $ '{"name":"Alice","age":30,"addr":{"city":"Beijing"}}'

# 读取特定字段（不需要取整个 JSON）
JSON.GET user:1000 $.name

# 修改嵌套字段
JSON.SET user:1000 $.addr.city '"Shanghai"'

# 数值自增
JSON.NUMINCRBY user:1000 $.age 1

# 数组追加
JSON.ARRAPPEND user:1000 $.hobbies '"reading"'
```

### 5.3 RediSearch

RediSearch 是构建在 Redis 上的全文搜索引擎，提供二级索引和复杂查询能力。

**适用场景**：需要对 Redis 中的 Hash/JSON 数据进行文本搜索、多条件过滤、聚合统计时，RediSearch 是首选。

```redis
# 创建索引
FT.CREATE productIdx ON HASH PREFIX 1 product:
  SCHEMA name TEXT WEIGHT 5.0 price NUMERIC tags TAG

# 全文搜索（价格 500-2000 的手机）
FT.SEARCH productIdx "手机" FILTER price 500 2000

# 聚合统计（按品牌分组，统计平均价格）
FT.AGGREGATE productIdx "*" GROUPBY 1 @brand REDUCE AVG 1 @price AS avg_price
```

### 5.4 RedisTimeSeries

RedisTimeSeries 是专为时间序列数据设计的模块，解决了用普通 ZSet 存时序数据时查询聚合不方便的问题。

**核心特性**：
- **自动降采样**：配置规则，自动将原始数据聚合为分钟、小时级别的数据
- **高效压缩**：专为时序数据优化的存储格式
- **多维度聚合**：avg/sum/min/max/count 等聚合函数

```redis
# 创建时间序列，数据保留 7 天
TS.CREATE cpu:usage:node1 RETENTION 604800000

# 添加数据点（时间戳:毫秒，值）
TS.ADD cpu:usage:node1 * 45.2   # * 表示使用当前时间

# 查询最近 1 小时，每 5 分钟平均值
TS.RANGE cpu:usage:node1 -3600000 + AGGREGATION avg 300000
```

---

## 6. CLIENT 命令

`CLIENT` 命令族用于管理和监控客户端连接，是日常运维和问题排查的重要工具。

```redis
CLIENT LIST               # 列出所有客户端连接的详细信息（IP、端口、命令、空闲时间等）
CLIENT GETNAME            # 获取当前连接的名称
CLIENT SETNAME myapp      # 给当前连接命名，便于 CLIENT LIST 时识别来源
CLIENT ID                 # 获取当前客户端 ID
CLIENT PAUSE 5000         # 暂停所有客户端命令执行 5 秒（主从切换时用）
CLIENT UNPAUSE            # 提前恢复
CLIENT KILL ID 42         # 踢掉 ID 为 42 的客户端连接
CLIENT NO-EVICT ON        # 标记当前连接不参与内存淘汰时的客户端剔除
```

**实际用途**：
- **排查连接泄漏**：通过 `CLIENT LIST` 查看连接数量和每个连接的空闲时间，找到一直不用却不关闭的连接
- **应用标识**：在应用启动时执行 `CLIENT SETNAME`，让 `CLIENT LIST` 输出中能看出是哪个服务
- **主从切换**：`CLIENT PAUSE` 可以暂停所有写入，等待主从同步完成后再恢复，减少切换时的数据不一致窗口

---

**相关面试题** → [[../../10_DevelopLanguage/004_NOSQL/01_RedisSubject/11、Redis 高级特性|11、Redis高级特性]]
