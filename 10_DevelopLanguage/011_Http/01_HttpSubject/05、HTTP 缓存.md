# 五、HTTP 缓存

###### 1. 什么是 HTTP 缓存？HTTP 缓存的优点是什么？

HTTP 缓存就是把服务器的响应结果**存储在客户端或代理节点**，下次相同请求直接用缓存数据，不用再向服务器请求。

**优点**：
- **减少网络传输**：缓存命中时不需要传输响应体，节省带宽
- **降低服务器压力**：大量重复请求被缓存挡住，服务器处理量大幅减少
- **提升响应速度**：从本地或就近节点（CDN）读取，比远程请求快得多
- **提升用户体验**：页面加载更快，弱网环境也能正常访问已缓存的资源

HTTP 缓存分两层：
- **浏览器缓存**：存在用户本地，最近、最快
- **代理缓存/CDN**：存在中间节点，多用户共享，覆盖范围大

缓存分两大类型：
- **强缓存**：直接用缓存，不问服务器
- **协商缓存**：先问服务器"我的缓存还有效吗"，服务器说有效就用，无效就给新的

---

###### 2. 什么是强缓存和协商缓存？

**强缓存**：浏览器直接用本地缓存，**不发请求给服务器**。判断依据是缓存是否还在有效期内，返回的 HTTP 状态码是 **200（from cache）**。

控制字段：
- `Expires`：绝对过期时间（HTTP/1.0）
- `Cache-Control: max-age=3600`：相对过期时间，优先级高于 Expires（HTTP/1.1）

```
首次请求：
  → GET /logo.png HTTP/1.1
  ← 200 OK, Cache-Control: max-age=86400

1小时后再次请求（在24小时内）：
  → 浏览器直接用本地缓存，不发网络请求
  ← 200 (from disk cache)
```

**协商缓存**：浏览器**发请求**给服务器，携带本地缓存的标识，让服务器判断缓存是否还有效。有效返回 **304 Not Modified**（无响应体），无效返回 **200** 带新数据。

控制字段：
- `Last-Modified` / `If-Modified-Since`：基于时间
- `ETag` / `If-None-Match`：基于内容标识（优先级更高）

```
首次请求：
  ← 200 OK, ETag: "abc123", Last-Modified: Sun, 15 Mar 2026 06:00:00 GMT

下次请求：
  → GET /logo.png, If-None-Match: "abc123"
  ← 304 Not Modified（缓存未变，用本地缓存）
  或
  ← 200 OK, ETag: "xyz789"（内容有变，下发新内容）
```

两者的工作顺序：**先看强缓存是否有效 → 失效后走协商缓存 → 协商缓存失效才拿新资源**。

---

###### 3. Cache-Control 有哪些常用的指令？

`Cache-Control` 是 HTTP/1.1 最核心的缓存控制字段，优先级高于 `Expires`。

**可缓存性**：
- `public`：响应可以被任何节点缓存（浏览器、CDN、代理）
- `private`：只能被浏览器缓存，代理和 CDN 不缓存（默认值）
- `no-cache`：不是"不缓存"！而是"先问服务器是否有效，再决定用不用缓存"（走协商缓存）
- `no-store`：完全不缓存，每次都从服务器拿新的（涉及敏感数据时使用）

**过期时间**：
- `max-age=N`：资源在 N 秒内有效（相对时间）
- `s-maxage=N`：专门针对共享缓存（CDN/代理）的有效时间，覆盖 max-age
- `max-stale=N`：客户端愿意接受已过期但不超过 N 秒的缓存

**重新验证**：
- `must-revalidate`：缓存过期后必须向服务器验证，不允许使用过期缓存
- `proxy-revalidate`：要求代理缓存过期后重新验证

**常用组合**：
```
Cache-Control: no-store                          # 敏感数据，完全不缓存
Cache-Control: no-cache                          # 每次都验证，适合动态内容
Cache-Control: max-age=31536000, immutable       # 静态资源（带 hash），一年不变
Cache-Control: private, max-age=3600             # 用户个人数据，缓存1小时
Cache-Control: public, max-age=86400, s-maxage=604800  # CDN缓存7天，浏览器1天
```

