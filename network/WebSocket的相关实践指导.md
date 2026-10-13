---
date: 2026-12-12
tags:
  - network
  - websocket
  - signalr
  - vueuse
---
这个笔记用于指导ws协议的相关实践，本身是对tcp协议里大致提了ws一些的补充，这里可以链接过去[[TCP协议]]

## SignalR

**SignalR 是微软推出的一个实时通信库**，核心目标是让服务器能**主动推送**消息给客户端，而不需要客户端反复轮询。客户端和服务器协商后，自动选择可用的最高优先级方式，开发者不需要关心底层细节。

<svg viewBox="0 0 680 378" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>SignalR 与 useWebSocket 的抽象层次对比</title>
<desc>SignalR 覆盖从业务代码到服务端 Hub 的完整五层，useWebSocket 只有三层且不提供服务端、group 与广播能力。</desc>
<rect x="40" y="32" width="600" height="326" rx="20" fill="#F7F6F3" stroke="#D3D1C7" stroke-width="0.5"/>
<text x="60" y="58" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="15" font-weight="500" fill="#2C2C2A">SignalR vs useWebSocket：各自覆盖到哪一层</text>
<text x="60" y="78" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#5F5E5A">左边是一整套端到端框架，右边只是一层浏览器端封装</text>

<text x="60" y="106" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#0C447C">SignalR</text>
<text x="130" y="106" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="11" fill="#185FA5">端到端框架 · 含服务端</text>
<text x="376" y="106" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#085041">useWebSocket</text>
<text x="466" y="106" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="11" fill="#0F6E56">客户端 composable</text>

<rect x="60" y="118" width="280" height="38" rx="8" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
<text x="76" y="133" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#042C53">你的业务代码</text>
<text x="76" y="148" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#185FA5">hub.invoke("SendMessage", msg)</text>

<rect x="60" y="164" width="280" height="38" rx="8" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
<text x="76" y="179" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#042C53">Hub 抽象</text>
<text x="76" y="194" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#185FA5">invoke / on / group · 跨两端</text>

<rect x="60" y="210" width="280" height="38" rx="8" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
<text x="76" y="225" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#042C53">Hub Protocol</text>
<text x="76" y="240" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#185FA5">0x1E 分隔的 JSON 信封</text>

<rect x="60" y="256" width="280" height="38" rx="8" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
<text x="76" y="271" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#042C53">传输协商</text>
<text x="76" y="286" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#185FA5">WS → SSE → Long Polling 择优</text>

<rect x="60" y="302" width="280" height="38" rx="8" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
<text x="76" y="317" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#042C53">原生 WebSocket</text>
<text x="76" y="332" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#185FA5">浏览器提供，协议层</text>

<rect x="376" y="118" width="264" height="38" rx="8" fill="#E1F5EE" stroke="#0F6E56" stroke-width="0.5"/>
<text x="392" y="133" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#04342C">你的业务代码</text>
<text x="392" y="148" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#0F6E56">自己定信封 {"type":"chat"}</text>

<rect x="376" y="164" width="264" height="38" rx="8" fill="#E1F5EE" stroke="#0F6E56" stroke-width="0.5"/>
<text x="392" y="179" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#04342C">useWebSocket</text>
<text x="392" y="194" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#0F6E56">重连 · 心跳 · 响应式</text>

<rect x="376" y="210" width="264" height="38" rx="8" fill="#E1F5EE" stroke="#0F6E56" stroke-width="0.5"/>
<text x="392" y="225" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#04342C">原生 WebSocket</text>
<text x="392" y="240" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#0F6E56">只有这一种传输，不降级</text>

<text x="376" y="272" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#993C1D">服务端、group、广播都得你自己写</text>
<text x="376" y="292" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#993C1D">它到浏览器为止，不跨这条线</text>
</svg>


对于多标签共享一条，也就是SharedWorker，SignalR是唯一现成的库。

