---
date: 2026-10-22
tags:
  - architecture
  - cloudflare
  - worker
  - serverless
---
Cloudflare Workers 是一种将 JavaScript/TypeScript 代码直接部署到 Cloudflare 全球边缘网络运行的无服务器（Serverless）计算平台。它的核心优势在于极低的延迟、近乎为零的冷启动时间，以及为全栈应用提供的一整套边缘原生生态。

理解 Workers 的关键在于它的执行模型，它与传统的基于容器的 Serverless（如 AWS Lambda）有本质不同。

workers没有使用虚拟机或者容器，而是利用了chrome和nodejs同款的v8引擎中的isolate技术，可以理解为极其轻量的js沙箱，多个isolate可以共享同一个运行时进程，内存完全隔离。由于不需要启动完整的操作系统和运行时，因此一个isolate的启动速度比传统容器快了约100倍，通常是5ms，用户是几乎感觉不到的，这个使得worker非常适合处理延迟敏感的API请求。

代码会运行在cloudflare遍布全球的300多个城市节点上，请求会被自动路由到最近的节点处理，它的计费基于请求数和CPU计算时间，而不是空闲的资源占用。

## 作为中间层处理请求

之前遇到的 `dog.ceo` 图片请求来做例子。浏览器直接 `fetch` 第三方 API 会被 CORS 策略拦截，因为目标服务器没有返回允许跨域的响应头。

Workers 作为一个中间层，可以完美解决这个问题。它的逻辑很简单：**让浏览器请求你的 Worker（同源），由 Worker 在服务端（不受 CORS 限制）去请求目标 API，并在返回给浏览器的响应中手动加上 CORS 头**。

```ts
export default {
  async fetch(request) {
    // 1. 从请求的 URL 中提取真正的目标地址
    const url = new URL(request.url);
    const targetUrl = url.searchParams.get("url");
    if (!targetUrl) {
      return new Response("Missing 'url' parameter", { status: 400 });
    }

    // 2. 在服务端发起对目标地址的请求（此处没有 CORS 限制）
    const response = await fetch(targetUrl);

    // 3. 复制响应，并注入 CORS 头，允许任何来源访问
    const newHeaders = new Headers(response.headers);
    newHeaders.set("Access-Control-Allow-Origin", "*");
    
    return new Response(response.body, {
      status: response.status,
      headers: newHeaders,
    });
  },
};
```

部署后，前端只需请求 `https://你的worker.workers.dev/?url=https://dog.ceo/api/breeds/image/random` 即可获取数据，浏览器不会再报 CORS 错误。

## Bindings

Workers 本身是无状态的，它通过 **Bindings** 这一机制来连接各种外部资源，这是构建全栈应用的基础。你不需要在代码里管理数据库密码或连接字符串，只需在配置中声明绑定，运行时通过 `env` 对象访问即可。

常见的绑定包括：

- **KV**：低延迟的键值存储，适合存储会话、配置或缓存。
    
- **D1**：基于 SQLite 的边缘关系型数据库，支持标准的 SQL 查询。
    
- **R2**：与 S3 兼容的对象存储，且**没有出口流量费用**，适合存放图片、文件等静态资源。
    
- **Service Bindings**：允许一个 Worker 直接调用另一个 Worker，用于构建微服务架构。
    
- **Durable Objects**：提供强一致性的有状态协调能力，适合实现实时协作、聊天室等场景。

我们之前熟悉 **Nitro** 微后端开发，它可以直接构建并部署到 Cloudflare Workers。你几乎不需要重写业务逻辑，只需在 `nitro.config.ts` 中指定 `preset: "cloudflare_module"`，Nitro 就会将你的代码打包成 Workers 可运行的格式。之后使用 `wrangler deploy` 即可发布。

在 Workers 生态中，**Hono** 是一个备受推崇的轻量级 Web 框架，它的 API 设计对熟悉 Express 的开发者非常友好，且完全基于 Web 标准（`Request`/`Response`）构建。你可以把它看作是 Workers 环境下的“Express”，是构建 API 和代理服务的高效选择。

>浏览器web worker是我们用来执行用户js代码、防止主线程卡死的方案，运行在用户的浏览器里，是前端页面的一部分。但是cloudflare workers是部署服务端代码的地方，运行在边缘网络上，是一个独立的serverless运行环境。

cloudflare workers和nodejs最大区别在于workers运行时基于web标准，也就是Request、Response、fetch构建，但是nodejs有自己的标准库fs http net，为了让你能在workers上使用nodejs api的npm包，cloudflare提供了的nodejs_compat兼容性标志，对于兼容性在2026-08-04或者之后的worker，这个标志是默认开启的。

尽管兼容性很好，但有些 Node.js 的能力在 Workers 的 Serverless 模型中**无法支持**，或者在实现上有本质区别：

- **文件系统 (`fs`) 是“虚拟”的**：Workers 没有传统的本地磁盘。`node:fs` 模块虽然可用，但它操作的是一个**基于内存的、每个 Worker 独立的虚拟文件系统**，用于处理临时文件，而非持久化存储。
    
- **C++ 原生插件不支持**：依赖 Node-API 的 C++ Addons 无法在 Workers 中运行。
    
- **部分模块完全不可用**：`Child Processes`、`Cluster`、`Readline`、`TTY`、`UDP/datagram` 等在 Serverless 上下文中没有意义的模块是不支持的。
    
- **网络实现是封装而非原生**：`node:net` 和 `node:http` 并非直接操作内核套接字，而是 Cloudflare 基于自己的 **Sockets API** 和 **`fetch` API** 封装实现的。这意味着其行为和底层机制与在 Node.js 中直接使用内核网络栈不同。

