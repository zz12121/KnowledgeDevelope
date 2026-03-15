###### 1. Redis 的 Lua 脚本有什么用?
Redis 的 Lua 脚本功能允许你在服务器端执行**复杂的多步操作**，并保证这些操作的**原子性**。这对于需要执行多个 Redis 命令且要确保数据一致性的场景非常有用。

**核心价值**

首先是**原子性保证**。Lua 脚本在执行时，其他客户端命令无法插入执行。Redis 使用单线程事件循环模型，通过内置的 Lua 解释器执行脚本，整个脚本作为一个任务单元被处理——这与 `MULTI/EXEC` 事务类似，但功能更强大，可以在脚本里写条件判断和循环。

其次是**减少网络开销**。将多个命令组合成一个脚本，只需一次网络往返，在高延迟网络环境下效果明显。

**源码角度浅析**

Redis 服务器启动时，`initServer` 函数会调用 `scriptingInit` 初始化 Lua 环境：创建 Lua 状态机（`lua_State`），加载基础库，并向 Lua 环境注入 `redis.call()` 和 `redis.pcall()` 两个关键函数，使得 Lua 脚本能够调用 Redis 命令。执行 `EVAL` 命令时，`scripting.c` 中的 `evalGenericCommand` 函数负责处理：先检查脚本缓存（按 SHA1 校验和），再用 `lua_pcall` 执行脚本。

**典型应用场景**：原子递减库存、带比较的分布式锁释放、批量数据预处理等，这些场景都需要"检查 + 操作"的原子复合语义，用 Lua 脚本是最优解。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/01、Lua脚本与高级特性#1-lua-脚本原理|知识库：Redis Lua 脚本原理]]

---

###### 2. Redis 的 EVAL 命令是什么?
`EVAL` 是执行 Lua 脚本的核心命令，语法如下：

```redis
EVAL script numkeys key [key ...] arg [arg ...]
```

**参数说明**：`script` 是 Lua 代码；`numkeys` 指定有多少个后续参数是 Key（这些 Key 通过 `KEYS[1]`、`KEYS[2]` 传入脚本）；剩余参数作为附加值，通过 `ARGV[1]`、`ARGV[2]` 传入脚本。

**执行流程**：Redis 收到 `EVAL` 后，计算脚本的 SHA1 校验和，先查询内部脚本缓存；缓存命中则直接执行，否则先编译再执行；最后把 Lua 返回值转换为 Redis 协议格式返回给客户端。

**示例：用 EVAL 实现一个简单的限流器**

```lua
-- 每分钟最多 10 次访问
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local current = tonumber(redis.call('GET', key) or "0")
if current + 1 > limit then
    return 0  -- 超过限制
else
    redis.call('INCR', key)
    redis.call('EXPIRE', key, 60)
    return 1  -- 允许通过
end
```

```redis
EVAL "上述 Lua 脚本" 1 user_ip:127.0.0.1 10
```

**`redis.call` vs `redis.pcall` 的区别**：`redis.call()` 遇到错误会直接抛出 Lua 异常，导致脚本中断，同时向客户端返回错误；`redis.pcall()` 会以 Lua 表的形式捕获错误，让脚本可以自行处理，适合需要容错的场景。原子性操作中通常用 `redis.call()`——出错就应该中断，不应该继续执行后续步骤。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/01、Lua脚本与高级特性#2-eval-命令详解|知识库：EVAL 命令与脚本缓存]]

---

###### 3. Redis 的 HyperLogLog 是什么?
HyperLogLog 是一种用于**基数估算**的概率数据结构，用来统计集合中**不重复元素的个数**。它最大的优势是内存极低——无论集合有多少元素，每个 HyperLogLog 键只占约 **12KB** 固定内存，可以估算数十亿量级的基数，标准误差约为 **0.81%**。

**核心特性**

这是一种空间换精度的典型设计：你用 12KB 就能处理海量数据的去重统计，代价是结果是概率性的，不是精确计数。如果业务对误差有严格要求，就不适合用 HyperLogLog。

另外它支持合并操作，多个 HyperLogLog 可以合并成一个，用于计算多个集合的并集基数，比如统计多天的 UV 合并后的总 UV。

**主要命令**

- `PFADD key element [element ...]`：添加元素（PF 是 Philippe Flajolet 的缩写，算法发明者）
- `PFCOUNT key [key ...]`：计算基数（多个 key 时返回并集基数）
- `PFMERGE destkey sourcekey [sourcekey ...]`：合并多个 HyperLogLog

**典型场景**：统计网站 UV（每天访问的唯一用户数）、统计搜索关键词数量。精确 UV 统计用 Set 会占用大量内存，HyperLogLog 以 0.81% 的误差换来了极低的内存消耗，对大多数业务场景已经足够。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/01、Lua脚本与高级特性#3-hyperloglog|知识库：HyperLogLog 基数估算]]

