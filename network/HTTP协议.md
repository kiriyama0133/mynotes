---
date: 2026-12-10
tags:
  - network
  - http
---
HTTP是工作在应用层的协议，用于在客户端比如浏览器和服务器之间传输数据，是web的基石，采用的是请求-响应模型，无状态、可扩展。

状态码来说：

1xx比如101 Switching Protocols。2xx开头是成功的，比如200 OK 201 Created、204 No Content。3xx开头为重定向，301是永久重定向，302是临时的。4xx开头是客户端错误，5开头的是服务端错误。

## 常见头部

常见请求头/响应头

| 头部                                    | 作用                   |
| ------------------------------------- | -------------------- |
| `Host`                                | 目标主机（HTTP/1.1 必需）    |
| `Content-Type`                        | 报文体的 MIME 类型         |
| `Content-Length`                      | 报文体长度                |
| `Accept`                              | 客户端可接受的类型            |
| `Authorization`                       | 认证凭证（如 Bearer Token） |
| `Cookie` / `Set-Cookie`               | 会话状态管理               |
| `Cache-Control`                       | 缓存策略                 |
| `ETag` / `If-None-Match`              | 协商缓存                 |
| `Last-Modified` / `If-Modified-Since` | 协商缓存                 |
| `Access-Control-Allow-Origin`         | CORS 跨域许可            |
浏览器同源策略限制跨域请求。CORS 通过响应头授权：

```
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST
Access-Control-Allow-Headers: Content-Type
Access-Control-Allow-Credentials: true
```

**预检请求**：非简单请求（如 PUT、自定义头）会先发 `OPTIONS` 请求确认。OPTIONS请求，cors预检请求的响应状态码通常是204 No Content。当使用了非简单方法比如PUT、DELETE、PATCH等的时候就会自动发送这个，或是使用了自定义请求头Authorization、X-Custom-Header也会。这个在另一个笔记里 [[RestfulAPI和GraphQL]] 这里有讲过，可以直接看。


## 长连接

http长连接核心是复用一个tcp连接发送多个请求，避免每次都重新握手等。短连接是http/1.0默认，每次都要经过tcp握手。长连接是http/1.1默认（通过keep-alive复用实现），多个请求会复用一个tcp。但是http/1.1会有队头阻塞问题，同一个连接上，前一个请求还没有响应完，后面的请求要等待。h2里面连接本身就是一个多路复用器了。

>只需要一个tcp连接是设计目标但是不是绝对的，准确说，每个origin通常有一条，不是全站一条

http/2的多路复用解决了这个问题：

```
一个 TCP 连接
    ├── Stream 1: 请求 A
    ├── Stream 2: 请求 B
    ├── Stream 3: 请求 C
    └── 并发传输，互不阻塞
```

