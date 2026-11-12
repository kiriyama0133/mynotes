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


## 加载代理配置

本文提供 YARP 的示例代理配置。 尽管配置示例使用 JSON，但任何 `IConfiguration` 源都应正常工作。 此外，当源文件更改时，配置会更新而不重启代理。


```cs
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

// Add the reverse proxy capability to the server
builder.Services.AddReverseProxy()
    // Initialize the reverse proxy from the "ReverseProxy" section of configuration
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

// Register the reverse proxy routes
app.MapReverseProxy();

app.Run();
```

当然可以可以多次加载不同的配置：

```cs
services.AddReverseProxy()
    .LoadFromConfig(Configuration.GetSection("ReverseProxy1"))
    .LoadFromConfig(Configuration.GetSection("ReverseProxy2"));
```

我们在配置里可以如下般配置：

```json
{
  "ReverseProxy": {
    "Routes": {
      "route1" : {
        "ClusterId": "cluster1",
        "Match": {
          "Path": "{**catch-all}",
          "Hosts" : [ "www.aaaaa.com", "www.bbbbb.com"]
        }
      }
    },
    "Clusters": {
      "cluster1": {
        "Destinations": {
          "cluster1/destination1": {
            "Address": "https://example.com/"
          }
        }
      }
    }
  }
}
```