---

###### 4. Redis 的 Geo 地理位置功能如何使用?
Redis 的 Geo 功能基于 **Sorted Set** 实现，提供了存储和查询地理位置信息的能力。

**底层原理**

Geo 功能使用 **Geohash 算法**将二维经纬度坐标编码成一个一维的 52 位整数，作为 Sorted Set 的 score 存储；地点名称作为 member。Geohash 的编码方式保留了空间关系——越接近的位置，其 Geohash 数值越接近，因此 Sorted Set 的范围查询天然支持地理空间查询。

**核心命令**

- `GEOADD key longitude latitude member`：添加地理位置
- `GEODIST key member1 member2 [unit]`：计算两点距离（支持 m/km/mi/ft）
- `GEORADIUS key longitude latitude radius unit`：查询指定坐标半径内的地点
- `GEORADIUSBYMEMBER key member radius unit`：以某个 member 为中心查询

**应用示例：查找附近的商家**

```redis
# 添加商家位置
GEOADD stores 116.405285 39.904989 "store_A"
GEOADD stores 121.4737 31.2304 "store_B"

# 查找经纬度 (116.40, 39.90) 周围 100 公里内的商家，并返回距离
GEORADIUS stores 116.40 39.90 100 km WITHDIST ASC
```

注意 Redis 7.0 之后推荐使用 `GEOSEARCH` 和 `GEOSEARCHSTORE` 替代 `GEORADIUS`，功能更强、支持矩形范围查询。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/01、Lua脚本与高级特性#4-geo-地理位置|知识库：Redis GEO 地理位置功能]]

---

###### 5. Redis 的 Bitmap 是什么?有什么应用?
Bitmap（位图）本质上是操作字符串二进制位的一套命令，每个 bit 表示一个状态（0 或 1），空间效率极高。

**核心特性**

每个用户的签到状态只用 1 bit 存储，10 亿用户的一天签到数据只需约 125MB。配合位运算命令，可以高效实现多维度的用户统计。

**主要命令**

- `SETBIT key offset value`：设置指定偏移量的位值（0 或 1）
- `GETBIT key offset`：获取指定偏移量的位值
- `BITCOUNT key [start end]`：统计值为 1 的位数（可指定字节范围）
- `BITOP operation destkey key [key ...]`：对多个 Bitmap 进行 AND/OR/XOR/NOT 位运算

**应用场景**

**用户签到统计**：以 `sign:userId:202601` 为 key，日期减 1 作为 offset，`SETBIT` 打卡，`BITCOUNT` 一次性统计全月签到天数，O(1) 时间复杂度。

**活跃用户分析**：按日创建 Bitmap，每位代表一个用户 ID，`BITOP AND` 求多日都活跃的用户，`BITOP OR` 求任意日活跃的用户。

**布隆过滤器**：基于 Bitmap 实现的概率型数据结构，用于快速判断元素是否存在，常见于缓存穿透防护。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/01、Lua脚本与高级特性#5-bitmap-位图|知识库：Redis Bitmap 位图]]

---

###### 6. Redis 的模块系统是什么?
Redis 模块系统允许开发者用 C 语言编写**动态链接库**来扩展 Redis 的功能，添加新的数据类型和命令。

**工作原理**

模块通过 `RedisModule_OnLoad` 函数初始化，向 Redis 注册新命令。新命令的处理函数可以直接操作 Redis 内核数据结构，也可以通过模块 API 创建全新的数据类型。由于模块与 Redis 服务器同进程运行，调用无需跨进程通信，性能几乎没有损耗。

**主流 Redis 模块**

- **RedisJSON**：为 Redis 提供原生 JSON 文档支持，可直接操作 JSON 文档内的字段，使用 JSONPath 语法
- **RediSearch**：全文搜索引擎，支持词干提取、模糊匹配、多字段联合查询、地理空间查询和聚合分析
- **RedisTimeSeries**：时间序列数据库，专为监控指标、IoT 数据设计，支持降采样、聚合和高效压缩

这些模块解决了原生 Redis 在特定领域的能力不足，让 Redis 从单纯的 KV 存储扩展成了多用途的数据平台。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/01、Lua脚本与高级特性#6-模块系统|知识库：Redis 模块系统]]

---

###### 7. RedisJSON 模块是什么?
RedisJSON 是 Redis Labs 开发的模块，为 Redis 提供了**原生 JSON 文档支持**，允许直接存储、更新和查询 JSON 文档的局部字段，而不需要把整个 JSON 取出来反序列化后修改再写回去。

**核心优势**

使用 **JSONPath** 语法可以精确操作文档中的某个字段，避免了传统方案"读取整个 JSON → 修改 → 写回"的低效流程。所有操作保证原子性。

**主要命令**