```ts
// workers/chat.shared.ts —— 用 dist/webworker/signalr.js，打包器会按 worker 条件解析
const conn = new HubConnectionBuilder().withUrl('/hubs/chat').withAutomaticReconnect().build()
await conn.start()

const ports: MessagePort[] = []

self.onconnect = (e: MessageEvent) => {
  const port = e.ports[0]
  ports.push(port)
  port.onmessage = ({ data }) => {
    if (data.type === 'send') conn.send('SendMessage', data.user, data.text)
  }
  port.start()
}

// 一条连接扇出给所有标签页
conn.on('ReceiveMessage', (user: string, text: string) => {
  for (const p of ports) p.postMessage({ type: 'ReceiveMessage', user, text })
})

```


后端部分可能长这样：

```cs
// Hubs/ChatHub.cs —— 一个类就是一组可被客户端调用的方法
public sealed class ChatHub : Hub
{
    // 客户端 conn.invoke('SendMessage', user, text) 落到这里
    public Task SendMessage(string user, string text)
        => Clients.All.SendAsync("ReceiveMessage", user, text);

    public Task<string> Ping() => Task.FromResult("pong");   // 有返回值，客户端 await 得到 "pong"

    // group 是白送的，不用自己维护连接表
    public Task JoinRoom(string room)
        => Groups.AddToGroupAsync(Context.ConnectionId, $"room:{room}");

    public Task SendToRoom(string room, string text)
        => Clients.Group($"room:{room}").SendAsync("ReceiveMessage", "room", text);
}

// Program.cs
builder.Services.AddSignalR(o =>
{
    o.KeepAliveInterval     = TimeSpan.FromSeconds(15); // 默认值，服务端发 {"type":6} ping
    o.ClientTimeoutInterval = TimeSpan.FromSeconds(30); // 默认 2x，30s 内无数据判死
    o.MaximumReceiveMessageSize = 32 * 1024;            // 单条上限 32KB，超了直接断
});
app.MapHub<ChatHub>("/hubs/chat");


```

我们在前端的代码就长这样，你可以在composables里完成：

```ts
import { HubConnectionBuilder, HubConnectionState, LogLevel } from '@microsoft/signalr'

let conn: HubConnection | null = null   // 模块级单例，别在组件里各建一条

export function useChatHub() {
  const status = ref(HubConnectionState.Disconnected)
  const messages = shallowRef<ChatMessage[]>([])

  async function connect() {
    if (conn) return conn

    conn = new HubConnectionBuilder()
      .withUrl('/hubs/chat')                 // 第一跳是 POST /hubs/chat/negotiate，不是 WS
      .withAutomaticReconnect()              // 源码默认退避 [0, 2s, 10s, 30s]，之后放弃
      .configureLogging(LogLevel.Warning)
      .build()

    // 服务端 -> 客户端：这个 key 必须和服务端 SendAsync("ReceiveMessage") 的字符串完全一致
    conn.on('ReceiveMessage', (user: string, text: string) => {
      messages.value = [...messages.value, { user, text }]
    })

    conn.onreconnecting(e => console.warn('reconnecting', e?.message))
    conn.onreconnected(async () => {
      // 重连后 connectionId 变了，之前 join 的 group 全部丢失，必须重新 join
      await conn!.invoke('JoinRoom', 'lobby')
    })
    conn.onclose(e => { status.value = HubConnectionState.Disconnected })

    await conn.start()
    status.value = conn.state
    await conn.invoke('JoinRoom', 'lobby')
    return conn
  }

  // invoke 等返回值（RPC 语义）；send 发完就走，少一个 RTT 的等待
  const sendMessage = (user: string, text: string) =>
    conn?.state === HubConnectionState.Connected
      ? conn.send('SendMessage', user, text)
      : Promise.resolve()

  // 服务端 IAsyncEnumerable<T> / ChannelReader<T> 持续推，客户端订阅
  function subscribeCounter() {
    const sub = conn!.stream<number>('Counter', 10)
    sub.subscribe({ next: n => console.log(n), complete: () => {}, error: e => console.error(e) })
    return () => sub.dispose()
  }

  onScopeDispose(() => { conn?.stop(); conn = null })
  return { status, messages, connect, sendMessage, subscribeCounter }
}

```