<svg viewBox="0 0 680 360" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>HTTP/1.1 与 HTTP/2 的连接模型对比</title>
<desc>HTTP/1.1 每个域名开多条 TCP 连接，每条连接上的请求串行排队；HTTP/2 每个域名只用一条 TCP 连接，多个流在同一连接上并行复用。</desc>
<style>
.bx{stroke-width:.5}
.tt{font:500 14px ui-sans-serif,system-ui,sans-serif}
.cp{font:400 12px ui-sans-serif,system-ui,sans-serif}
.sm{font:500 11px ui-monospace,SFMono-Regular,Consolas,monospace}
</style>
<rect class="bx" x="40" y="40" width="280" height="300" rx="16" fill="#FFFFFF" stroke="#D3D1C7"/>
<text class="tt" x="60" y="68" dominant-baseline="central" fill="#633806">HTTP/1.1</text>
<text class="cp" x="60" y="90" dominant-baseline="central" fill="#854F0B">每域名 6 条连接，请求串行排队</text>
<rect class="bx" x="60" y="120" width="56" height="180" rx="6" fill="#FAEEDA" stroke="#EF9F27"/>
<rect class="bx" x="68" y="132" width="40" height="44" rx="4" fill="#FFFFFF" stroke="#EF9F27"/>
<text class="sm" x="88" y="154" text-anchor="middle" dominant-baseline="central" fill="#854F0B">1</text>
<rect class="bx" x="68" y="188" width="40" height="44" rx="4" fill="#FFFFFF" stroke="#EF9F27"/>
<text class="sm" x="88" y="210" text-anchor="middle" dominant-baseline="central" fill="#854F0B">2</text>
<rect class="bx" x="68" y="244" width="40" height="44" rx="4" fill="#FFFFFF" stroke="#EF9F27"/>
<text class="sm" x="88" y="266" text-anchor="middle" dominant-baseline="central" fill="#854F0B">3</text>
<rect class="bx" x="152" y="120" width="56" height="180" rx="6" fill="#FAEEDA" stroke="#EF9F27"/>
<rect class="bx" x="160" y="132" width="40" height="44" rx="4" fill="#FFFFFF" stroke="#EF9F27"/>
<text class="sm" x="180" y="154" text-anchor="middle" dominant-baseline="central" fill="#854F0B">1</text>
<rect class="bx" x="160" y="188" width="40" height="44" rx="4" fill="#FFFFFF" stroke="#EF9F27"/>
<text class="sm" x="180" y="210" text-anchor="middle" dominant-baseline="central" fill="#854F0B">2</text>
<rect class="bx" x="160" y="244" width="40" height="44" rx="4" fill="#FFFFFF" stroke="#EF9F27"/>
<text class="sm" x="180" y="266" text-anchor="middle" dominant-baseline="central" fill="#854F0B">3</text>
<rect class="bx" x="244" y="120" width="56" height="180" rx="6" fill="#FAEEDA" stroke="#EF9F27"/>
<rect class="bx" x="252" y="132" width="40" height="44" rx="4" fill="#FFFFFF" stroke="#EF9F27"/>
<text class="sm" x="272" y="154" text-anchor="middle" dominant-baseline="central" fill="#854F0B">1</text>
<rect class="bx" x="252" y="188" width="40" height="44" rx="4" fill="#FFFFFF" stroke="#EF9F27"/>
<text class="sm" x="272" y="210" text-anchor="middle" dominant-baseline="central" fill="#854F0B">2</text>
<rect class="bx" x="252" y="244" width="40" height="44" rx="4" fill="#FFFFFF" stroke="#EF9F27"/>
<text class="sm" x="272" y="266" text-anchor="middle" dominant-baseline="central" fill="#854F0B">3</text>
<text class="cp" x="60" y="318" dominant-baseline="central" fill="#5F5E5A">一个慢请求挡住后续请求</text>
<rect class="bx" x="360" y="40" width="280" height="300" rx="16" fill="#FFFFFF" stroke="#D3D1C7"/>
<text class="tt" x="380" y="68" dominant-baseline="central" fill="#0C447C">HTTP/2</text>
<text class="cp" x="380" y="90" dominant-baseline="central" fill="#185FA5">每域名 1 条连接，多路复用</text>
<rect class="bx" x="380" y="120" width="240" height="180" rx="8" fill="#E6F1FB" stroke="#85B7EB"/>
<text class="sm" x="392" y="154" dominant-baseline="central" fill="#185FA5">S1</text>
<rect class="bx" x="414" y="137" width="44" height="34" rx="4" fill="#FFFFFF" stroke="#85B7EB"/>
<rect class="bx" x="464" y="137" width="44" height="34" rx="4" fill="#FFFFFF" stroke="#85B7EB"/>
<rect class="bx" x="514" y="137" width="44" height="34" rx="4" fill="#FFFFFF" stroke="#85B7EB"/>
<rect class="bx" x="564" y="137" width="44" height="34" rx="4" fill="#FFFFFF" stroke="#85B7EB"/>
<text class="sm" x="392" y="210" dominant-baseline="central" fill="#185FA5">S2</text>
<rect class="bx" x="414" y="193" width="44" height="34" rx="4" fill="#FFFFFF" stroke="#85B7EB"/>
<rect class="bx" x="464" y="193" width="44" height="34" rx="4" fill="#FFFFFF" stroke="#85B7EB"/>
<rect class="bx" x="514" y="193" width="44" height="34" rx="4" fill="#FFFFFF" stroke="#85B7EB"/>
<rect class="bx" x="564" y="193" width="44" height="34" rx="4" fill="#FFFFFF" stroke="#85B7EB"/>
<text class="sm" x="392" y="266" dominant-baseline="central" fill="#185FA5">S3</text>
<rect class="bx" x="414" y="249" width="44" height="34" rx="4" fill="#FFFFFF" stroke="#85B7EB"/>
<rect class="bx" x="464" y="249" width="44" height="34" rx="4" fill="#FFFFFF" stroke="#85B7EB"/>
<rect class="bx" x="514" y="249" width="44" height="34" rx="4" fill="#FFFFFF" stroke="#85B7EB"/>
<rect class="bx" x="564" y="249" width="44" height="34" rx="4" fill="#FFFFFF" stroke="#85B7EB"/>
<text class="cp" x="380" y="318" dominant-baseline="central" fill="#5F5E5A">多个流并行，互不阻塞</text>
</svg>

左边是为什么当年要做 h2:浏览器被逼着对一个域名开 6 条 TCP,每条上面请求还得排成一队,一个慢接口就把后面全卡住。h2 把这些塌缩成右边这一条连接,里面用 **stream(流)​ 区分不同请求,可以交错传输。