部署方面的话，wrangler目前支持两种格式，可以选一，你可以写wrangler.toml或者是wrangler.jsonc，后者是目前更推荐的新格式，可以更好地获得ide的语法提示和校验。


```toml
name = "my-nitro-app"
main = "./.output/server/index.mjs"
compatibility_date = "2024-11-12"
compatibility_flags = ["nodejs_compat"]

# 指定静态资源目录，并绑定到 ASSETS 变量
[assets]
directory = "./.output/public"
binding = "ASSETS"
```


```json
//jsonc
{
  // 启用 IDE 智能提示和校验（可选，但强烈推荐）
  "$schema": "node_modules/wrangler/config-schema.json",

  // 【核心配置】
  "name": "my-nitro-app",           // Worker 项目名称
  "main": ".output/server/index.mjs", // 入口文件（Nitro 构建后的路径）
  
  // 【兼容性设置】
  "compatibility_date": "2026-10-24", // 建议设置为当前日期
  "compatibility_flags": ["nodejs_compat"], // 开启 Node.js API 兼容

  // 【静态资源绑定】（Nitro/Nuxt 项目常用）
  "assets": {
    "directory": ".output/public",  // 静态资源目录
    "binding": "ASSETS"            // 在代码中通过 env.ASSETS 访问
  },

  // 【环境变量】（非敏感信息）
  "vars": {
    "API_ENDPOINT": "https://api.example.com"
  },

  // 【KV 命名空间绑定】（示例）
  "kv_namespaces": [
    {
      "binding": "MY_KV",
      "id": "<YOUR_KV_NAMESPACE_ID>"
    }
  ]
}
```

当终端执行了wrangler login登录了之后，就会启动一个标准的oauth2.0流程，授权了wranler代理之后本地就会保存凭证，之后在同一台机器部署的时候就不需要登录了。

Nuxt 在默认执行 `nuxt build`（Node 预设）后，Nitro 构建出的服务端入口文件名称是 **`index.mjs`**，完整路径位于项目根目录的 `.output/server/index.mjs`。

Free Plan:

适合个人项目、学习或低流量场景，每月成本 $0。

请求数：每天 100,000 次（涵盖所有 Worker 脚本，UTC 0点重置）。

CPU 时间：每次请求 10 毫秒。注意，这只是 CPU 计算时间，等待网络请求（如 fetch）的时间不计算在内。

其他限制：最多 100 个 Worker 脚本，每天 200,000 条日志。

## Serverless

通常，Serverless 包含两个层面：

1. **FaaS (Function as a Service，函数即服务)**：这是 Serverless 的核心。你只写一个个独立的函数，只有当请求来的时候，平台才会启动一个实例来执行你的函数，执行完立刻销毁。**按运行时间或请求次数计费**（没请求时完全不花钱）。比如AWS Lambda、阿里云函数计算（FC）、腾讯云云函数（SCF）、Google Cloud Functions。
    
2. **BaaS (Backend as a Service，后端即服务)**：比如云数据库（如你的 D1）、对象存储（如 R2）、身份认证服务。你直接调 API 就能用，不用自己建数据库服务器。前端直接连数据库，权限、登录、存储全用平台提供的现成功能，甚至不需要自己写后端代码。

serverless有四个特征，不用管服务器，不用打补丁，不用担心宕机问题，是零维护。从0到10000个并发，是由平台搞定，半夜没人访问的时候自动缩容到0，流量洪峰的时候能瞬间拉起几万个实例来抗住。按量计费，用多少算多少。函数通常由特定事件触发，比如一个http请求、一条数据库更新、一张图片上传。

那么具体业务该如何设计，肯定不可能把核心主干给剥离出去，我们可以把一些无状态、短生命周期、事件触发、按需运行的分离出去。比如说webhook接收器，接入了微信支付、GitHub、Stripe 等第三方服务。当用户支付成功时，第三方会发一个 HTTP 请求到你的服务器。或者是文件和图片处理，我们需要裁切为圆形、生成缩略图，加水印等操作，也可以分离出去，因为结合对象存储比如cloudflare R2/AWS S3，一旦有文件上传，事件触发之后就会自动调用FaaS函数进行图片压缩，处理完了存回去，极其省钱且响应极快。一些定时任务，比如凌晨2点清理数据库过期验证码、同步第三方汇率等等，生成昨日报表等等，也可以，因为云厂商都支持定时触发FaaS，不需要专门搞一台机器挂个Crontab。边缘代理和BFF也可以。

适合作为baas拆分出去的比如说身份认证和权限管理，我们可以直接用firebase Auth， Supabase Auth都可以，或者国内的Authing 、腾讯云开发。文件存储比如用户头像、商品图片、视频、附件等等都可以自检搭建fastDFS或者minio，也可以直接用cloudflare r2/aws s3、阿里云os、配合cdn、全球自带加速和极低的防盗链成本。之前提到的 Cloudflare D1 (基于 SQLite)，或者 Supabase（基于 PostgreSQL），己装 MySQL、Redis 还要搞主从复制、容灾备份 -> 直接用 Serverless DB，自带按需扩缩容，而且很多直接提供前端 SDK，前端可以直接查数据库（靠行级安全策略保护数据）。消息推送比比如app推送到手机、网站弹窗、邮件发送、短信发送等等，可以用baas平台接管，不需要自己维护推送服务器和长连接池。搜索引擎不需要自己搭建庞大的Elasticsearch 集群，直接用 Algolia 或 Meilisearch 的云服务。