SignalR 官方带 `dist/webworker/signalr.js`，源码里的 `document` 都写成 `window.document` 且有 `Platform.isBrowser` 守卫，在没有 `window` 的 Worker 里不会炸。**代价**：`Platform.isBrowser` 为 false 后不再监听窗口 `freeze` 事件，你会在日志里少一条"标签页被冻结导致断连"的线索。

和vueuse相比，signalr的优势在于 hub rpc、多传输降级、group、backplane这些在服务端侧，vueuse一个都不提供，也都是SignalR的立身之本。

hub可以叫做集线器，是服务端的核心类，负责连接管理和消息分发。backplane是signalr用于多服务器横向扩展的核心组件，当你应用部署在多台服务器上的时候，它负责让所有服务器之间能够互相转发消息，确保连接到不同服务器的客户端也能收到广播。group是signalr中用于管理链接分组的核心机制，可以让你只向订阅了某类消息的客户端推送，而不是广播给所有人。

### Hub RPC

它不是"透明地把本地方法调用传过去"，而是**方法名当字符串传、服务端反射查表找回来**。这一句话解释了它：

- 服务端查表用的是 `StringComparer.OrdinalIgnoreCase`，所以 **`invoke('SendMessage')` 和 `invoke('sendMessage')` 都能命中**；但客户端 `conn.on('ReceiveMessage')` 是 JS 对象键，**精确匹配**。这个不对称是最常踩的坑。
- 方法名既然是字符串，重构就没有编译期保护——这才有了**强类型 Hub**：

```cs
public interface IChatClient { Task ReceiveMessage(string user, string text); }

public class ChatHub : Hub<IChatClient>          // 泛型参数一加，字符串魔法全消失
{
    public Task SendMessage(string user, string text)
        => Clients.All.ReceiveMessage(user, text);   // 改名不再静默失效

    [HubMethodName("say")]                        // 对客户端暴露成 'say'
    public Task Say(string text) => Clients.All.ReceiveMessage("me", text);

    public Task<string> Ping() => Task.FromResult("pong");   // 客户端 await 拿得到

    // 反向调用客户端并等结果 —— 客户端也要写「有返回值」的 handler
    public Task<string> AskClient(string id) => Clients.Client(id).InvokeAsync<string>("GetInput");
}
```

- **Hub 是 transient 的**。官方文档原文：_"Hubs are transient: Don't store state in a property of the hub class. Each hub method call is executed on a new hub instance."_ 所以下面这段是经典的错：


```cs
public class BadHub : Hub { private int _count; }   // 每次调用都是新实例，永远是 0

// 对的：连接级状态放 Context.Items（同一连接跨多次调用保留）
public class GoodHub : Hub {
    public Task Add() {
        var n = (int)(Context.Items["count"] ?? 0) + 1;
        Context.Items["count"] = n;
        return Clients.Caller.SendAsync("Count", n);
    }
}

```

**Hub 方法参数会先从 DI 解析**（.NET 7 起的破坏性变更，`DisableImplicitFromServicesParameters` 可关），所以 `SendMessage(string a, string b, AppDbContext db)` 里的 `db` 是从容器来的，不是客户端传的；**流式**则是 `IAsyncEnumerable<T>` 对客户端的 `conn.stream<T>()`。

### 多传输降级

这个能力的前提是：SignalR 的"连接"是**逻辑概念**，不绑定任何一种 HTTP 形态：

<svg viewBox="0 0 680 324" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>SignalR 三种传输的形态与降级顺序</title>
<desc>客户端按 negotiate 返回的顺序依次尝试 WebSockets、ServerSentEvents、LongPolling，三种传输的 HTTP 形态各不相同。</desc>
<rect x="40" y="24" width="600" height="280" rx="20" fill="#F7F6F3" stroke="#D3D1C7" stroke-width="0.5"/>
<text x="60" y="50" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="15" font-weight="500" fill="#2C2C2A">同一个 Hub 连接，底下的 HTTP 形态可以完全不同</text>
<text x="60" y="70" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#5F5E5A">优先级由客户端决定，服务端只提供 availableTransports 候选列表</text>