>1. 刚才我们说过tcp按照origin算，不是按照页面算，如果站点使用了CDN、图床、第三方统计，那就是好几个不同的origin，各开各的。h2的连接合并不能跨origin。
>2. 浏览器没有义务只保持一条。连接建立慢时会开"竞速连接",连接空闲久了会回收,有网络切换或异常时也可能重建。

HTTP版本协商路径与占比：

<svg viewBox="0 0 680 340" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>HTTP 版本是怎么协商出来的</title>
<desc>浏览器只在 HTTPS 上通过 TLS 的 ALPN 扩展协商协议版本，服务端在 h3、h2、http/1.1 中挑一个；明文 http 不参与协商，只能用 HTTP/1.1。</desc>
<style>
.bx{stroke-width:.5}
.tt{font:500 13px ui-sans-serif,system-ui,sans-serif}
.cp{font:400 12px ui-sans-serif,system-ui,sans-serif}
.sm{font:500 11px ui-monospace,SFMono-Regular,Consolas,monospace}
</style>
<defs>
<marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
</marker>
</defs>
<rect class="bx" x="220" y="40" width="240" height="44" rx="8" fill="#F4F3F0" stroke="#D3D1C7"/>
<text class="tt" x="340" y="62" text-anchor="middle" dominant-baseline="central" fill="#2C2C2A">浏览器请求 https://…</text>
<path d="M340 84 V118" fill="none" stroke="#888780" stroke-width="1.5" marker-end="url(#arrow)"/>
<rect class="bx" x="180" y="124" width="320" height="56" rx="8" fill="#E6F1FB" stroke="#85B7EB"/>
<text class="tt" x="340" y="143" text-anchor="middle" dominant-baseline="central" fill="#0C447C">TLS 握手 · ALPN 扩展</text>
<text class="cp" x="340" y="163" text-anchor="middle" dominant-baseline="central" fill="#185FA5">客户端列出 h3 / h2 / http/1.1</text>
<path d="M340 180 V200" fill="none" stroke="#888780" stroke-width="1.5"/>
<path d="M132 200 H548" fill="none" stroke="#888780" stroke-width="1.5"/>
<path d="M132 200 V220" fill="none" stroke="#888780" stroke-width="1.5" marker-end="url(#arrow)"/>
<path d="M340 200 V220" fill="none" stroke="#888780" stroke-width="1.5" marker-end="url(#arrow)"/>
<path d="M548 200 V220" fill="none" stroke="#888780" stroke-width="1.5" marker-end="url(#arrow)"/>
<text class="sm" x="356" y="212" dominant-baseline="central" fill="#888780">服务端挑一个</text>
<rect class="bx" x="40" y="224" width="184" height="72" rx="8" fill="#E1F5EE" stroke="#5DCAA5"/>
<text class="tt" x="132" y="246" text-anchor="middle" dominant-baseline="central" fill="#085041">HTTP/3</text>
<text class="cp" x="132" y="266" text-anchor="middle" dominant-baseline="central" fill="#0F6E56">QUIC over UDP</text>
<text class="sm" x="132" y="286" text-anchor="middle" dominant-baseline="central" fill="#0F6E56">≈21% 请求</text>
<rect class="bx" x="248" y="224" width="184" height="72" rx="8" fill="#E6F1FB" stroke="#85B7EB"/>
<text class="tt" x="340" y="246" text-anchor="middle" dominant-baseline="central" fill="#0C447C">HTTP/2</text>
<text class="cp" x="340" y="266" text-anchor="middle" dominant-baseline="central" fill="#185FA5">TCP · 多路复用</text>
<text class="sm" x="340" y="286" text-anchor="middle" dominant-baseline="central" fill="#185FA5">≈51% 请求</text>
<rect class="bx" x="456" y="224" width="184" height="72" rx="8" fill="#F4F3F0" stroke="#B4B2A9"/>
<text class="tt" x="548" y="246" text-anchor="middle" dominant-baseline="central" fill="#2C2C2A">HTTP/1.1</text>
<text class="cp" x="548" y="266" text-anchor="middle" dominant-baseline="central" fill="#5F5E5A">TCP · 串行复用</text>
<text class="sm" x="548" y="286" text-anchor="middle" dominant-baseline="central" fill="#5F5E5A">≈28% 请求</text>
<text class="cp" x="40" y="318" dominant-baseline="central" fill="#5F5E5A">明文 http:// 不参与 ALPN 协商，浏览器实际只用 HTTP/1.1</text>
</svg>

h2 只跑在 TLS 上。浏览器的明文 h2(`h2c`)支持早就移除了,所以 `http://` 站点实际只能是 h1.1。这就是为什么"h2 默认"约等于"HTTPS 默认"。通过一个连接并发多个请求，将请求/响应拆成帧，交错传输和头部压缩的方式解决了，http/2的长连接更加高效，一个连接搞定所有请求。