每个路由至少需要以下的字段：
- `RouteId`：路由的唯一名称。
- `ClusterId`：节中 `Clusters` 条目的名称。
- `Match`：可以是`Hosts`数组或`Path`模式字符串。 可以使用 ASP.NET Core > 路由模板中的 [路由定义路径](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/routing?view=aspnetcore-10.0#route-templates)。
路由匹配基于优先级最高的特定路由，如 [URL 匹配](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/routing?view=aspnetcore-10.0#url-matching)中所述。 可以使用字段实现 `order` 显式排序，其中较低的值采用更高的优先级。

可以在每个路由条目上配置 [YARP 标头](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/servers/yarp/header-routing?view=aspnetcore-10.0)、[身份验证和授权](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/servers/yarp/authn-authz?view=aspnetcore-10.0)、[跨源请求（CORS）](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/servers/yarp/cors?view=aspnetcore-10.0)和其他基于路由的策略。 有关其他字段，请参阅 [RouteConfig](https://learn.microsoft.com/zh-cn/dotnet/api/yarp.reverseproxy.configuration.routeconfig) 参考。

该 `Clusters` 部分是命名群集的无序集合。 群集主要包含命名目标的集合及其地址，其中任何一个都被视为能够处理给定路由的请求。 代理根据路由和群集配置处理请求，以选择目标。

### HTTP客户端配置

每个 [群集](https://learn.microsoft.com/zh-cn/dotnet/api/yarp.reverseproxy.configuration.clusterconfig) 都有一个专用 [HttpMessageInvoker](https://learn.microsoft.com/zh-cn/dotnet/api/system.net.http.httpmessageinvoker) 实例，用于将请求转发到其 [目标](https://learn.microsoft.com/zh-cn/dotnet/api/yarp.reverseproxy.configuration.destinationconfig)。 配置是按每个群集定义的。 在 YARP 启动时，所有群集都会获得新的 `HttpMessageInvoker` 实例，但如果以后群集配置发生更改，则 [IForwarderHttpClientFactory](https://learn.microsoft.com/zh-cn/dotnet/api/yarp.reverseproxy.forwarder.iforwarderhttpclientfactory) 将重新运行并决定是否应创建新的 `HttpMessageInvoker` 或使用现有实例。 当 `IForwarderHttpClientFactory`发生更改时，默认 `HttpMessageInvoker` 实现会创建新的 [](https://learn.microsoft.com/zh-cn/dotnet/api/yarp.reverseproxy.configuration.httpclientconfig)。

HTTP 客户端配置基于 [HttpClientConfig](https://learn.microsoft.com/zh-cn/dotnet/api/yarp.reverseproxy.configuration.httpclientconfig)，并由以下配置架构表示。 如果你需要更精细的方法，请使用 的[](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/servers/yarp/http-client-config?view=aspnetcore-10.0#custom-iforwarderhttpclientfactory)`IForwarderHttpClientFactory`。

配置HttpClient可以让我们精细控制yarp作为客户端去请求后端目标服务的时候的网络行为，通过配置这些，我们可以做到连接池与并发控制；安全与TLS设置；调试和代理穿透；字符编码处理。

**`MaxConnectionsPerServer`** 能够限制对同一目标服务器的最大并发HTTP/1.1 连接数。默认是 `int32.MaxValue`。如果你的后端服务承受能力有限，或者你想防止突发流量打垮下游，可以调低这个值（例如设为 `10`）。[相关文档](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/yarp/http-client-config?view=aspnetcore-10.0)

**`nableMultipleHttp2Connections`**：当 HTTP/2 连接达到并发流上限时，允许开启额外的连接。默认 `true`，通常无需改动。

**`slProtocols`**：指定 YARP 连接后端时允许使用的 TLS 版本（如 `Tls12`, `Tls13`）。你可以借此强制使用高版本协议，禁用不安全的旧协议。

**`angerousAcceptAnyServerCertificate`**：设为 `true` 会**完全禁用**对后端服务器 SSL 证书的验证。官方命名里带 `Dangerous` 就是警告你，这通常只用于**本地开发或测试**（比如后端用了自签名证书），生产环境务必保持 `false`

**`WebProxy`**：让 YARP 的转发请求走一个出站代理。这在你需要用 **Fiddler** 等工具抓包调试 YARP 到后端的流量时非常有用

**`UseProxy`**：YARP 默认关闭了系统代理检测（`UseProxy = false`）以提升性能。如果你想临时借用系统代理，可以在代码里通过 `ConfigureHttpClient` 把它改回 `true`

### 限流策略

为特定路由配置限流是 YARP 作为 API 网关的核心能力之一。YARP 的限流功能直接构建在 [ASP.NET](https://asp.net/) Core 原生的限流中间件之上，因此你可以利用 [ASP.NET](https://asp.net/) Core 强大的限流策略，并精细地应用到每一条 YARP 路由上。

我们可以使用AddRateLimiter来注册一个或多个具名的限流策略，可以用固定窗口算法来限制每个窗口最多通过的请求数量：

```cs
// Program.cs
builder.Services.AddRateLimiter(options =>
{
    // 定义一个名为 "customPolicy" 的限流策略
    options.AddFixedWindowLimiter("customPolicy", opt =>
    {
        opt.PermitLimit = 4; // 时间窗口内允许的最大请求数
        opt.Window = TimeSpan.FromSeconds(12); // 时间窗口长度
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst; // 队列处理顺序
        opt.QueueLimit = 2; // 当请求超限时，可以进入队列排队的最大数量
    });
});
```

注册之后，别忘了在中间件管道中启用它。**`UseRateLimiter()` 必须在 `MapReverseProxy()` 之前调用**:：

```cs
// Program.cs
var app = builder.Build();

// ...
app.UseRateLimiter(); // 启用限流中间件
app.MapReverseProxy(); // 映射反向代理路由

app.Run();
```

以及代理的配置这里要写上使用的策略：

```json
  "ReverseProxy": {
    "Routes": {
      "weather-route" : {
        "ClusterId": "cluster1",
        "RateLimiterPolicy": "policy",
        "Match": {
          "Path": "/weather/{**catch-all}"
        },
        "Transforms": [
          { "PathSet": "/weatherforecast" }
        ]
      }
    },
```

### 标头路由

标头；路由配置方式很灵活，可以精确地将带有不同 Header 的请求分发到不同的后端集群。头路由的规则定义在路由的 `Match` 节里。一个路由可以配置一个或多个标头规则，**如果配置了多个，那么请求必须同时满足所有规则，才能匹配成功**。


```json
{
  "ReverseProxy": {
    "Routes": {
      "header-route": {
        "ClusterId": "cluster1",
        "Match": {
          "Path": "/{**catch-all}",
          "Headers": [
            {
              "Name": "X-Tenant-Id",
              "Values": [ "tenant-a" ],
              "Mode": "ExactHeader"
            }
          ]
        }
      }
    }
  }
}
```

- **`Name`**：请求头的名称。
- **`Values`**：需要匹配的值列表。如果提供多个值，只要匹配其中**任意一个**即可（此为 OR 逻辑）。
- **`Mode`**：匹配模式，决定了值如何比较。

匹配模式有好几种选择：

- **`ExactHeader`**：请求头的值必须与 `Values` 中的某个值**完全相等**（默认模式）。
- **`HeaderPrefix`**：请求头的值必须以 `Values` 中的某个值**开头**。
- **`Exists`**：**只要请求头存在即可**，不关心具体的值（此时可忽略 `Values`）。
- **`Contains`**：请求头的值必须**包含** `Values` 中的某个子串。
- **`NotContains`**：请求头的值**不能包含** `Values` 中的任何子串。
- **`NotExists`**：请求头**必须不存在**。

比如我们配置个要求传入X-Weather-Id才会转发的策略：

```json
"weather-route" : {
"ClusterId": "cluster1",
"RateLimiterPolicy": "policy",
"Match": {
  "Path": "/weather/{**catch-all}",
  "Headers": [
	{
	  "Name": "X-Weather-Id",
	  "Values": [ "sunday" ],
	  "Mode": "ExactHeader"
	}
  ]
},
```

结果如下：

```shell
(berry) kiriyama@kiriyama:/media/kiriyama/新加卷/mujoco/berry$ curl -k -i -H "X-Weather-Id: sunday" https://localhost:7293/weather
HTTP/2 200 
content-type: application/json; charset=utf-8
date: Thu, 12 Nov 2026 07:48:37 GMT
server: Kestrel

[{"date":"2026-11-13","temperatureC":46,"summary":"Balmy","temperatureF":114},{"date":"2026-11-14","temperatureC":27,"summary":"Scorching","temperatureF":80},{"date":"2026-11-15","temperatureC":51,"summary":"Cool","temperatureF":123},{"date":"2026-11-16","temperatureC":7,"summary":"Sweltering","temperatureF":44},{"date":"2026-11-17","temperatureC":-1,"summary":"Scorching","temperatureF":31}](berry(berry) kiriyama@kiriyama:/media/kiriyama/新加卷/mujoco/berry$ curl -k -i -H "X-Weather-Id: monday" https://localhost:7293/weather
HTTP/2 404 
date: Thu, 12 Nov 2026 07:48:41 GMT
server: Kestrel
content-length: 0
```

### 查询参数路由

查询参数路由和标头路由非常相似，都是在 `Match` 节点中定义，用请求 URL 中 `?` 后面的参数来决定流量的去向。询参数路由通过 `QueryParameters` 数组定义。**多条规则之间是“且（AND）”的关系**，必须同时满足，路由才会匹配。

配置支持的 `Mode` 有以下几种：

- **`Exact`**：参数值必须**完全等于** `Values` 中的某一个（默认模式）。
- **`Prefix`**：参数值必须以 `Values` 中的某一个**开头**。
- **`Exists`**：**只要参数存在**即可，不关心具体值（此时可省略 `Values`）。
- **`Contains`**：参数值必须**包含** `Values` 中的某个子串。
- **`NotContains`**：参数值**不能包含** `Values` 中的任何子串。

YARP 默认的路由匹配优先级顺序是：**Path → Method → Host → Headers → Query Parameters**。

这意味着，一个指定了 `Method` 但没有查询参数的路由，会**优先**于一个指定了查询参数但没有 `Method` 的路由被匹配。你可以通过设置路由的 **`Order`** 属性（数值越小优先级越高）来覆盖这个默认行为。

```json
{
  "ReverseProxy": {
    "Routes": {
      "sunday-query-route": {
        "ClusterId": "cluster1",
        "Order": 1,
        "Match": {
          "Path": "/weather",
          "QueryParameters": [
            { "Name": "day", "Values": [ "sunday" ], "Mode": "Exact" }
          ]
        },
        "Transforms": [ { "PathSet": "/weatherforecast" } ]
      },
      "default-query-route": {
        "ClusterId": "cluster1",
        "Order": 2,
        "Match": { "Path": "/weather" },
        "Transforms": [ { "PathSet": "/weatherforecast" } ]
      }
    },
    "Clusters": {
      "cluster1": {
        "Destinations": { "destination1": { "Address": "https://localhost:7293/" } },
        "HttpClient": { "DangerousAcceptAnyServerCertificate": true }
      }
    }
  }
}
```


```shell
# 测试 A：携带查询参数 ?day=sunday
curl -k -i "https://localhost:7293/weather?day=sunday"

# 测试 B：携带错误的参数值 ?day=monday
curl -k -i "https://localhost:7293/weather?day=monday"

# 测试 C：不携带任何查询参数
curl -k -i "https://localhost:7293/weather"

- **测试 A** 应命中 `sunday-query-route`（日志打印 `Request matched route 'sunday-query-route'`）。
    
- **测试 B 和 C** 应命中 `default-query-route`（日志打印 `Request matched route 'default-query-route'`）。
```

### 负载均衡策略

每当有多个可用的健康目标时，YARP 必须决定哪一个用于给定请求。 YARP 附带内置负载均衡算法，但也为任何自定义负载均衡方法提供扩展性。

负载均衡策略通过 `AddLoadBalancingPolicies()` 方法在 DI 容器中注册，由 `AddReverseProxy()`自动调用。

中间件随 `UseLoadBalancing()`一起添加，该中间件默认包含在无参数 `MapReverseProxy` 方法中。载均衡策略配置在 **`Clusters`** 节点下，而不是 `Routes` 里。

```json
{
  "ReverseProxy": {
    "Clusters": {
      "cluster1": {
        "LoadBalancingPolicy": "RoundRobin",
        "Destinations": {
          "destination1": { "Address": "https://localhost:7293/" },
          "destination2": { "Address": "https://localhost:7294/" }
        }
      }
    }
  }
}
```

如果不显式指定，YARP 会使用默认的 **`PowerOfTwoChoices`** 策略。
ARP 内置了以下策略，直接在配置中填写名称即可：

**`FirstAlphabetical`**：选择可用列表中按字母顺序排在最前的目标，完全不考虑当前负载。适合只有主备两个节点、需要明确故障转移顺序的场景。

**`Random`**：完全随机选择。实现简单，但在节点性能或负载不均时，可能把请求分给已经很忙的节点。

**`RoundRobin`**：按顺序循环分发。在节点性能一致、请求耗时相近的场景下最直观，也最容易验证。

**`LeastRequests`**：选择当前已分配请求数最少的目标。适合请求耗时差异较大的场景，但需要检查所有可用节点，开销相对更高。

`ILoadBalancingPolicy` 负责从可用健康目标列表中选择目标。

可以在 DI 中提供自定义实现。

```cs
// Implement the ILoadBalancingPolicy
public sealed class LastLoadBalancingPolicy : ILoadBalancingPolicy
{
    public string Name => "Last";

    public DestinationState? PickDestination(HttpContext context, ClusterState cluster, IReadOnlyList<DestinationState> availableDestinations)
    {
        return availableDestinations[^1];
    }
}

// Register it in DI in ConfigureServices method
services.AddSingleton<ILoadBalancingPolicy, LastLoadBalancingPolicy>();

// Set the LoadBalancingPolicy on the cluster
cluster.LoadBalancingPolicy = "Last";
```

### 会话亲和性

会话亲和性是一种机制，用于将有因果关系的请求序列绑定到在多个目标之间均衡负载时处理第一个请求的目标。 在序列中的大多数请求处理相同数据以及处理请求的不同节点（目标）的数据访问成本不同的情况下，它很有用。 最常见的示例是暂时性缓存（例如内存中），其中第一个请求将数据从较慢的永久性存储提取到快速本地缓存中，而其他请求则只处理缓存的数据，从而增加吞吐量。

这个也叫做粘性会话，当负载均衡器收到某个用户的请求的时候它会想办法把后续的请求都粘在同一个后端服务器上。