<rect x="60" y="86" width="560" height="56" rx="10" fill="#E1F5EE" stroke="#0F6E56" stroke-width="0.5"/>
<text x="80" y="118" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#04342C">1 · WebSockets</text>
<text x="240" y="112" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#0F6E56">升级成一条真正双向的长连接，原生帧</text>
<text x="240" y="130" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#0F6E56">首选，几乎总是胜出</text>

<rect x="60" y="152" width="560" height="56" rx="10" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
<text x="80" y="184" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#042C53">2 · ServerSentEvents</text>
<text x="240" y="178" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#185FA5">下行是一条永不结束的响应体，上行另开 POST</text>
<text x="240" y="196" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#185FA5">只能发文本，浏览器对每域名有连接数限制</text>

<rect x="60" y="218" width="560" height="56" rx="10" fill="#FAEEDA" stroke="#BA7517" stroke-width="0.5"/>
<text x="80" y="250" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#412402">3 · LongPolling</text>
<text x="240" y="244" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#854F0B">请求被服务端挂住，有数据就返回，返回后立刻再发一个</text>
<text x="240" y="262" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#854F0B">兼容性兜底：每个响应结束就是一次「假连接」</text>

<text x="60" y="292" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#993C1D">三者形态差别这么大，所以连接必须能被拆成多个 HTTP 请求 —— 这也是它需要 sticky 的原因</text>
</svg>

**多传输降级的机制**其实是"先在应用层假装一条连接，再挑具体怎么实现"。

```json
{
  "negotiateVersion": 1,
  "connectionToken": "…",         ← 后续请求都带着它，服务端据此找回这条逻辑连接
  "availableTransports": [
    { "transport": "WebSockets",      "transferFormats": ["Text","Binary"] },
    { "transport": "ServerSentEvents","transferFormats": ["Text"] },
    { "transport": "LongPolling",     "transferFormats": ["Text","Binary"] }
  ]
}

```

客户端按这个列表**从上往下试**，谁先通用谁。几个由此推出的结论：

- **SSE 只能 Text**，所以二进制协议（MessagePack）会自动把 SSE 排除掉。
- LongPolling 是"每个响应就是一次假连接"​：服务端把请求挂住（默认 `PollTimeout` 90 秒），有数据就吐出来，客户端拿到后立刻再发一个。所以一条逻辑连接会变成成百上千个 HTTP 请求。
- 正因为这个，**negotiate 拿到的 token 必须和后续每次请求落到同一台服务器**——这就是 sticky sessions 是硬要求的原因，不是可选项。**​即使挂了 backplane 也仍然需要**（backplane 只解决跨机广播，不解决路由）。
- 有个反直觉的代价：**WS 成功时那次 negotiate 是白花的一个 RTT**。所以才有 `skipNegotiation: true`，但它**只在锁定 `HttpTransportType.WebSockets` 时可用**——你没法"跳过协商再降级"。

### Group

它和"用户"是**两种交叉的聚合维度**：

<svg viewBox="0 0 680 360" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>SignalR 推送目标选择器的命中范围</title>
<desc>五列代表五个连接，实心表示被该选择器命中。可以看到 group 是一个任意连接集合，user 则会聚合同一用户的多个连接。</desc>
<rect x="40" y="24" width="600" height="316" rx="20" fill="#F7F6F3" stroke="#D3D1C7" stroke-width="0.5"/>
<text x="60" y="50" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="15" font-weight="500" fill="#2C2C2A">Clients 的每种写法，各自会命中哪些连接</text>
<text x="60" y="70" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#5F5E5A">实心 = 被命中。u1 开了两个标签页，所以是两个独立连接</text>