所以，阅读完上面的文字，我们就能清楚。使用明文的http的时候基本就是h1.1协议，"h2 需要 TLS"、"尽量维持一条连接更好"，一条连接同时省掉 TCP 和 TLS 两套握手成本,而且**版本是绑死在连接上的**(ALPN 在建立时定,中途不能换),所以换连接就等于重来一遍。


长连接通过Connection: keep-alive声明，依靠tcp keep-alive传输层：

```
setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &on, sizeof(on));
```

可以检测连接是否还或者，对端挂了，网络断了。应用层自己会发送心跳包，比如说30s一个，来保持链接活跃。如果说应用崩溃，连接不会通过FIN关闭的话，内核会替他关闭，可能是RST，也可能是FIN，比如说进程如果被SIGKILL 杀死，还有未发送的消息，他就会发RST，处理完了就会发FIN。



## 头部压缩

头部压缩式http/2的核心优化，因为http/1.1头部每次都会携带大量重复头部：

```
GET /index.html
Host: example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...
Accept: text/html,application/xhtml+xml,...
Accept-Language: zh-CN,zh;q=0.9
Accept-Encoding: gzip, deflate, br
Cookie: session=abc123; theme=dark; ...
Connection: keep-alive
```

下一个请求也是一样。每个头部都会重复传递这些，显然会浪费带宽，**HTTP/1.1 用 gzip 压缩 body，但不压缩头部**——头部一直是明文。**HTTP/2 引入头部压缩**——**HPACK**。61个常见头部，然后给编号压缩，这就是静态表，对于连接期间动态维护的头部，使用索引来压缩，这就是静态表。对字符串使用哈夫曼编码，这样就可以让头部压缩很大一部分。

## https

我们都知道http是明文传输，数据裸奔，中间人可以直接看到密码、cookie等信息 还能篡改，https可以用tls解决这些问题，https分为四层：

```
HTTP          ← 应用层
  ↓
TLS/SSL       ← 安全层（加解密、认证、完整性）
  ↓
TCP           ← 传输层
  ↓
IP            ← 网络层
```

https = http + tls，http报文会被tls加密之后走tcp传输。tls使用非对称加密，在握手阶段解决。公钥是公开的，私钥是服务器自己保留，握手阶段会协商出**会话秘钥**，对称加密使用一个秘钥加解密，在**非对称加密协商**的时候，这个秘钥会被协商出来。

我们用tls1.2版本解释：

```
客户端                                服务器
  │                                    │
  │ ① Client Hello                    │
  │   - 支持的 TLS 版本                 │
  │   - 支持的加密套件                   │
  │   - 客户端随机数（Random1）          │
  │ ─────────────────────────────────> │
  │                                    │
  │ ② Server Hello                    │
  │   - 选定的 TLS 版本                 │
  │   - 选定的加密套件                   │
  │   - 服务器随机数（Random2）          │
  │ <───────────────────────────────── │
  │                                    │
  │ ③ Certificate                     │
  │   - 服务器证书（含公钥）             │
  │ <───────────────────────────────── │
  │                                    │
  │ ④ Server Key Exchange（可选）      │
  │   - 密钥交换参数                    │
  │ <───────────────────────────────── │
  │                                    │
  │ ⑤ Server Hello Done               │
  │ <───────────────────────────────── │
  │                                    │
  │ ⑥ 客户端验证证书                    │
  │   - 检查证书是否可信                 │
  │   - 检查域名是否匹配                 │
  │   - 检查是否过期                    │
  │                                    │
  │ ⑦ Client Key Exchange             │
  │   - 用服务器公钥加密"预主密钥"       │
  │ ─────────────────────────────────> │
  │                                    │
  │ ⑧ Change Cipher Spec              │
  │   - 通知"后续用协商的密钥加密"       │
  │ ─────────────────────────────────> │
  │                                    │
  │ ⑨ Finished                        │
  │   - 加密的握手摘要                  │
  │ ─────────────────────────────────> │
  │                                    │
  │ ⑩ Change Cipher Spec              │
  │ <───────────────────────────────── │
  │                                    │
  │ ⑪ Finished                        │
  │ <───────────────────────────────── │
  │                                    │
  │ ⑫ 开始加密通信（对称加密）           │
  │ <════════════════════════════════> │
```

客户端会检查证书链，证书是否是可信CA签发，浏览器会内置CA列表。证书里面的域名是否匹配访问的域名，以及是否过期。客户端会生成**预主秘钥**，用于服务器公钥加密，发送给服务器，服务器用**私钥解密**，拿到预主密钥。

```
主密钥 = PRF(预主密钥, "master secret", Random1 + Random2)
```

主密钥用来生成会话秘钥，也就是对称加密的秘钥，这样握手就完成了。