```redis
# 存储用户信息
JSON.SET user:1000 $ '{"name":"Alice","age":30,"address":{"city":"Beijing"}}'

# 只获取用户年龄（不需要取整个文档）
JSON.GET user:1000 .age

# 只修改城市字段（不影响其他字段）
JSON.SET user:1000 $.address.city '"Shanghai"'

# 获取某个路径的值类型
JSON.TYPE user:1000 $.address
```

**适用场景**：用户画像、商品信息、配置数据等需要频繁读写 JSON 局部字段的场景。相比把 JSON 序列化成字符串存 String 类型，RedisJSON 在字段级操作上性能更好，也更灵活。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/01、Lua脚本与高级特性#7-redisjson-模块|知识库：RedisJSON 模块]]

---

###### 8. RedisSearch 模块是什么?
RediSearch 是构建在 Redis 之上的**全文搜索引擎**，提供二级索引、复杂查询和聚合功能。

**核心特性**

支持**全文检索**（词干提取、同义词、模糊匹配、中文分词）、**多字段联合查询**（文本 + 数值 + 标签 + 地理位置组合查询）、**聚合分析**（分组、排序、统计）。索引构建在内存中，查询速度远超 MySQL 的 LIKE 查询。

**主要命令**

```redis
# 创建商品索引（索引以 item: 为前缀的所有 Hash）
FT.CREATE itemIdx ON HASH PREFIX 1 item:
  SCHEMA name TEXT WEIGHT 5.0
         description TEXT
         price NUMERIC
         tags TAG

# 搜索"智能手机"，价格在 1000~5000 之间
FT.SEARCH itemIdx "@name:智能手机 @price:[1000 5000]"

# 聚合统计各品牌商品数量
FT.AGGREGATE itemIdx "*" GROUPBY 1 @brand REDUCE COUNT 0 AS count
```

**适用场景**：电商商品搜索、文章检索、日志查询等需要复杂查询能力且追求低延迟的场景。优势是数据和索引都在 Redis 里，没有数据同步延迟问题。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/01、Lua脚本与高级特性#8-redisearch-模块|知识库：RediSearch 全文搜索]]

---

###### 9. RedisTimeSeries 模块是什么?
RedisTimeSeries 是专为**时间序列数据**设计的 Redis 模块，优化了时序数据的存储和查询效率。

**核心特性**

使用专为时间序列设计的压缩算法，支持按时间窗口进行多种聚合操作（avg/sum/min/max/count），支持**降采样**（Downsampling）——把高精度原始数据按规则聚合成低精度数据，用更少的空间保留趋势信息。

**主要命令**

```redis
# 创建时间序列，保留 7 天（604800000 毫秒）
TS.CREATE temperature RETENTION 604800000

# 添加数据点（时间戳 + 值）
TS.ADD temperature 1640995200000 25.5

# 查询 1 小时内数据，每 10 分钟计算平均值（降采样查询）
TS.RANGE temperature 1640991600000 1640995200000 AGGREGATION avg 600000

# 查询多个时间序列（如多台机器的 CPU）
TS.MRANGE - + FILTER type=cpu
```

**适用场景**：系统监控指标（CPU/内存/QPS 等）、IoT 传感器数据、金融行情数据。相比把时序数据存到普通 Redis ZSet 或 Hash，RedisTimeSeries 在存储效率和聚合查询性能上有明显优势。

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/01、Lua脚本与高级特性#9-redistimeseries-模块|知识库：RedisTimeSeries 时序数据]]

---

###### 10. Redis 的 CLIENT 命令有什么用?
`CLIENT` 命令用于**管理和监控 Redis 的客户端连接**，是运维和调试的重要工具。

**主要子命令**

- `CLIENT LIST`：查看所有客户端连接详情，包括 IP、端口、空闲时间、当前正在执行的命令、内存占用等。排查"谁在跑慢查询"时非常有用
- `CLIENT SETNAME name` / `CLIENT GETNAME`：给当前连接命名。建议在应用启动时调用，这样 `CLIENT LIST` 里能看到连接来自哪个服务
- `CLIENT ID`：获取当前连接的唯一 ID，在 Lua 脚本调试或需要标识连接身份时使用
- `CLIENT PAUSE millis`：暂停所有客户端请求指定时间，常用于主从切换前确保数据一致性
- `CLIENT KILL`：断开指定客户端连接，可按 ID、IP:port 或连接名过滤

**典型运维场景**

- 线上出现慢查询时，用 `CLIENT LIST` 结合 `cmd` 字段快速找到问题连接
- 应用上线时通过 `CLIENT SETNAME` 标识连接来源，方便多服务混部时区分
- 主从切换时用 `CLIENT PAUSE` 暂停写入，等待主从数据同步完成后再切换，避免数据丢失

> 📖 [[../../../23_RedisKnowledge/10_高级特性与运维/01、Lua脚本与高级特性#10-client-命令|知识库：Redis CLIENT 连接管理命令]]