<text x="250" y="94" text-anchor="middle" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#2C2C2A">c1</text>
<text x="330" y="94" text-anchor="middle" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#2C2C2A">c2</text>
<text x="410" y="94" text-anchor="middle" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#2C2C2A">c3</text>
<text x="490" y="94" text-anchor="middle" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#2C2C2A">c4</text>
<text x="570" y="94" text-anchor="middle" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#2C2C2A">c5</text>
<text x="250" y="110" text-anchor="middle" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="11" fill="#5F5E5A">u1</text>
<text x="330" y="110" text-anchor="middle" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="11" fill="#5F5E5A">u1</text>
<text x="410" y="110" text-anchor="middle" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="11" fill="#5F5E5A">u2</text>
<text x="490" y="110" text-anchor="middle" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="11" fill="#5F5E5A">u2</text>
<text x="570" y="110" text-anchor="middle" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="11" fill="#5F5E5A">u3</text>

<text x="60" y="144" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="12" fill="#04342C">Clients.All</text>
<circle cx="250" cy="140" r="9" fill="#0F6E56"/><circle cx="330" cy="140" r="9" fill="#0F6E56"/><circle cx="410" cy="140" r="9" fill="#0F6E56"/><circle cx="490" cy="140" r="9" fill="#0F6E56"/><circle cx="570" cy="140" r="9" fill="#0F6E56"/>

<text x="60" y="178" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="12" fill="#04342C">Clients.Caller</text>
<circle cx="250" cy="174" r="9" fill="#0F6E56"/><circle cx="330" cy="174" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/><circle cx="410" cy="174" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/><circle cx="490" cy="174" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/><circle cx="570" cy="174" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/>

<text x="60" y="212" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="12" fill="#04342C">Clients.Others</text>
<circle cx="250" cy="208" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/><circle cx="330" cy="208" r="9" fill="#0F6E56"/><circle cx="410" cy="208" r="9" fill="#0F6E56"/><circle cx="490" cy="208" r="9" fill="#0F6E56"/><circle cx="570" cy="208" r="9" fill="#0F6E56"/>

<text x="60" y="246" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="12" fill="#04342C">Clients.Client("c3")</text>
<circle cx="250" cy="242" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/><circle cx="330" cy="242" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/><circle cx="410" cy="242" r="9" fill="#0F6E56"/><circle cx="490" cy="242" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/><circle cx="570" cy="242" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/>

<text x="60" y="280" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="12" fill="#993C1D">Clients.Group("room:1")</text>
<circle cx="250" cy="276" r="9" fill="#BA7517"/><circle cx="330" cy="276" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/><circle cx="410" cy="276" r="9" fill="#BA7517"/><circle cx="490" cy="276" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/><circle cx="570" cy="276" r="9" fill="#BA7517"/>

<text x="60" y="314" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="12" fill="#185FA5">Clients.User("u1")</text>
<circle cx="250" cy="310" r="9" fill="#185FA5"/><circle cx="330" cy="310" r="9" fill="#185FA5"/><circle cx="410" cy="310" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/><circle cx="490" cy="310" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/><circle cx="570" cy="310" r="9" fill="none" stroke="#B4B2A9" stroke-width="1"/>
</svg>
**group 是四样东西里最被误解的一个**。它的准确定位是：**「服务端推送时的目标集合」**，不是"订阅"。

- 客户端**不能监听**一个 group，只有服务端能往 group 推。客户端只能"加入/离开"。
- 而且客户端**不能直接操作 group**——`Groups.AddToGroupAsync` 是服务端 API，必须由你包一个 Hub 方法暴露出去。这是权限边界：否则任何人都能把自己塞进任意房间。

