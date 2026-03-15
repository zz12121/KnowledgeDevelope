# 九、HTTP 内容协商

###### 1. 什么是内容协商（Content Negotiation）？

内容协商是指：**客户端和服务器协商决定响应的具体内容格式**——同一个 URL 可以根据客户端能力返回不同格式的数据（不同语言、不同编码、不同媒体类型等）。

**协商方式**：

**主动协商（客户端驱动）**：客户端在请求头里声明自己的偏好：
- `Accept`：期望的媒体类型
- `Accept-Encoding`：支持的压缩方式
- `Accept-Language`：期望的语言
- `Accept-Charset`：支持的字符集（现代已基本废弃，都 UTF-8 了）

服务器根据这些偏好选择最匹配的格式响应，并在响应头 `Content-Type`、`Content-Encoding`、`Content-Language` 里告知实际使用的格式。

**被动协商（服务器驱动）**：服务器返回多个可用版本的列表（配合 `Vary` 头），由客户端选择。实际很少用。

**内容协商在 RESTful API 中的应用**：同一个 `/api/users/1` 接口，根据 `Accept` 头返回 JSON 或 XML 格式。

---

###### 2. Accept 请求头的作用是什么？

`Accept` 告诉服务器**客户端能接受的响应内容类型**（MIME 类型），多个类型用逗号分隔，可以用 `q=` 参数指定优先级（质量因子，0-1，默认 1）。

```
Accept: application/json                        # 只要 JSON
Accept: text/html, application/json             # HTML 或 JSON 都行
Accept: application/json;q=0.9, text/html;q=0.8 # JSON 优先，HTML 次之
Accept: */*                                     # 什么都行（浏览器默认值）
```

**服务器处理**：
- 找到匹配的类型 → 返回对应格式，响应头 `Content-Type` 指明实际格式
- 没有匹配的类型 → 返回 **406 Not Acceptable**

**Spring MVC 中的体现**：

```java
// produces 控制这个方法能生产哪些媒体类型
@GetMapping(value = "/user", produces = {
    MediaType.APPLICATION_JSON_VALUE,
    MediaType.APPLICATION_XML_VALUE
})
public User getUser() { ... }
// 请求 Accept: application/xml 就返回 XML 格式
```

---

###### 3. Accept-Encoding 请求头的作用是什么？

`Accept-Encoding` 告诉服务器**客户端支持哪些压缩算法**，服务器选择合适的算法压缩响应体，在 `Content-Encoding` 里告知实际使用的算法。

```
Accept-Encoding: gzip, deflate, br
Accept-Encoding: br;q=1.0, gzip;q=0.8, *;q=0.1
```

**常见压缩算法**：
- **gzip**：最广泛支持，压缩率好（对文本类型约 60-80%），几乎所有客户端都支持
- **deflate**：基于 zlib，实现有历史争议，实际使用少
- **br（Brotli）**：Google 开发，压缩率比 gzip 高 15-20%，现代浏览器都支持，推荐优先使用
- **zstd（Zstandard）**：Facebook 开发，速度更快，新版本 Chrome/Firefox 支持

**Nginx 配置 gzip**：
```nginx
gzip on;
gzip_types text/plain application/json application/javascript text/css;
gzip_min_length 1024;      # 小于 1KB 不压缩（压缩开销不值得）
gzip_comp_level 6;          # 压缩级别 1-9
```

注意：压缩对图片、视频等已压缩格式效果很差甚至负优化，应该限制只压缩文本类型。

---

###### 4. Accept-Language 请求头的作用是什么？

`Accept-Language` 告诉服务器**用户偏好的语言**，服务器据此返回对应语言的内容（国际化 i18n）。

```
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-US;q=0.7
```

含义：首选简体中文，其次泛中文，再次英语，最后美式英语。

**服务器处理**：
- 有对应语言的内容 → 返回，响应头 `Content-Language: zh-CN`
- 没有对应语言 → 返回默认语言或 406 Not Acceptable
- 响应头 `Vary: Accept-Language` 告知缓存服务器：相同 URL 但不同语言是不同的响应，分别缓存

