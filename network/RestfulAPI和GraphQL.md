---
date: 2026-10-11
tags:
  - network
  - restful
  - graphql
---
RESTful API 和 GraphQL 是两种对立的 API 设计范式 ，restfulapi的哲学是一切皆资源，用url标识资源

```
/users          → 用户集合
/users/123      → 单个用户
/users/123/posts → 用户的文章
```

## 介绍

在restfulapi中，http方法表意动词操作资源，比如GET方法读取 GET /user/123，POST创建，POST /users，PUT表全量更新 PATCH部分更新 DELETE删除。

**方法 + URL = 完整语义**。`DELETE /users/123` 一看就知道"删除 123 号用户"。每个请求都是独立的，服务端不会保存客户端状态，有统一接口，状态码表意。

REST好用是好用，也有几个缺点，一个是过渡获取，客户端要的字段少服务器返回一堆，比如你可能只要name，但是一次性拿到的是全部（其实和这个要后端端点的设计），一个页面需要多个资源可能要多个请求。后端要版本管理，加字段可能破坏兼容性。

GraphQL核心思想是描述客户端需求之后服务端再返回这些。只需要单一端点，客户端指定好字段就行：

```
POST /graphql

query {
    user(id: 123) {
        name
        posts {
            title
        }
    }
}

直接返回
{
    "data": {
        "user": {
            "name": "Alice",
            "posts": [
                { "title": "文章 1" },
                { "title": "文章 2" }
            ]
        }
    }
}
```

缺点是缓存复杂，rest可以用HTTP天然缓存，GRAPHQL要客户端缓存，查询有复杂度，嵌套查询可能触发大量数据库查询。

除此外，还有好几种其他的： gRPC、tRPC、WS、SOAP。

## options请求

在 RESTful API 中，**OPTIONS 请求**主要用于**协商跨域资源共享和**查询资源支持的 HTTP 方法。它本身不修改资源，属于“安全”和“幂等”的方法。

在非简单的请求前，浏览器会先发送OPTIONS请求去询问浏览器是否允许，比如触发场景是跨域+非简单请求，这个操作叫做CORS预检请求。

除此以外，OPTIONS会参与到了查询Allows头，客户端询问某个资源支持哪些HTTP方法，在手动调用或者工具探测的时候会发生。还有API能力发现，获取资源的原信息，支持的格式等。

我们最常用的请求叫做简单请求，就比如GET HEAD POST，请求头只包含：`Accept`、`Accept-Language`、`Content-Language`、`Content-Type`（且值只能是 `application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`）

非简单请求会触发OPTIONS，比如PUT DELETE PATCH CONNECT TRACE OPTIONS本身。 或 `POST` 但 `Content-Type` 是 `application/json`或带有自定义请求头（如 `Authorization`、`X-Custom-Header`）

那么这个说完了我们来看看浏览器在CORS预检中的请求是什么样子：

```
OPTIONS /api/users/1 HTTP/1.1
Origin: https://example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization
```

当浏览器发送了这个请求之后，可能返回的就是：

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

记住，浏览器会返回允许的源、允许的方法、允许的请求头、还有预检结果缓存时间，期间内不再重复预检。

## 安全性和幂等性

作为一个前端开发，我们有必要记忆清楚每个方法的安全性和幂等性：

|方法|用途|安全|幂等|典型状态码|
|---|---|---|---|---|
|`GET`|获取资源|✅|✅|200|
|`POST`|创建资源|❌|❌|201|
|`PUT`|全量替换资源|❌|✅|200 / 204|
|`PATCH`|部分更新资源|❌|✅|200 / 204|
|`DELETE`|删除资源|❌|✅|204|
|`HEAD`|获取响应头（无 body）|✅|✅|200|
|`OPTIONS`|查询支持的方法 / CORS 预检|✅|✅|204|
安全表示不修改资源，只读。幂等表示执行一次和多次请求的效果是相同的。**`POST` 既不安全也不幂等**，这是它和其他方法最大的区别。 **`PUT` 是幂等的**，多次 `PUT` 同一个资源，结果一样。

## 状态码

记住**分组含义**：

|分组|含义|常见|
|---|---|---|
|`1xx`|信息|很少用|
|`2xx`|成功|`200`、`201`、`204`|
|`3xx`|重定向|`301`、`302`、`304`|
|`4xx`|客户端错误|`400`、`401`、`403`、`404`、`405`、`429`|
|`5xx`|服务端错误|`500`、`502`、`503`|

**高频状态码**：

|状态码|含义|易错点|
|---|---|---|
|`200`|成功|通用|
|`201`|已创建|`POST` 创建资源后返回，通常带 `Location` 头|
|`204`|无内容|`DELETE`、`PUT` 成功后返回，无 body|
|`301`|永久重定向|资源永久换了 URL|
|`302`|临时重定向|资源临时换了 URL|
|`304`|未修改|缓存命中，客户端用本地缓存|
|`400`|请求错误|参数格式错、缺少字段|
|`401`|未认证|没登录，或 token 无效|
|`403`|无权限|登录了，但没权限访问|
|`404`|未找到|资源不存在|
|`405`|方法不允许|URL 存在，但不支持这个方法|
|`429`|请求过多|限流|
|`500`|服务器内部错误|代码异常|
|`502`|网关错误|上游服务挂了|
|`503`|服务不可用|服务维护或过载|
## 内容协商和其他分类

客户端和服务端通过 HTTP 头协商**数据格式**：

|请求头|作用|
|---|---|
|`Accept`|客户端想要什么格式，如 `application/json`|
|`Content-Type`|请求体的格式，如 `application/json`|
|`Accept-Language`|想要什么语言|
|`Accept-Encoding`|想要什么压缩格式|
ACCEPT是客户端向服务端发送自己想要的数据格式，Content-Type是发送方告诉接收方它发的数据是什么格式，如果服务端返回406，就表示Not Acceptable，无法满足Accept。

CORS相关的有很多的头，跨域的时候服务端需要返回这些头：

|响应头|作用|
|---|---|
|`Access-Control-Allow-Origin`|允许的源|
|`Access-Control-Allow-Methods`|允许的方法|
|`Access-Control-Allow-Headers`|允许的请求头|
|`Access-Control-Allow-Credentials`|是否允许携带 cookie|
|`Access-Control-Max-Age`|预检结果缓存时间|
|`Access-Control-Expose-Headers`|客户端能读取的响应头|
请求头也用来认证和授权，比如我们常见的Bearer Token，`Authorization: Bearer <token>`，最常用。或者`API KEY：X-API-Key: <key>` 等。

其他的还有缓存控制、条件请求、范围请求、代理转发、连接管理、安全、追踪，这个我先不讲，HTTP笔记我会细讲。