```cs
public class ChatHub : Hub
{
    // 客户端 conn.invoke('Join', 'room:1') —— 必须经你自己的方法，这是唯一的入口
    public async Task Join(string room)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, room);   // 一定要 await
    }

    public Task Leave(string room) => Groups.RemoveFromGroupAsync(Context.ConnectionId, room);

    // 连接建立时自动进组 —— 重连也会重新触发，这是它最可靠的地方
    public override async Task OnConnectedAsync()
    {
        if (Context.UserIdentifier is { } uid)
            await Groups.AddToGroupAsync(Context.ConnectionId, $"user:{uid}");
        await base.OnConnectedAsync();
    }

    public Task Push(string room) => Task.WhenAll(
        Clients.Group(room).SendAsync("Msg", "给整个组"),
        Clients.OthersInGroup(room).SendAsync("Msg", "组里除我之外"),
        Clients.User("u1").SendAsync("Msg", "u1 的全部连接"),
        Clients.Caller.SendAsync("Msg", "只回给我自己")
    );
}

```

**group 成员关系是内存态、不持久化**。应用重启就全没了——但因为 `OnConnectedAsync` 在每次（重）连时都会跑，把 join 逻辑写在那里就等于自动重建了。这也是它比"手动维护一张组表"省心的地方。

### backplane

它解决的是上面那个"映射存在哪"的直接后果：

<svg viewBox="0 0 680 324" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>SignalR backplane 如何把消息扇出到所有实例</title>
<desc>两个服务器各自只持有本地连接，消息先发布到 Redis，再由每个服务器投递给自己的本地客户端。</desc>
<defs>
<marker id="ar2" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M2 1L8 5L2 9" fill="none" stroke="#888780" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
</marker>
</defs>
<rect x="40" y="24" width="600" height="280" rx="20" fill="#F7F6F3" stroke="#D3D1C7" stroke-width="0.5"/>
<text x="60" y="50" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="15" font-weight="500" fill="#2C2C2A">backplane 让 Clients.All 真的等于「所有」</text>
<text x="60" y="70" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#5F5E5A">没有它时，Clients.All 只等于「连到我这台的」—— 这是扩容后第一个撞上的墙</text>

<rect x="230" y="86" width="220" height="44" rx="10" fill="#FAEEDA" stroke="#BA7517" stroke-width="0.5"/>
<text x="340" y="104" text-anchor="middle" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#412402">Redis pub/sub</text>
<text x="340" y="121" text-anchor="middle" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="11" fill="#854F0B">backplane channel</text>

<line x1="185" y1="164" x2="268" y2="126" stroke="#888780" stroke-width="1.5" marker-start="url(#ar2)" marker-end="url(#ar2)"/>
<line x1="412" y1="126" x2="495" y2="164" stroke="#888780" stroke-width="1.5" marker-start="url(#ar2)" marker-end="url(#ar2)"/>

<rect x="80" y="168" width="210" height="44" rx="10" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
<text x="185" y="186" text-anchor="middle" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#042C53">Server 1</text>
<text x="185" y="203" text-anchor="middle" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="11" fill="#185FA5">只认识本地的连接与组</text>

<rect x="390" y="168" width="210" height="44" rx="10" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
<text x="495" y="186" text-anchor="middle" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="13" font-weight="500" fill="#042C53">Server 2</text>
<text x="495" y="203" text-anchor="middle" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="11" fill="#185FA5">只认识本地的连接与组</text>

<path d="M150 212 V226 H120" fill="none" stroke="#888780" stroke-width="0.5"/>
<path d="M220 212 V226 H250" fill="none" stroke="#888780" stroke-width="0.5"/>
<path d="M460 212 V226 H430" fill="none" stroke="#888780" stroke-width="0.5"/>
<path d="M530 212 V226 H560" fill="none" stroke="#888780" stroke-width="0.5"/>

<circle cx="112" cy="236" r="9" fill="#0F6E56"/>
<circle cx="258" cy="236" r="9" fill="#0F6E56"/>
<circle cx="422" cy="236" r="9" fill="#0F6E56"/>
<circle cx="568" cy="236" r="9" fill="#0F6E56"/>

<text x="112" y="262" text-anchor="middle" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="11" fill="#5F5E5A">本地连接</text>
<text x="568" y="262" text-anchor="middle" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="11" fill="#5F5E5A">本地连接</text>