常见误区：`no-cache ≠ no-store`，no-cache 还是会缓存的，只是每次用前要验证。

---

###### 4. Expires 和 Cache-Control 有什么区别？

两者都用来控制强缓存的有效期，但有本质差异：

**Expires（HTTP/1.0）**：
- 值是**绝对时间**（GMT 格式）：`Expires: Sun, 22 Mar 2026 08:00:00 GMT`
- 问题：**依赖客户端和服务器的时钟同步**，如果客户端时间不准，缓存行为会出错
- 现在基本作为兼容方案，已经淡出主流

**Cache-Control（HTTP/1.1）**：
- 值是**相对时间**（秒数）：`Cache-Control: max-age=3600`
- 从响应生成时刻计算，不依赖时钟同步，更可靠
- 功能更丰富（no-cache/no-store/public/private 等）

**优先级**：当两者同时存在时，**Cache-Control 优先级高于 Expires**，Expires 被忽略。

现代 Web 开发应该优先使用 `Cache-Control`，`Expires` 只作为给老旧 HTTP/1.0 客户端的兼容字段。

---

###### 5. 什么是 ETag？ETag 的作用是什么？

ETag（Entity Tag，实体标签）是服务器给资源生成的**唯一标识符**，类似资源的"指纹"。

当资源内容变化时，ETag 也会变化；内容不变，ETag 就不变。

**工作流程**：
1. 服务器首次响应时，在 `ETag` 响应头里附上资源标识
2. 客户端缓存资源和 ETag 值
3. 下次请求时，把 ETag 放在 `If-None-Match` 请求头发给服务器
4. 服务器对比 ETag：一致 → 返回 304，不一致 → 返回 200 + 新资源

```
← ETag: "33a64df551425fcc55e4d42a148795d9f25f89d"
→ If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d"
← 304 Not Modified
```

**ETag 的两种类型**：
- **强 ETag**：`"abc123"` —— 字节级精确匹配，内容任何变化都会改变 ETag
- **弱 ETag**：`W/"abc123"` —— 语义级匹配，允许内容有细微差异（如时间戳变化）也认为等价

**ETag 的生成方式**（常见）：
- 文件内容的 MD5/SHA-1 哈希
- 文件大小 + 最后修改时间的组合
- 版本号

---

###### 6. Last-Modified 和 ETag 有什么区别？

两者都用于协商缓存，都是"资源是否变化"的判断依据。

**Last-Modified / If-Modified-Since（基于时间）**：
- `Last-Modified`：服务器返回资源最后修改时间
- `If-Modified-Since`：客户端下次请求时携带，服务器对比时间决定 304 还是 200
- **精度问题**：时间精度只到秒，1秒内多次修改无法区分
- **误判问题**：文件被修改了但内容没变（比如重新保存），时间变了但内容没变，会不必要地返回新资源
- **分布式问题**：多台服务器文件的修改时间可能不一致

**ETag / If-None-Match（基于内容标识）**：
- ETag 是基于内容计算的哈希值，内容没变 ETag 就不变，完全准确
- 能检测到秒级内的多次修改
- 精度更高，但计算 ETag 需要额外开销

**优先级**：当两者同时存在时，**ETag / If-None-Match 优先级高于 Last-Modified**，服务器优先用 ETag 判断。

实际建议：对精度要求高的资源（如 API 数据、频繁变更的文件）用 ETag；对静态资源（图片、大文件）可以用 Last-Modified，开销小一点。

---

###### 7. 什么是 304 状态码？什么情况下会返回 304？

**304 Not Modified**：表示客户端的缓存还是有效的，响应体为空，客户端直接用本地缓存数据。

304 是协商缓存命中的标志，整个流程是：

1. 浏览器发条件请求（携带 `If-None-Match` 或 `If-Modified-Since`）
2. 服务器对比：
   - 如果资源没变 → 返回 304，不传响应体（省流量）
   - 如果资源变了 → 返回 200，带上新资源

