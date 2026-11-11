---
date: 2026-11-10
tags:
  - plugin
  - yarp
  - aspnet
---
yarp是微软官方直接构建在 ASP.NET Core管道之上的反向代理库——只要你熟悉 ASP.NET Core 中间件，上手 YARP 会非常快。国内有很多优质的开源实战参考项目都是通过yarp实现的。

FastGithub / Watt Toolkit (原 Steam++)是国内比较知名的开源加速工具，底层使用的就是YARP+kestrel实现的本地反向代理与证书劫持，非常适合学习yarp在桌面端/本地代理中的高级玩法。

yarp官方的github的sample目录有很多练手代码，官方给了我们十几个独立的，可以直接运行的最小化demo项目，非常推荐直接 git clone 下来边跑边改：：https://github.com/dotnet/yarp/tree/main/samples

• BasicYarpSample：最基础的 appsettings.json 路由转发。
• ReverseProxy.Code.Sample：纯 C# 代码内存动态配置（不依赖 JSON 文件）。
• ReverseProxy.Transforms.Sample：如何修改请求头、重写路径、注入自定义 Header。
• ReverseProxy.Auth.Sample：在网关层统一做 JWT / Cookie 鉴权，再把用户信息透传给下游服务。
• ReverseProxy.Direct.Sample：使用 IHttpForwarder 在普通 Controller / Minimal API 中按需手动转发请求。

## 功能

概述请看：[YARP 概述 | Microsoft Learn](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/servers/yarp/yarp-overview?view=aspnetcore-10.0) 。YARP可以做到的是：

- 路由：根据预定义规则（例如 URL 模式或请求标头）将请求定向到不同的后端服务器。 例如，`/images`、`/api` 和 `/db` 请求可以被路由到映像、API 和数据库服务器。
- 负载均衡：跨多个后端服务器分配传入流量，以防止特定服务器过载。 分布提高了性能和可靠性。
- 可伸缩性：通过跨多个服务器分配流量，反向代理有助于缩放应用来处理更多用户和更高的负载。 后端服务器缩放（添加或删除），不影响客户端。
- [SSL/TLS 终止](https://learn.microsoft.com/zh-cn/azure/application-gateway/ssl-overview)：从后端服务器卸载 TLS 加密和解密进程，从而减少其工作负荷。
- 连接抽象、分离和控制 URL 空间：来自外部客户端的入站请求和来自后端的出站响应是独立的。 这种独立性允许存在不同的：
    - HTTP、ie、HTTP/1.1、HTTP/2、HTTP/3 的版本。 代理可以升级或降级 HTTP 版本。
    - 连接生存期，支持在后端建立长期连接，同时保持短客户端连接。
    - 控制 URL 空间：在转发到后端之前，可以转换传入 URL。 这会将外部 URL 如何映射到内部服务的过程进行抽象化。 内部服务终结点可能会更改，而不会影响外部 URL。
- 安全性：内部服务终结点可以从外部泄露中隐藏，从而防范某些类型的网络攻击，例如 [DDoS 攻击](https://www.microsoft.com/security/business/security-101/what-is-a-ddos-attack?msockid=3e35ed3aa4666d8003aaf830a5006c74)。
- 缓存：可以缓存频繁请求的资源，以减少后端服务器上的负载并提高响应时间。
- 版本控制：可以使用不同的 URL 映射支持不同版本的 API。
- 简化维护：反向代理可以处理 [SSL/TLS 终止](https://learn.microsoft.com/zh-cn/azure/application-gateway/ssl-overview) 和其他任务，从而简化后端服务器的配置和维护。 例如，可以在反向代理级别而不是在每个单独的服务器上管理 SSL 证书和安全策略。

使用yarp反向代理是优于其他代理的，yarp提供了多种独特的优势，使它成为开发人员的有吸引力的选择：

- 自定义：YARP 高度可自定义，使开发人员能够以最少的努力根据特定需求定制代理。
- 与 .NET 集成：基于 ASP.NET Core 构建，YARP 与 .NET 生态系统无缝集成，使其成为 .NET 开发人员的理想选择。
- 扩展性：YARP 提供了一组丰富的扩展点，使开发人员能够根据需要使用熟悉的 C# 代码添加自定义逻辑和功能。
- 可伸缩性：直接转发扩展性选项使 YARP 能够缩放以支持大多数反向代理无法实现的域名和后端缩放。
- 积极开发：[YARP 由 Microsoft 积极维护和开发](https://github.com/dotnet/yarp)，确保其保持最新技术和最佳实践。
- 综合维护的文档：YARP 附带了广泛的文档和示例，便于入门和实现高级功能。
- [开放源代码](https://github.com/dotnet/yarp)。 YARP 和 YARP 文档是开放源代码。 欢迎[贡献](https://github.com/dotnet/yarp/blob/main/README.md)、评论和反馈。