**Spring Boot 国际化应用**：

```java
// Spring Boot 国际化：读取 messages_zh_CN.properties 或 messages_en.properties
@Bean
public LocaleResolver localeResolver() {
    AcceptHeaderLocaleResolver resolver = new AcceptHeaderLocaleResolver();
    resolver.setDefaultLocale(Locale.CHINESE);
    return resolver;
}
```

---

###### 5. Content-Type 响应头的作用是什么？

`Content-Type` 告诉接收方**响应体（或请求体）的媒体类型和字符编码**，让接收方知道如何解析数据。

```
Content-Type: application/json; charset=UTF-8
Content-Type: text/html; charset=UTF-8
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary
Content-Type: image/png
```

**作用**：
- 响应头：浏览器根据 Content-Type 决定如何渲染（显示为 HTML、下载文件、播放音频...）
- 请求头：服务器根据 Content-Type 决定如何解析请求体

**常见类型**：
- `application/json`：JSON 数据（最常用的 API 格式）
- `application/x-www-form-urlencoded`：HTML 表单默认编码（`name=value&age=25`）
- `multipart/form-data`：上传文件时必须用这个
- `text/html; charset=UTF-8`：HTML 页面
- `application/octet-stream`：二进制流，触发浏览器下载

**MIME 嗅探安全问题**：浏览器有时会忽略 Content-Type 猜测内容类型（MIME 嗅探），可能导致安全漏洞。用 `X-Content-Type-Options: nosniff` 响应头禁用。

---

###### 6. 什么是 MIME 类型？

MIME（Multipurpose Internet Mail Extensions，多用途互联网邮件扩展）是描述数据类型的标准格式，最初为电子邮件设计，后来被 HTTP 广泛采用。

**格式**：`type/subtype`，可选 `;parameter`

**主要类型（type）**：
- `text`：文本类型（text/html, text/plain, text/css）
- `image`：图片（image/png, image/jpeg, image/webp, image/svg+xml）
- `audio`：音频（audio/mp3, audio/ogg）
- `video`：视频（video/mp4, video/webm）
- `application`：应用程序数据（application/json, application/pdf, application/zip）
- `multipart`：多部分数据（multipart/form-data, multipart/mixed）

**常见 MIME 类型速查**：

| 文件类型 | MIME 类型 |
|---------|----------|
| HTML | text/html |
| CSS | text/css |
| JavaScript | application/javascript |
| JSON | application/json |
| XML | application/xml |
| PDF | application/pdf |
| ZIP | application/zip |
| PNG | image/png |
| JPEG | image/jpeg |
| GIF | image/gif |
| WebP | image/webp |
| MP4 | video/mp4 |

---

###### 7. 【追问】Vary 响应头的作用是什么？

`Vary` 头告诉缓存服务器（CDN、代理）：**缓存时除了 URL 之外，还要把哪些请求头也作为缓存 Key 的一部分**。

```
Vary: Accept-Encoding
Vary: Accept-Language
Vary: Accept-Encoding, Accept-Language
```

**为什么需要 Vary**：

同一个 URL `/api/data`，A 客户端发 `Accept-Encoding: gzip` 得到的是压缩版，B 客户端不支持压缩得到的是原始版。如果 CDN 把 A 得到的压缩版缓存后，B 请求同一 URL 拿到了压缩版但无法解压，就会出错。

加上 `Vary: Accept-Encoding` 后，CDN 会把 `Accept-Encoding` 不同的响应分开缓存，各取各的。

**常见 Vary 设置**：
- `Vary: Accept-Encoding`：几乎所有支持 gzip/br 的服务都应该设置
- `Vary: Accept-Language`：多语言网站
- `Vary: Accept`：同一 URL 支持多种格式（JSON/XML）

**注意**：Vary 值越多，缓存命中率越低（每种组合都是独立缓存），要合理使用。