<text x="60" y="288" font-family="system-ui, -apple-system, 'Segoe UI', sans-serif" font-size="12" fill="#993C1D">成员关系仍留在各实例内存里，backplane 只搬消息、不搬状态</text>
</svg>
**backplane 的根源问题**是：连接与分组的映射是**进程内内存**。单机时这没问题，一旦上第二台机器，`Clients.All` 就悄悄退化成"连到我这台的所有"——不报错，只是少推了一半人。修复比较简单：

```cs
builder.Services.AddSignalR()
    .AddStackExchangeRedis("redis:6379");   // Hub 代码一行不改

```

**它换掉的是 `HubLifetimeManager` 这个抽象的实现**（`DefaultHubLifetimeManager` 内存版 → `RedisHubLifetimeManager`），而不是让你改业务代码。这个抽象是全篇的关键：所有"谁来记住谁在哪"的逻辑都被它收口了。





## Vueuse实践

VueUse 是一套基于 **Composition API** 的 Vue 工具函数集合，`@vueuse/core` 是它的核心包，提供了大量开箱即用的组合式函数。具有纯函数式、响应式、SSR友好、类型完整、体积小等优点。覆盖了状态、元素、浏览器、传感器、网络、时间、工具等方面。

`useWebSocket` 只对应上面「客户端 → 原生 WebSocket」这一列，**服务端、信封、group 全要自己写**：

```ts
// app/composables/useChatSocket.ts
import { useWebSocket } from '@vueuse/core'   // 经 @vueuse/nuxt 已自动导入，这行可省

interface Envelope { type: 'chat' | 'ping' | 'pong'; user?: string; text?: string }

export function useChatSocket(url: string) {
  const messages = shallowRef<ChatMessage[]>([])

  const { status, data, send, open, close, ws } = useWebSocket<Envelope>(url, {
    // 这两个默认都是 false，必须显式开（SignalR 恰好相反，默认就开）
    autoReconnect: {
      retries: 10,
      delay: n => Math.min(1000 * 2 ** n, 30_000),   // 1s -> 2s -> 4s ... 封顶 30s
      onFailed: () => console.warn('重连放弃'),
    },
    heartbeat: {
      message: JSON.stringify({ type: 'ping' }),
      responseMessage: JSON.stringify({ type: 'pong' }),
      interval: 25_000,      // 卡在 nginx 60s 的一半
      pongTimeout: 5_000,    // 5s 等不到 pong 就重连
    },
    immediate: false,        // 先不连，由页面决定（比如等登录态就绪）

    onMessage(_, ev) {
      const msg = JSON.parse(ev.data) as Envelope
      if (msg.type === 'pong') return              // 心跳回执不进业务队列
      messages.value = [...messages.value, msg as ChatMessage]
    },
    onDisconnected: (_, ev) => console.warn('closed', ev.code, ev.reason), // 1006 = 异常断开
  })

  function sendChat(user: string, text: string) {
    if (status.value !== 'OPEN') return false
    // 第二参 useBuffer=true：未连接时排队，连上后补发
    send(JSON.stringify({ type: 'chat', user, text }), true)
    return true
  }

  onScopeDispose(close)     // 组件卸载自动断，避免"走后门"的连接
  return { status, data, messages, open, close, ws, sendChat }
}
```

 `data` 是「最新一条」，不是队列。**​ 它是个 `ShallowRef<T | null>`，只保留最后收到的消息。你以为它会累积？不会。要历史就必须自己在 `onMessage` 里攒（上面就是这么做的）。
 
**`autoReconnect` 和 `heartbeat` 默认都是 `false`。​ SignalR 是默认开、默认有值；VueUse 是默认关。不显式配，断线了它不会自己回来，也不会发心跳。

**`heartbeat` 发的是应用层消息，不是协议层 ping。因为浏览器 JS 拿不到协议 ping（这里绕不开）。所以**你的服务端必须能识别 `{"type":"ping"}` 并回 `{"type":"pong"}`**——否则 `pongTimeout` 一到就误判断线，然后开始无限重连风暴。这是最容易炸的一处


