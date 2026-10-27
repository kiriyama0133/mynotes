---
date: 2026-10-09
tags:
  - network
  - socketio
  - nodejs
---
[Socket.IO](https://socket.io/) 是一个基于事件驱动的实时双向通信库，它的核心价值在于**封装了 WebSocket 的复杂性**，并提供了**自动降级、自动重连、房间（Room）分组**等开箱即用的高级功能。

[Socket.IO](https://socket.io/) 的使用需要服务端和客户端配合。

在Nodejs服务端上，你需要将SocketIO实例挂载到一个HTTP服务器上：

```ts
// 使用 ES Modules 语法
import express from 'express';
import { createServer } from 'node:http';
import { Server } from 'socket.io';

const app = express();
const server = createServer(app);
// 初始化 Socket.IO，并传入 HTTP server 实例
const io = new Server(server);

// 监听客户端连接
io.on('connection', (socket) => {
  console.log('一个用户已连接，ID:', socket.id);

  // 监听客户端发来的自定义事件 'chat message'
  socket.on('chat message', (msg) => {
    console.log('收到消息:', msg);
    // 将消息广播给所有已连接的客户端
    io.emit('chat message', msg);
  });
  
  // 监听客户端断开连接
  socket.on('disconnect', () => {
    console.log('用户已断开:', socket.id);
  });
});

server.listen(3000, () => {
  console.log('服务端运行在 http://localhost:3000');
});
```

在客户端也就是浏览器上，也需要引入socket.io-client库来创建连接：

```ts
<!-- 引入客户端库（由服务端自动提供） -->
<script src="/socket.io/socket.io.js"></script>
<script>
  // 连接到服务器，不填 URL 则默认连接当前域名
  const socket = io();

  socket.on('connect', () => {
    console.log('已连接到服务器');
    // 向服务器发送 'chat message' 事件
    socket.emit('chat message', '你好，服务端！');
  });

  // 监听服务端广播的 'chat message' 事件
  socket.on('chat message', (msg) => {
    console.log('收到消息:', msg);
  });
</script>
```

在vue/react等框架中，首先通过 npm 安装 `socket.io-client`，然后在组件中引入并使用。关键是在组件的**生命周期钩子**中建立连接，并在**组件销毁前断开连接**，以避免内存泄漏。


```ts
// Vue 3 示例 (Composition API)
import { onMounted, onUnmounted } from 'vue';
import { io } from 'socket.io-client';

let socket;

onMounted(() => {
  socket = io('http://localhost:3000');
  socket.on('connect', () => {
    console.log('Vue 组件已连接');
  });
  socket.on('chat message', (msg) => {
    console.log('收到消息:', msg);
  });
});

onUnmounted(() => {
  if (socket) {
    socket.disconnect();
    console.log('Vue 组件已断开');
  }
});
```

[Socket.IO](https://socket.io/) 的强大之处在于其封装的高级特性，核心是**事件**、**房间**和**命名空间**。

**事件驱动通信 (`emit` 与 `on`)**  
这是 [Socket.IO](https://socket.io/) 最基础的通信方式，类似订阅-发布模式。

- `socket.emit('eventName', data)`：**发送**事件。
    
- `socket.on('eventName', callback)`：**监听**事件。

**房间 (Rooms) 与命名空间 (Namespaces)**  
这是 [Socket.IO](https://socket.io/) 用于**组织和管理连接**的两个核心工具，用于向特定用户群组精确发送消息。[](https://socket.io/es/how-to/emit-to-a-list-of-sockets)

|特性|房间 (Room)|命名空间 (Namespace)|
|---|---|---|
|**概念**|在**一个命名空间内**的任意分组。|在**服务器层面**的**逻辑分离**，相当于不同的通信通道。|
|**创建方式**|动态创建，客户端或服务端均可让 socket 加入/离开（`socket.join('roomId')`）。|在服务端通过 `io.of('/namespace')` 显式创建，客户端连接时指定（`io('/namespace')`）。|
|**典型场景**|聊天室成员、游戏房间、文档的协作者。|将应用分隔为 `/admin`、`/chat`、`/notifications` 等模块。|
|**核心优势**|**动态、灵活**，非常适合基于业务逻辑（如用户ID、项目ID）进行分组广播。[](https://socket.io/es/how-to/emit-to-a-list-of-sockets)|**严格的隔离**，不同命名空间之间的连接和事件完全独立，可分别设置中间件。|
.NET 生态中有成熟的 [Socket.IO](https://socket.io/) 解决方案，虽然官方（[Socket.IO](https://socket.io/) 团队）并未提供原生支持，但社区贡献了许多高质量的 NuGet 包，涵盖了客户端和服务端的实现，在服务端或桌面端（如 WPF、控制台）连接 [Socket.IO](https://socket.io/) 服务时，主要有以下几个选择：

**`SocketIOClient`** 是目前为止下载量最高的，功能最全面的，支持 `EmitAsync`、Ack 回调、二进制传输、自定义序列化（默认 `System.Text.Json`）等。适合需要高兼容性、功能完备的通用场景。

**`H.Socket.IO`** 主打高性能和轻量级，API 设计简洁，支持命名空间（Namespaces），完全异步。适合追求极致性能、代码简洁，且不需要复杂 Ack 机制的轻量级应用。

.NET 客户端库的主要用途是**连接官方（Node.js）的 [Socket.IO](https://socket.io/) 服务端**。虽然大多数库在协议层面兼容，但版本迭代时仍可能遇到细微的兼容性问题。

>最典型的例子是：如果你的服务端使用 [Socket.IO](https://socket.io/) **v4**，应优先选择明确声明支持该版本的客户端库（如 `SocketIOClient` 或 `H.Socket.IO`），并留意其文档中对 `EIO` 协议版本（[Engine.IO](https://engine.io/) V3/V4）的配置说明。

对于ASP.NET开发者来说，有相应的服务端使用的SOCKETIO库，比如说**SocketIOSharp2** ，基于 **[Socket.IO](https://socket.io/) 协议 v4** ([Engine.IO](https://engine.io/) v4)，是对原 `SocketIOSharp` 的活跃分支。**关键优势是支持连接预处理（`OnConnecting`）**，可在握手阶段进行Token验证，非常适合需要鉴权的生产环境。

**CORE.HELPER** 提供 `SocketIOServer` 类，API 设计简洁，支持连接、事件订阅等基本功能，适合快速原型验证或简单场景。

以 `SocketIOSharp2` 为例，演示如何将其嵌入 [ASP.NET](https://asp.net/) Core 应用。关键是将 [Socket.IO](https://socket.io/) 服务作为一个 **Hosted Service（后台服务）** 启动。

需要先创建socket.io的后台服务：

```cs
using SocketIOSharp.Server;
using SocketIOSharp.Server.Client;
using SocketIOSharp.Common;

public class SocketIoServerService : BackgroundService
{
    private readonly ILogger<SocketIoServerService> _logger;
    private SocketIOServer? _server;

    public SocketIoServerService(ILogger<SocketIoServerService> logger)
    {
        _logger = logger;
    }

    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // 1. 配置监听端口（可与 ASP.NET Core 的 HTTP 端口不同，这里用 9092）
        var option = new SocketIOServerOption(port: 9092);
        _server = new SocketIOServer(option);

        // 2. 连接预处理（可选，用于鉴权）
        _server.OnConnecting(context =>
        {
            // context.Item1 包含查询参数，可从中提取 token
            var token = context.Item1["token"];
            bool isValid = !string.IsNullOrWhiteSpace(token);
            _server.AddVerificationResult(context.Item2, isValid);
        });

        // 3. 监听客户端连接
        _server.OnConnection(socket =>
        {
            _logger.LogInformation("Client connected: {SocketId}", socket.Id);

            // 监听客户端发来的 'chat message' 事件
            socket.On("chat message", data =>
            {
                _logger.LogInformation("Received message: {Data}", data?.ToString());
                
                // 向所有客户端广播（这里简化处理，实际应遍历所有连接的 socket）
                socket.Emit("chat message", $"Echo: {data}");
            });

            // 监听断开连接事件
            socket.On(SocketIOEvent.DISCONNECT, () =>
            {
                _logger.LogInformation("Client disconnected: {SocketId}", socket.Id);
            });
        });

        // 4. 启动 Socket.IO 服务
        _server.Start();
        _logger.LogInformation("Socket.IO server started on port 9092");

        // 保持服务运行，直到应用关闭
        stoppingToken.WaitHandle.WaitOne();

        // 5. 关闭时清理资源
        _server.Stop();
        _server.Dispose();
        return Task.CompletedTask;
    }
}
```

然后在Program.cs中注册和启动服务：

```cs
var builder = WebApplication.CreateBuilder(args);

// 注册 Socket.IO 后台服务
builder.Services.AddHostedService<SocketIoServerService>();

var app = builder.Build();

// 你的其他 ASP.NET Core 中间件（如 MapControllers 等）...

app.Run();
```

前端就使用Socket.IO客户端库，需要连接到上一步配置的端口：

```ts
// 假设 Socket.IO 服务运行在 localhost:9092
const socket = io('http://localhost:9092', {
    query: { token: 'your_auth_token' } // 若配置了 OnConnecting 鉴权
});

socket.on('connect', () => {
    console.log('Connected to Socket.IO server');
    socket.emit('chat message', 'Hello from Web Client!');
});

socket.on('chat message', (msg) => {
    console.log('Received:', msg);
});
```

[Socket.IO](https://socket.io/) 服务端需要**独立监听一个端口**（如上述 `9092`），它与 [ASP.NET](https://asp.net/) Core 的 HTTP/Kestrel 管道是**并行运行**的。你无法直接通过 [ASP.NET](https://asp.net/) Core 的 `/socket.io` 路径来访问它。务必确认你选择的库所支持的 **[Engine.IO](https://engine.io/) 协议版本**（`SocketIOSharp2` 支持 v4）与前端 [Socket.IO](https://socket.io/) 客户端版本匹配，这是连接成功的关键。如果你没有**必须使用 [Socket.IO](https://socket.io/) 客户端**的硬性要求，在纯 .NET 技术栈中，官方推荐的 **[ASP.NET](https://asp.net/) Core SignalR** 是更原生的选择，它深度集成于 [ASP.NET](https://asp.net/) Core 管道，无需额外端口和第三方库。