返回 304 的具体条件：
- 请求头带 `If-None-Match: "etag值"`，且与服务器当前 ETag 匹配
- 请求头带 `If-Modified-Since: 时间`，且资源在该时间后未被修改

304 响应的特点：
- **没有响应体**（body），省去了数据传输
- 仍然需要一个 HTTP 请求/响应的往返（这点不如强缓存，强缓存完全不发请求）
- 响应头里会更新缓存相关的头部（如新的 `Cache-Control`、`Date`）

---

###### 8. 如何设置不缓存？

不同场景有不同的"不缓存"方案：

**完全禁止缓存**（最严格，适合敏感数据）：
```
Cache-Control: no-store
```

**每次请求都重新验证**（推荐用于动态内容）：
```
Cache-Control: no-cache
# 或者等价的更完整写法（兼容老浏览器）：
Cache-Control: no-cache, no-store, must-revalidate
Pragma: no-cache            # HTTP/1.0 兼容
Expires: 0                  # 兼容字段，设为过去时间
```

**Spring Boot 接口默认配置**：

Spring Boot 的 Controller 接口默认不带缓存头，实际上相当于 no-cache 行为（浏览器不会缓存 API 响应）。如果要给静态资源加缓存，可以在 `WebMvcConfigurer` 里配置：

```java
// 强制 API 接口不缓存
response.setHeader("Cache-Control", "no-cache, no-store, must-revalidate");
response.setHeader("Pragma", "no-cache");
response.setDateHeader("Expires", 0);
```

**Nginx 配置**：
```nginx
location /api/ {
    add_header Cache-Control "no-cache, no-store, must-revalidate";
}
location /static/ {
    add_header Cache-Control "public, max-age=31536000, immutable";
}
```

---

###### 9. 什么是启发式缓存？

当服务器响应**没有明确指定缓存有效期**（既没有 `Cache-Control`，也没有 `Expires`），浏览器不是直接放弃缓存，而是会**根据一定规则估算一个缓存时间**——这就是启发式缓存。

**计算公式**（RFC 7234 规范）：

```
缓存时长 = (Date - Last-Modified) × 10%
```

也就是：从"上次修改"到"本次响应时间"的时间差乘以 10%。

举例：
- 响应时间：2026-03-15
- Last-Modified：2026-03-08（7 天前）
- 启发式缓存时间 = 7天 × 10% = 0.7 天 ≈ 17 小时

如果没有 `Last-Modified`，不同浏览器处理方式不同，Chrome 会使用 0 或一个很短的时间。

**实际建议**：不要依赖启发式缓存，服务端应该**明确设置 Cache-Control**，行为明确可控。对动态接口设 `no-cache`，对静态资源设具体的 `max-age`。

---

###### 10. 【追问】前端静态资源缓存策略如何设计才最优？

这是工程实践中很高频的问题，关键在于**"长缓存 + 内容 Hash"**的组合策略。

**核心矛盾**：想让浏览器长时间缓存（提升性能），但又要能在版本更新时立刻让缓存失效（保证用户看到最新版）。

**最优方案**：

```
index.html    → Cache-Control: no-cache（每次验证，保证拿到最新的入口文件）
app.abc123.js → Cache-Control: max-age=31536000, immutable（文件名带 hash，内容变才换名，一年不过期）
```

**原理**：
- `index.html` 是入口，不缓存（或短缓存），每次都能拿到最新的 JS/CSS 引用
- `app.abc123.js` 文件名包含内容 hash，内容不变 hash 不变（可以永久缓存），内容变了 hash 就变了（URL 变了，强制拿新文件）
- `immutable` 指令告诉浏览器：这个文件永远不会变，连 max-age 到期后的重新验证都不用做

**Webpack/Vite 等构建工具**默认就支持 contenthash 文件名，配合 CDN 的 `Cache-Control: max-age=31536000, immutable` 可以最大化缓存效果。
