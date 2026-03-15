# 十二、HTTP 性能优化

###### 1. 如何优化 HTTP 请求性能？

HTTP 性能优化可以从**减少请求数量**、**减少传输体积**、**加快响应速度**三个维度入手：

**减少请求次数**：
- 资源合并（CSS Sprite、代码打包）
- 浏览器缓存（强缓存让资源不用再请求）
- HTTP/2 多路复用（降低多连接开销）
- 懒加载（只请求当前需要的资源）

**减少传输体积**：
- 开启 GZIP/Brotli 压缩（文本资源压缩 60-80%）
- 压缩图片（WebP 比 JPEG/PNG 小 25-35%）
- 代码 Tree Shaking、死代码删除
- HPACK 头部压缩（HTTP/2 自带）

**加快响应速度**：
- CDN 就近加速（减少物理距离延迟）
- DNS 预解析（`<link rel="dns-prefetch">`）
- 连接预建立（`<link rel="preconnect">`）
- 服务端缓存（减少后端计算时间）
- 升级 HTTP/2（多路复用 + 头部压缩）

**减少阻塞**：
- CSS 放 `<head>`（先加载，避免重排）
- JS 放 `</body>` 前或用 `async`/`defer` 属性（避免阻塞渲染）
- 关键资源 `<link rel="preload">`（提前加载，不阻塞渲染）

---

###### 2. 什么是资源合并？有什么优缺点？

资源合并是把多个小文件合并成一个大文件，减少 HTTP 请求数量。

**HTTP/1.1 时代的常见做法**：
- CSS Sprite：把多张小图标合成一张大图，用 `background-position` 切割
- JS Bundle：webpack 把多个 JS 模块打包成一个（或少数几个）文件
- CSS 合并：多个 CSS 文件合并成一个

**优点**：
- 减少请求数（HTTP/1.1 下每个请求都有连接开销）
- 减少 HTTP Header 开销（越多请求，重复的 Header 越多）

**缺点**：
- 牵一发而动全身：一个小文件修改，整个 bundle 缓存失效
- 不需要的代码也要下载（用户只看 A 页面，却要加载 B 页面的代码）
- 延迟加载变难：大文件必须完整下载后才能执行
- HTTP/2 多路复用后，合并的必要性大幅降低

**HTTP/2 下的调整**：HTTP/2 下多个小文件并发请求成本很低，过度合并反而影响缓存效率。现代构建工具（Vite/webpack）会做**代码分割（Code Splitting）**，按路由/功能拆分 chunk，兼顾性能和缓存。

---

###### 3. 什么是域名分片？

域名分片（Domain Sharding）是 HTTP/1.1 时代提升并发数的技巧：把资源分散到多个子域名，绕过浏览器"同一域名最多 6 个并发连接"的限制。

```
// 把静态资源分散到多个 CDN 子域名
img1.example.com/image1.jpg    ← 6 个并发
img2.example.com/image2.jpg    ← 再来 6 个并发
img3.example.com/image3.jpg    ← 再来 6 个并发
```

**HTTP/1.1 时代**：这是个有效的优化手段，淘宝、Google 都在用。

**HTTP/2 时代**：**域名分片已经有害无益**——
- HTTP/2 单条连接就能多路复用，不需要多个连接
- 多个域名会导致多次 TLS 握手，额外开销
- 域名分片会破坏 HTTP/2 的头部压缩效果（每个域名的动态表独立）

**结论**：用 HTTP/1.1 可以考虑域名分片；用 HTTP/2 应该反向操作，把资源收归同一域名（Domain Consolidation）。

---

###### 4. 什么是 CDN？CDN 的作用是什么？

**CDN（Content Delivery Network，内容分发网络）** 是由分布在全球各地的缓存服务器组成的网络。当用户请求资源时，由**距用户最近的节点**响应，而不是每次都回源到中心服务器。

**核心作用**：

**就近访问，减少延迟**：上海的用户请求资源，直接从上海 CDN 节点获取，不用跨越半个中国访问北京的源站。物理距离从 1200km 降到 20km，延迟从 30ms 降到 2ms。

**减轻源站压力**：静态资源（图片/JS/CSS）全部由 CDN 节点缓存响应，90%+ 的请求不到源站，源站只处理动态请求和缓存 MISS。

**提升可用性**：CDN 节点遍布全球，即使源站故障，静态资源仍可正常访问；CDN 还有 DDoS 防护能力。

**全球加速**：对海外用户效果显著，不需要在各国部署服务器。

**工作原理**：
1. 用户访问 `cdn.example.com/image.jpg`
2. DNS 把域名解析到离用户最近的 CDN 节点 IP
3. CDN 节点查本地缓存，命中 → 直接返回；未命中 → 回源站拉取，缓存后返回
4. 响应带 `Cache-Control` 头，控制 CDN 缓存时长

---

###### 5. 什么是 HTTP 压缩？常见的压缩算法有哪些？

HTTP 压缩是服务器在返回响应体前，对内容进行压缩，客户端收到后解压，减少传输数据量。

**工作机制**：
- 客户端声明支持的压缩算法：`Accept-Encoding: gzip, br, zstd`
- 服务器选择算法压缩后响应，并在 `Content-Encoding` 头里告知：`Content-Encoding: gzip`
- 客户端根据 `Content-Encoding` 解压

**常见压缩算法对比**：

