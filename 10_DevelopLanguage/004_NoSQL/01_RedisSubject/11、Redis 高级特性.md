###### 1. Redis 的 Lua 脚本有什么用?
Redis的Lua脚本功能允许你在服务器端执行**复杂的多步操作**，并保证这些操作的**原子性**。这对于需要执行多个Redis命令且要确保数据一致性的场景非常有用。
**核心价值与实现机制：**
1. **原子性保证**：Lua脚本在执行时，其他客户端命令无法插入执行。Redis使用**单线程事件循环**模型，通过内置的Lua解释器执行脚本，整个脚本作为一个任务单元被处理。在脚本执行期间，服务器不会处理其他请求，这与`MULTI`/`EXEC`事务类似，但功能更强大。
2. **减少网络开销**：将多个命令组合成一个脚本，只需一次网络往返，显著降低延迟，尤其在高网络延迟环境下效果明显。
3. **复杂逻辑封装**：可在脚本中编写条件判断、循环等复杂逻辑，实现如**分布式锁、限流、秒杀**等业务，这些逻辑在服务端执行，效率更高。
**源码角度浅析**：
Redis服务器启动时，在`server.c`的`initServer`函数中会调用`scriptingInit`初始化Lua环境。它创建Lua状态机（`lua_State`），加载基础库，并特别重要的是，向Lua环境中注入了`redis.call()`和`redis.pcall()`这两个关键函数，使得Lua脚本能够调用Redis命令。当执行`EVAL`命令时，`scripting.c`中的`evalGenericCommand`函数负责处理：它先检查脚本缓存，然后使用`lua_pcall`执行脚本。
**典型应用场景示例**：
- **原子递减库存**：确保检查库存和扣减操作的原子性。
- **带比较的分布式锁**：使用Lua脚本实现锁的获取和释放，避免误删其他客户端的锁。
- **批量数据预处理**：在服务端直接对获取的数据进行过滤、聚合等操作。
###### 2. Redis 的 EVAL 命令是什么?
`EVAL`是执行Lua脚本的核心命令。其基本语法为：
```redis
EVAL script numkeys key [key ...] arg [arg ...]
```
**参数详解与工作流程：**
- `script`：Lua脚本代码。
- `numkeys`：指定后续参数中有多少个是键（Key）。这些键名通过`KEYS`全局数组传递给Lua脚本（例如`KEYS[1]`, `KEYS[2]`）。
- `key [key ...]`：传递给脚本的键。
- `arg [arg ...]`：传递给脚本的附加参数。这些参数通过`ARGV`全局数组传递给Lua脚本（例如`ARGV[1]`, `ARGV[2]`）。
**执行过程**：
1. Redis服务器接收到`EVAL`命令。
2. 计算脚本的SHA1校验和，查询内部脚本缓存。若缓存中存在已编译的脚本，则直接使用；否则，会先编译脚本。
3. 执行编译后的Lua脚本。
4. 将Lua脚本的返回值转换为Redis协议格式并返回给客户端。
**示例：实现一个简单的限流器**
```lua
-- 使用 EVAL 实现一个简单的限流器（每分钟最多10次访问）
local key = KEYS[1] -- 限流的键，如用户ID或IP
local limit = tonumber(ARGV[1]) -- 限制次数
local current = tonumber(redis.call('GET', key) or "0")
if current + 1 > limit then
    return 0 -- 超过限制
else
    redis.call('INCR', key)
    redis.call('EXPIRE', key, 60) -- 设置过期时间
    return 1 -- 允许通过
end
```
执行命令：
```redis
EVAL "上述Lua脚本" 1 user_ip:127.0.0.1 10
```
**错误处理**：
在Lua脚本中，使用`redis.call()`执行Redis命令如果出错，会直接抛出Lua错误，导致脚本停止。而`redis.pcall()`则会以Lua表的形式捕获错误，允许脚本进行后续处理。在脚本中应合理选择，通常原子性操作中使用`redis.call()`，需要容错时考虑`redis.pcall()`。
###### 3. Redis 的 HyperLogLog 是什么?
HyperLogLog是一种用于**基数估算**的概率数据结构。它用于统计一个集合中**不重复元素的个数**，其最大优势是占用空间极小（每个HyperLogLog键只需约12KB内存），可估算数十亿个元素的基数，标准误差约为0.81%。
**核心特性：**
- **极低内存消耗**：无论集合包含多少元素，HyperLogLog都只使用固定大小的内存（约12KB）。
- **近似估算，非精确计数**：结果是概率性的，存在一定误差，但适用于大规模数据去重统计场景。
- **合并操作**：支持将多个HyperLogLog合并为一个，用于统计多个集合的并集基数。
**主要命令：**
- `PFADD key element [element ...]`：添加元素。
- `PFCOUNT key [key ...]`：计算基数。
- `PFMERGE destkey sourcekey [sourcekey ...]`：合并多个HyperLogLog。
**应用场景：**
- **统计网站UV**：统计每天访问网站的唯一用户数。
- **统计搜索关键词**：统计每天不同的搜索词数量。
- **大数据去重**：在数据量极大、可接受一定误差的场景下进行快速去重统计。
###### 4. Redis 的 Geo 地理位置功能如何使用?
Redis的Geo功能基于**Sorted Set**实现，提供了存储和查询地理位置信息的能力。
**核心命令：**
- `GEOADD key longitude latitude member`：添加地理位置信息。
- `GEODIST key member1 member2 [unit]`：计算两地距离。
- `GEORADIUS key longitude latitude radius unit`：查询指定半径内的地点。
- `GEORADIUSBYMEMBER key member radius unit`：以成员为中心查询半径内的地点。
**实现原理：**
Geo功能底层使用Sorted Set存储。地理位置经过**Geohash算法**编码后，作为一个分数（score）存入Sorted Set，成员（member）就是地点名称。这种设计使得Redis可以利用Sorted Set的特性高效实现距离计算和范围查询。
**应用示例：查找附近的商家**
```redis
# 添加商家地理位置
GEOADD stores 116.405285 39.904989 "store_A"
GEOADD stores 121.4737 31.2304 "store_B"

# 查找经度116.40，纬度39.90周围100公里内的商家
GEORADIUS stores 116.40 39.90 100 km WITHDIST
```
###### 5. Redis 的 Bitmap 是什么?有什么应用?
Bitmap（位图）通过操作字符串的二进制位来存储和查询信息，每个位表示一个状态（0或1），极其节省空间。
**核心特性：**
- **极致空间效率**：每个用户仅用1位存储信息，适合大规模布尔值存储。
- **高性能位运算**：支持AND、OR、XOR等位操作，实现复杂查询。
**主要命令：**
- `SETBIT key offset value`：设置指定偏移量的位值。
- `GETBIT key offset`：获取指定偏移量的位值。
- `BITCOUNT key [start end]`：统计值为1的位数。
- `BITOP operation destkey key [key ...]`：对多个Bitmap进行位运算。
**应用场景：**
- **用户签到统计**：每位代表一天签到状态，`BITCOUNT`可快速统计签到天数。
- **活跃用户分析**：按日创建Bitmap，每位代表一个用户是否活跃，使用`BITOP`可分析多日活跃用户。
- **布隆过滤器**：基于Bitmap实现，用于高效判断元素是否存在。
###### 6. Redis 的模块系统是什么?
Redis模块系统允许开发者使用C语言编写**动态链接库**来扩展Redis的功能，添加新的数据类型和命令。
**核心价值：**
- **功能扩展**：添加原生Redis不支持的数据结构和命令。
- **高性能**：模块与Redis服务器同一进程运行，无性能损耗。
- **灵活性**：可根据业务需求定制专用数据结构。
**工作原理：**
模块通过`RedisModule_OnLoad`函数初始化，向Redis注册新命令。新命令的处理函数可直接操作Redis内核数据结构，或使用模块API创建全新的数据类型。
**主流Redis模块：**
- **RedisJSON**：提供原生JSON支持，可直接操作JSON文档内的字段。
- **RediSearch**：全文搜索引擎，支持复杂查询、聚合和排序。
- **RedisTimeSeries**：时间序列数据库，支持降采样、聚合和压缩。
###### 7. RedisJSON 模块是什么?
RedisJSON是Redis Labs开发的模块，为Redis提供了**原生JSON文档支持**，允许直接存储、更新和查询JSON文档。
**核心特性：**
- **完整JSON支持**：支持所有JSON数据类型，验证文档有效性。
- **高效路径操作**：使用JSONPath语法选择和处理文档局部。
- **原子操作**：所有操作保证原子性。
**主要命令：**
- `JSON.SET key path value`：设置JSON文档。
- `JSON.GET key [path]`：获取JSON文档或指定路径值。
- `JSON.TYPE key path`：获取JSON值类型。
**应用示例：用户信息存储**
```redis
# 存储用户信息
JSON.SET user:1000 $ '{"name":"Alice","age":30,"address":{"city":"Beijing"}}'

# 获取用户年龄
JSON.GET user:1000 .age

# 修改用户城市
JSON.SET user:1000 $.address.city '"Shanghai"'
```
###### 8. RedisSearch 模块是什么?
RediSearch是构建在Redis之上的**全文搜索引擎**，提供二级索引、复杂查询和聚合功能。
**核心特性：**
- **全文检索**：支持词干提取、同义词、模糊匹配等。
- **复杂查询**：支持多字段联合查询、范围查询、地理空间查询。
- **聚合分析**：支持对结果分组、排序、统计。
**主要命令：**
- `FT.CREATE`：创建索引。
- `FT.SEARCH`：执行搜索。
- `FT.AGGREGATE`：执行聚合操作。
**应用示例：商品搜索**
```redis
# 创建商品索引
FT.CREATE itemIdx ON HASH PREFIX 1 item: SCHEMA
name TEXT WEIGHT 5.0
description TEXT
price NUMERIC
tags TAG

# 搜索商品
FT.SEARCH itemIdx "智能手机" WHERE price 1000 5000
```
###### 9. RedisTimeSeries 模块是什么?
RedisTimeSeries是专为**时间序列数据**设计的模块，优化了存储和查询效率。
**核心特性：**
- **高效压缩**：专为时间序列设计的压缩算法。
- **聚合查询**：支持按时间窗口进行多种聚合操作。
- **下采样**：支持不同精度存储数据。
**主要命令：**
- `TS.CREATE`：创建时间序列。
- `TS.ADD`：添加数据点。
- `TS.RANGE`：查询时间范围数据。
- `TS.MRANGE`：查询多个时间序列。
**应用示例：监控指标存储**
```redis
# 创建时间序列
TS.CREATE temperature RETENTION 604800000 # 保留7天

# 添加数据点
TS.ADD temperature 1640995200000 25.5

# 查询最近1小时数据，每10分钟平均值
TS.RANGE temperature 1640991600000 1640995200000 AGGREGATION avg 600000
```
###### 10. Redis 的 CLIENT 命令有什么用?
`CLIENT`命令用于**管理和监控Redis的客户端连接**，是运维和调试的重要工具。
**主要子命令及用途：**
- `CLIENT LIST`：查看所有客户端连接详情（IP、端口、空闲时间、当前命令等）。
- `CLIENT GETNAME`/`SETNAME`：获取/设置当前连接名称，便于识别。
- `CLIENT PAUSE`：暂停所有客户端请求指定时间（用于主从切换等场景）。
- `CLIENT KILL`：断开指定客户端连接。
- `CLIENT ID`：获取当前客户端ID。
**应用场景：**
- **连接监控**：通过`CLIENT LIST`查看客户端状态，识别异常连接。
- **问题排查**：通过客户端名称标识不同应用连接，快速定位问题源。
- **安全控制**：使用`CLIENT KILL`断开恶意或异常连接。
- **运维管理**：在主从切换时使用`CLIENT PAUSE`确保数据一致性。