| 算法 | 压缩率 | 速度 | 兼容性 | 推荐场景 |
|------|--------|------|--------|----------|
| gzip | 好 | 中 | 几乎全部浏览器 | 通用首选 |
| deflate | 类似 gzip | 中 | 广泛 | 不推荐（实现有歧义） |
| Brotli（br） | 比 gzip 高 15-20% | 稍慢 | 现代浏览器 | HTTPS 下推荐 |
| zstd | 与 gzip 相近 | 非常快 | 新版 Chrome/Firefox | 实时性要求高的场景 |

**适合压缩的类型**：text/html、application/json、application/javascript、text/css、text/plain、application/xml

**不适合压缩的类型**：image/jpeg、image/png、image/gif、video/mp4（已经是压缩格式，再压效果极差甚至增大）

**压缩效果参考**：一个 500KB 的 JSON 响应，gzip 后约 100-120KB，Brotli 后约 80-90KB。

---

###### 6. 什么是懒加载和预加载？

两者是相反方向的优化策略：

**懒加载（Lazy Loading）**：延迟加载非关键资源，等到**真正需要时**才加载。

- **图片懒加载**：用户滚动到图片位置前不加载，用 `loading="lazy"` 属性或 Intersection Observer API 实现
- **路由懒加载**：用户访问某个路由时才加载对应的 JS chunk（Vue/React 的动态 import）
- **按需加载**：组件库只加载用到的组件，不打包整个库

```html
<!-- HTML 原生懒加载 -->
<img src="photo.jpg" loading="lazy" alt="...">
```

```javascript
// Vue 路由懒加载
const UserPage = () => import('./pages/UserPage.vue')
```

**预加载（Preloading）**：提前加载接下来**很可能需要**的资源，减少用户实际需要时的等待。

- `<link rel="preload">`：高优先级预加载当前页面即将用到的资源（字体、关键 CSS/JS）
- `<link rel="prefetch">`：低优先级预加载下一页面可能用到的资源
- `<link rel="preconnect">`：提前建立与第三方域名的连接（DNS + TCP + TLS）
- `<link rel="dns-prefetch">`：只提前解析 DNS

```html
<!-- 预加载关键字体，避免 FOUT（闪烁问题） -->
<link rel="preload" href="/fonts/font.woff2" as="font" crossorigin>

<!-- 预加载下一步可能访问的页面 -->
<link rel="prefetch" href="/next-page.html">
```

---

###### 7. 如何减少 HTTP 请求次数？

减少 HTTP 请求是前端性能优化最直接的手段，具体措施：

**资源合并和打包**：
- 使用 webpack/Vite 打包 JS/CSS
- CSS Sprite 合并小图标（HTTP/1.1 时代更有效）
- 使用图标字体（Iconfont）或 SVG Sprite 替代多个 png

**缓存利用**：
- 强缓存（带 hash 的静态资源）让重复访问无请求
- Service Worker 缓存离线资源

**内联小资源**：
- 小图片转为 Base64 内联在 CSS/HTML 中（<4KB 的图片）
- 关键 CSS 内联到 HTML `<style>` 标签（Critical CSS）

**按需加载**：
- 图片懒加载
- 路由级代码分割，避免一次性下载所有 JS

**减少重定向**：每次重定向都是额外的往返，提前配置好 HTTPS 和 www 规范化，避免链式重定向。

**数据合并**：
- GraphQL：一次请求获取所有需要的数据，不用多次 REST 请求
- 批量接口：把多个小请求合并成一个批量请求

---

###### 8. 什么是 HTTP/2 服务器推送的应用场景？

HTTP/2 服务器推送（Server Push）允许服务器在客户端请求 HTML 时，预测性地推送相关资源（CSS/JS/图片），避免客户端需要额外请求。

**理想场景**：

```
客户端: GET /index.html
服务器:
  ← PUSH_PROMISE: /style.css   (提前推送 CSS)
  ← PUSH_PROMISE: /app.js      (提前推送 JS)
  ← 返回 index.html
  ← 返回 style.css 内容
  ← 返回 app.js 内容
// 客户端解析 HTML 发现需要 style.css 时，已经在本地了
```

**实际面临的挑战**：
- 很难判断该推什么——如果客户端有缓存，推送就是浪费带宽
- 客户端可以拒绝推送（RST_STREAM），但已造成带宽消耗
- 实现复杂，收益不稳定

**现实状况**：Chrome 已于 2022 年移除对 HTTP/2 Server Push 的支持，HTTP/3 规范也去掉了 Server Push。

**替代方案**（更实用）：
- `<link rel="preload">`：让浏览器提前请求关键资源，效果类似但更可控
- `103 Early Hints`：服务器在发送完整响应前，先发一些 `Link` 头，让浏览器提前预加载
- CDN 预热：提前把资源推到 CDN 节点

---

###### 9. 【追问】核心 Web 指标（Core Web Vitals）与 HTTP 性能优化的关系是什么？

Google 的核心 Web 指标是衡量用户体验的三大指标，HTTP 优化直接影响这些指标：

**LCP（Largest Contentful Paint，最大内容绘制）** → 主要内容加载时间：
- CDN 加速减少 TTFB（Time to First Byte）
- 关键资源 preload 提前加载
- 图片格式优化（WebP）、响应式图片

**FID（First Input Delay）/ INP（Interaction to Next Paint）** → 交互响应速度：
- 减少长任务，JS 代码分割、lazy load
- 延迟非关键 JS 加载（`<script defer>`）

**CLS（Cumulative Layout Shift，累积布局偏移）** → 视觉稳定性：
- 图片/视频预留尺寸，避免加载后撑开布局
- 字体 preload 避免 FOUT（字体加载导致的布局偏移）

**优化目标**（Good 阈值）：
- LCP < 2.5 秒
- INP < 200ms
- CLS < 0.1
