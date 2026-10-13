---
date: 2026-12-10
tags:
  - network
  - tcp
---
tcp工作在不可靠的IP层之上，它之所以可靠是因为依靠 序列号、ACK、超时重传、快速重传、校验和、连接管理。

## TCP的可靠

TCP 有个反直觉的地方值得先说清楚：**它并不保证数据不丢**。它保证的是「丢了能被发现，并且补上」。所以可靠性不是靠某个黑科技，而是**四组机**

<svg viewBox="0 0 680 208" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>TCP 实现可靠传输的四组机制</title>
<desc>序号与确认负责发现丢包与乱序，超时与快速重传负责补发，滑动窗口负责限速，校验和与去重负责排除损坏和重复数据。</desc>
<rect x="40" y="40" width="290" height="64" rx="12" fill="#0C447C" stroke="#85B7EB" stroke-width="0.5"/>
<text x="64" y="66" dominant-baseline="central" fill="#B5D4F4" font-size="14" font-weight="500">序号 + 确认</text>
<text x="64" y="86" dominant-baseline="central" fill="#85B7EB" font-size="12">每个字节编号，按序累积确认</text>
<rect x="350" y="40" width="290" height="64" rx="12" fill="#0C447C" stroke="#85B7EB" stroke-width="0.5"/>
<text x="374" y="66" dominant-baseline="central" fill="#B5D4F4" font-size="14" font-weight="500">超时与快速重传</text>
<text x="374" y="86" dominant-baseline="central" fill="#85B7EB" font-size="12">没等到确认就重发，超时值动态估算</text>
<rect x="40" y="124" width="290" height="64" rx="12" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
<text x="64" y="150" dominant-baseline="central" fill="#9FE1CB" font-size="14" font-weight="500">滑动窗口</text>
<text x="64" y="170" dominant-baseline="central" fill="#5DCAA5" font-size="12">接收窗口 + 拥塞窗口，两头都限速</text>
<rect x="350" y="124" width="290" height="64" rx="12" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
<text x="374" y="150" dominant-baseline="central" fill="#9FE1CB" font-size="14" font-weight="500">校验和与去重</text>
<text x="374" y="170" dominant-baseline="central" fill="#5DCAA5" font-size="12">16 位校验和查损坏，序号重排去重</text>
</svg>




## 传输过程

传输过程中每个字节都有自己的编号：

```
发送方：字节 1-1000 是第 1 段，1001-2000 是第 2 段
接收方：收到后按序号重组，乱序也能还原
```

这样重复的段会按照序号丢弃，乱序到达之后会按照序号重排，确认应答后告诉对方收到了哪些。接收方收到数据之后会回复一个ACK：

```
发送方 → [SEQ=1, 1000字节] → 接收方
发送方 ← [ACK=1001]        ← 接收方
```

ACK=1001 意思是"1000 及之前的字节我都收到了，下一个期望 1001。然后这样累计确认即可。发送发发完数据之后会启动定时器，接着等待ACK，如果没有就会重传SEQ=1， **超时时间（RTO）动态计算**：根据 RTT（往返时间）动态调整。快速重传不需要等待超时，收到3个重复ACK就重传。
```
发送 [1][2][3][4][5]
接收方收到 [1][3][4][5]，缺 [2]
接收方回 ACK=2（重复 3 次）
发送方收到 3 个 ACK=2 → 立即重传 [2]
```
每个TCP都有校验和，用于检测数据是否损坏。连接管理让双方完成三次握手建立连接、四次挥手关闭，能够确保双方都准备好，确保数据都传输完成。

<svg viewBox="0 0 680 430" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>一次丢包如何被快速重传补上</title>
<desc>发送方发出 SEQ 1000 并收到确认后，SEQ 2460 在途中丢失，接收方连续回送三次重复确认 2460，触发快速重传，最终累积确认到 8300。</desc>
<defs>
<marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
</marker>
</defs>
<line x1="150" y1="68" x2="150" y2="410" stroke="#888780" stroke-width="0.5" stroke-dasharray="4 4"/>
<line x1="530" y1="68" x2="530" y2="410" stroke="#888780" stroke-width="0.5" stroke-dasharray="4 4"/>
<text x="150" y="54" text-anchor="middle" dominant-baseline="central" fill="#D3D1C7" font-size="13" font-weight="500">发送方</text>
<text x="530" y="54" text-anchor="middle" dominant-baseline="central" fill="#D3D1C7" font-size="13" font-weight="500">接收方</text>
<line x1="150" y1="96" x2="530" y2="96" stroke="#85B7EB" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="340" y="88" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4" font-size="12">SEQ 1000（1460 B）</text>
<line x1="530" y1="126" x2="150" y2="126" stroke="#5DCAA5" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="340" y="118" text-anchor="middle" dominant-baseline="central" fill="#9FE1CB" font-size="12">ACK 2460</text>
<line x1="150" y1="156" x2="326" y2="156" stroke="#F09595" stroke-width="1.5" stroke-dasharray="5 4"/>
<path d="M334 150L346 162M346 150L334 162" fill="none" stroke="#F09595" stroke-width="1.5" stroke-linecap="round"/>
<text x="236" y="148" text-anchor="middle" dominant-baseline="central" fill="#F7C1C1" font-size="12">SEQ 2460</text>
<text x="358" y="156" text-anchor="start" dominant-baseline="central" fill="#F09595" font-size="12">丢在途中</text>
<line x1="150" y1="186" x2="530" y2="186" stroke="#85B7EB" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="340" y="178" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4" font-size="12">SEQ 3920（乱序，接收方先缓存）</text>
<line x1="530" y1="216" x2="150" y2="216" stroke="#5DCAA5" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="340" y="208" text-anchor="middle" dominant-baseline="central" fill="#9FE1CB" font-size="12">ACK 2460 · 重复确认 #1</text>
<line x1="150" y1="246" x2="530" y2="246" stroke="#85B7EB" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="340" y="238" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4" font-size="12">SEQ 5380</text>
<line x1="530" y1="276" x2="150" y2="276" stroke="#5DCAA5" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="340" y="268" text-anchor="middle" dominant-baseline="central" fill="#9FE1CB" font-size="12">ACK 2460 · 重复确认 #2</text>
<line x1="150" y1="306" x2="530" y2="306" stroke="#85B7EB" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="340" y="298" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4" font-size="12">SEQ 6840</text>
<line x1="530" y1="336" x2="150" y2="336" stroke="#5DCAA5" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="340" y="328" text-anchor="middle" dominant-baseline="central" fill="#9FE1CB" font-size="12">ACK 2460 · 重复确认 #3</text>
<text x="142" y="336" text-anchor="end" dominant-baseline="central" fill="#EF9F27" font-size="12">×3 触发</text>
<line x1="150" y1="366" x2="530" y2="366" stroke="#EF9F27" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="340" y="358" text-anchor="middle" dominant-baseline="central" fill="#FAC775" font-size="12">立刻重传 SEQ 2460，不等超时</text>
<line x1="530" y1="396" x2="150" y2="396" stroke="#5DCAA5" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="340" y="388" text-anchor="middle" dominant-baseline="central" fill="#9FE1CB" font-size="12">ACK 8300（缺口补齐，累积确认）</text>
</svg>

重传达到上限（Linux 默认约 15 次）后 TCP 会放弃并报错——「可靠」不等于「一定会送达」，只等于「一定会告诉你成没成」。而且 **它不保证时延**，TCP 为了可靠可以无限排队，所以时间敏感的场景宁可丢掉过期数据（这就是 WebRTC 用 UDP 的原因）。


顺便讲解udp，udp是故意不做可靠性保证的，他把控制器交给了应用层，udp也有校验和，但是没有序列号、ack、重传、排序、去重、流量控制等等行为。

## tcp窗口

窗口是tcp的流量控制机制，接收方可以提醒发送方自己还能接受多少。接受窗口rwnd是接收方的剩余缓冲区大小：

```
接收方缓冲区 64KB
已用 16KB
剩余 48KB → 告诉发送方"窗口 = 48KB"
```

然后发送方就可以根据窗口大小决定能发多少，这是为了防止接收方被大量的流量淹没，有窗口才能让双方能够按照节奏来发送。

窗口为0的话，发送方就会停止发送，并启动零窗口探测定时器，询问接收方窗口是否开了，当接收方处理完之后窗口才会>0，然后通知发送方继续，如果接收方窗口开了但通知丢了，发送方靠探测恢复。

并且有指标可以计算：  **吞吐量 = 窗口大小 / RTT**

```
窗口 64KB，RTT 100ms
吞吐量 = 64KB / 0.1s = 640KB/s ≈ 5Mbps

窗口 64KB，RTT 10ms
吞吐量 = 64KB / 0.01s = 6.4MB/s ≈ 51Mbps
```

RTT越大，窗口越重要，RTT越大的话，我们自然需要窗口越大，不然吞吐量上不去。

TCP头部窗口字段只有16位，也就是最大64KB，如果还不够，可以使用窗口缩放选项RFC 1323，能够把窗口扩大到8MB，现代TCP窗口都会启用窗口缩放。

tcp窗口分几个层次：内核自动管理、系统参数调整、应用层通过socket选项来间接影响。应用层**看不到也改不了** TCP 窗口字段本身。窗口是内核根据**接收缓冲区剩余空间**自动计算的。

linux可以通过命令查看：

```
# 看系统级参数
sysctl net.ipv4.tcp_rmem
sysctl net.ipv4.tcp_wmem

# 输出示例：
# net.ipv4.tcp_rmem = 4096 131072 6291456
#                    min  default  max
```

## tcp握手和挥手

<svg viewBox="0 0 680 235" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>TCP 三次握手的报文与时序</title>
<desc>客户端发 SYN 携带初始序号 x 并进入 SYN_SENT，服务端回 SYN+ACK 携带初始序号 y 并确认 x+1，客户端再回 ACK 确认 y+1，双方进入 ESTABLISHED。</desc>
<defs>
<marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
</marker>
</defs>
<line x1="170" y1="60" x2="170" y2="215" stroke="#888780" stroke-width="0.5" stroke-dasharray="4 4"/>
<line x1="500" y1="60" x2="500" y2="215" stroke="#888780" stroke-width="0.5" stroke-dasharray="4 4"/>
<text x="170" y="48" text-anchor="middle" dominant-baseline="central" fill="#D3D1C7" font-size="13" font-weight="500">客户端</text>
<text x="500" y="48" text-anchor="middle" dominant-baseline="central" fill="#D3D1C7" font-size="13" font-weight="500">服务端</text>
<line x1="170" y1="92" x2="500" y2="92" stroke="#85B7EB" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="335" y="84" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4" font-size="12">① SYN · seq = x</text>
<line x1="500" y1="142" x2="170" y2="142" stroke="#5DCAA5" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="335" y="134" text-anchor="middle" dominant-baseline="central" fill="#9FE1CB" font-size="12">② SYN + ACK · seq = y, ack = x + 1</text>
<text x="335" y="166" text-anchor="middle" dominant-baseline="central" fill="#B4B2A9" font-size="12">服务端把 ACK 和 SYN 合并成一步</text>
<line x1="170" y1="192" x2="500" y2="192" stroke="#85B7EB" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="335" y="184" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4" font-size="12">③ ACK · ack = y + 1</text>
<text x="162" y="78" text-anchor="end" dominant-baseline="central" fill="#888780" font-size="12">CLOSED</text>
<text x="162" y="122" text-anchor="end" dominant-baseline="central" fill="#888780" font-size="12">SYN_SENT</text>
<text x="162" y="210" text-anchor="end" dominant-baseline="central" fill="#97C459" font-size="12">ESTABLISHED</text>
<text x="508" y="78" text-anchor="start" dominant-baseline="central" fill="#888780" font-size="12">LISTEN</text>
<text x="508" y="172" text-anchor="start" dominant-baseline="central" fill="#888780" font-size="12">SYN_RCVD</text>
<text x="508" y="210" text-anchor="start" dominant-baseline="central" fill="#97C459" font-size="12">ESTABLISHED</text>
</svg>


**为什么必须是三次，两次不行？** 因为服务端在收到 ① 之后**无法确认自己的 ISN 是否被客户端收到**。如果只有两次，服务端发完 SYN+ACK 就认为连接建立，那还有一种糟糕情况：客户端很久之前发出的一个重复 SYN（网络里迷路的旧报文）突然抵达，服务端兴冲冲建好连接、分配资源，而客户端根本不知道有这条连接——资源就这么白占了，实际成了一条「幽灵连接」。第三次的 ACK 就是让服务端确认「这次握手是当下发生的一次真实往返」。**​三次还能顺带解决序号同步**：只有客户端回了 `ack = y + 1`，服务端才知道自己的序号 y 已被对方接受。

<svg viewBox="0 0 680 328" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>TCP 四次挥手与 TIME_WAIT 状态</title>
<desc>主动方发 FIN 进入 FIN_WAIT_1，被动方回 ACK 进入半关闭，之后被动方再发自己的 FIN，主动方回 ACK 并进入 TIME_WAIT，等待 2MSL 后才真正关闭。</desc>
<defs>
<marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
</marker>
</defs>
<line x1="170" y1="60" x2="170" y2="262" stroke="#888780" stroke-width="0.5" stroke-dasharray="4 4"/>
<line x1="170" y1="262" x2="170" y2="308" stroke="#EF9F27" stroke-width="4"/>
<line x1="500" y1="60" x2="500" y2="308" stroke="#888780" stroke-width="0.5" stroke-dasharray="4 4"/>
<text x="170" y="48" text-anchor="middle" dominant-baseline="central" fill="#D3D1C7" font-size="13" font-weight="500">主动关闭方</text>
<text x="500" y="48" text-anchor="middle" dominant-baseline="central" fill="#D3D1C7" font-size="13" font-weight="500">被动关闭方</text>
<line x1="170" y1="96" x2="500" y2="96" stroke="#85B7EB" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="335" y="88" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4" font-size="12">① FIN · seq = u</text>
<line x1="500" y1="136" x2="170" y2="136" stroke="#5DCAA5" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="335" y="128" text-anchor="middle" dominant-baseline="central" fill="#9FE1CB" font-size="12">② ACK · ack = u + 1</text>
<text x="335" y="176" text-anchor="middle" dominant-baseline="central" fill="#B4B2A9" font-size="12">半关闭：对方还可能继续发数据，ACK 与 FIN 不能合并</text>
<line x1="500" y1="206" x2="170" y2="206" stroke="#5DCAA5" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="335" y="198" text-anchor="middle" dominant-baseline="central" fill="#9FE1CB" font-size="12">③ FIN · seq = w</text>
<line x1="170" y1="246" x2="500" y2="246" stroke="#85B7EB" stroke-width="1.5" marker-end="url(#arrow)"/>
<text x="335" y="238" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4" font-size="12">④ ACK · ack = w + 1</text>
<text x="162" y="88" text-anchor="end" dominant-baseline="central" fill="#888780" font-size="12">FIN_WAIT_1</text>
<text x="162" y="178" text-anchor="end" dominant-baseline="central" fill="#888780" font-size="12">FIN_WAIT_2</text>
<text x="162" y="280" text-anchor="end" dominant-baseline="central" fill="#FAC775" font-size="12">TIME_WAIT</text>
<text x="162" y="298" text-anchor="end" dominant-baseline="central" fill="#EF9F27" font-size="12">等 2MSL 后关闭</text>
<text x="508" y="158" text-anchor="start" dominant-baseline="central" fill="#888780" font-size="12">CLOSE_WAIT</text>
<text x="508" y="232" text-anchor="start" dominant-baseline="central" fill="#888780" font-size="12">LAST_ACK</text>
<text x="508" y="280" text-anchor="start" dominant-baseline="central" fill="#97C459" font-size="12">CLOSED</text>
</svg>

挥手的核心在于：**FIN 不是「我要关了」，而是「我这个方向没数据要发了」**。所以它天然是一个半关闭（half-close）​操作，四次挥手就是两个方向各关一次：

- 被动方收到 FIN 后，**内核立刻回 ACK**（协议栈自动完成，不等应用），此时连接变成「我可以收、你也可以继续发」——这就是 `CLOSE_WAIT`。它应用层的代码可能还在跑、还要回数据，所以**不能顺手把自己的 FIN 也塞进这个 ACK**。
- 等它真的处理完、调了 `close()`，才发出自己的 ③ FIN。主动方回 ④ ACK 后，**不能马上关闭，要进 `TIME_WAIT`**。


## WS协议

WS 借用了 HTTP/1.1 的 `Upgrade` 机制，**把同一条已经建好的 HTTP 连接上的应用层协议换掉**。没有新建连接、没有新端口、TCP 和 TLS 的握手一个都没重来——这也是 WS 能拿到极低延迟的根本原因。

<svg viewBox="0 0 680 300" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>HTTP 与 WebSocket 的协议分层</title>
<desc>WebSocket 与 HTTP 共用同一条 TCP 连接，Upgrade 换掉的只是最上面那一层应用层协议，TLS、TCP、IP 完全不变。</desc>
<defs>
<marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
</marker>
</defs>
<rect x="70" y="40" width="200" height="48" rx="8" fill="#0C447C" stroke="#85B7EB" stroke-width="0.5"/>
<text x="170" y="64" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4" font-size="14" font-weight="500">HTTP/1.1 请求响应</text>
<rect x="410" y="40" width="200" height="48" rx="8" fill="#633806" stroke="#EF9F27" stroke-width="0.5"/>
<text x="510" y="64" text-anchor="middle" dominant-baseline="central" fill="#FAC775" font-size="14" font-weight="500">WebSocket 数据帧</text>
<line x1="284" y1="64" x2="396" y2="64" stroke="#B4B2A9" stroke-width="1" marker-end="url(#arrow)"/>
<text x="340" y="46" text-anchor="middle" dominant-baseline="central" fill="#B4B2A9" font-size="12">Upgrade</text>
<rect x="70" y="104" width="540" height="34" rx="8" fill="none" stroke="#5F5E5A" stroke-width="0.5" stroke-dasharray="5 4"/>
<text x="340" y="121" text-anchor="middle" dominant-baseline="central" fill="#888780" font-size="12">TLS —— 只有 wss:// 才有，ws:// 直接省略这一层</text>
<rect x="70" y="154" width="540" height="40" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
<text x="340" y="174" text-anchor="middle" dominant-baseline="central" fill="#9FE1CB" font-size="14" font-weight="500">TCP —— 自始至终是它：可靠、有序的字节流</text>
<rect x="70" y="210" width="540" height="40" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
<text x="340" y="230" text-anchor="middle" dominant-baseline="central" fill="#9FE1CB" font-size="14" font-weight="500">IP</text>
<text x="70" y="276" text-anchor="start" dominant-baseline="central" fill="#B4B2A9" font-size="12">握手阶段是普通 HTTP 报文；收到 101 之后，同一个 socket 上跑的就全是帧了</text>
</svg>

补一个容易踩的细节：**`Upgrade` 这个头只在 HTTP/1.1 里存在**。所以如果要走 HTTP/2（比如通过 h2 的 CDN 回源），就不能用这套升级舞步，得改用 RFC 8441 的「扩展 CONNECT」——用 `:protocol` 伪头来协商 WS，机制完全不同。这也侧面说明：WS 的「升级」是**绑在 HTTP/1.1 语义上的一个历史设计**。

接下来是握手本身。使用的是 SHA-1 + magic GUID ， RFC 6455 里的原例算了一遍对账：

<svg viewBox="0 0 680 400" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>WebSocket 握手的三个步骤</title>
<desc>客户端发出带 Upgrade 和随机 Key 的 HTTP 请求，服务端把 Key 拼上固定 GUID 做 SHA-1 再 base64，然后回 101 Switching Protocols 并带上 Sec-WebSocket-Accept。</desc>
<defs>
<marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
</marker>
</defs>
<rect x="40" y="40" width="600" height="72" rx="12" fill="#0C447C" stroke="#85B7EB" stroke-width="0.5"/>
<text x="64" y="64" dominant-baseline="central" fill="#B5D4F4" font-size="14" font-weight="500">① 客户端发一个普通 HTTP/1.1 请求，夹带升级意图</text>
<text x="64" y="88" dominant-baseline="central" fill="#85B7EB" font-size="12" font-family="ui-monospace, monospace">Upgrade: websocket   ·   Sec-WebSocket-Key: 16 字节随机数的 base64</text>
<line x1="340" y1="112" x2="340" y2="168" stroke="#B4B2A9" stroke-width="1" marker-end="url(#arrow)"/>
<text x="352" y="140" dominant-baseline="central" fill="#B4B2A9" font-size="12">拼接 + SHA-1 + base64</text>
<rect x="40" y="168" width="600" height="84" rx="12" fill="#633806" stroke="#EF9F27" stroke-width="0.5"/>
<text x="64" y="192" dominant-baseline="central" fill="#FAC775" font-size="14" font-weight="500">② 服务端做一次摘要，这一步没有任何秘密</text>
<text x="64" y="216" dominant-baseline="central" fill="#EF9F27" font-size="12" font-family="ui-monospace, monospace">SHA1( Sec-WebSocket-Key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11" )</text>
<text x="64" y="238" dominant-baseline="central" fill="#EF9F27" font-size="12" font-family="ui-monospace, monospace">→ s3pPLMBiTxaQ9kYGzzhZRbK+xOo=</text>
<line x1="340" y1="252" x2="340" y2="308" stroke="#B4B2A9" stroke-width="1" marker-end="url(#arrow)"/>
<text x="352" y="280" dominant-baseline="central" fill="#B4B2A9" font-size="12">101 之后协议切换生效</text>
<rect x="40" y="308" width="600" height="72" rx="12" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
<text x="64" y="332" dominant-baseline="central" fill="#9FE1CB" font-size="14" font-weight="500">③ 服务端回 101 Switching Protocols</text>
<text x="64" y="356" dominant-baseline="central" fill="#5DCAA5" font-size="12" font-family="ui-monospace, monospace">Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=</text>
</svg>

实际的报文长这样（Key 用的是 RFC 6455 里的示例值，Accept 是我上面实测算出来的）：

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**那个 GUID 是写死的常量**，`258EAFA5-E914-47DA-95CA-C5AB0DC85B11`，全世界所有 WS 连接都一样，出现在 RFC 6455 §1.3。所以整个变换过程**没有任何保密性**——任何人都能复算。 **客户端必须自己复算一遍并比对** Accept，不符就直接断开，不进入帧阶段。

握手还有个实用细节：`Sec-WebSocket-Protocol` 用来协商子协议（比如 `graphql-ws`、`mqtt`），`Sec-WebSocket-Extensions` 用来协商扩展（`permessage-deflate` 就是在这里开启的压缩）。服务端选的子协议如果不合法，客户端应当断开——协议没规定要严格，但规范建议如此。

升完级之后，字节流上跑的就是**帧**了：

<svg viewBox="0 0 680 212" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>WebSocket 帧的字节结构</title>
<desc>一个 WebSocket 帧由两字节头部、可选的扩展长度、可选的四字节掩码密钥和载荷组成，掩码在客户端到服务端方向是强制的。</desc>
<rect x="40" y="48" width="170" height="88" rx="8" fill="#0C447C" stroke="#85B7EB" stroke-width="0.5"/>
<text x="125" y="74" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4" font-size="13" font-weight="500">头部 2 字节</text>
<text x="125" y="98" text-anchor="middle" dominant-baseline="central" fill="#85B7EB" font-size="11">FIN · RSV · opcode</text>
<text x="125" y="118" text-anchor="middle" dominant-baseline="central" fill="#85B7EB" font-size="11">MASK · 长度 7 位</text>
<rect x="222" y="48" width="150" height="88" rx="8" fill="#0C447C" stroke="#85B7EB" stroke-width="0.5"/>
<text x="297" y="74" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4" font-size="13" font-weight="500">扩展长度</text>
<text x="297" y="98" text-anchor="middle" dominant-baseline="central" fill="#85B7EB" font-size="11">0 / 16 / 64 位</text>
<text x="297" y="118" text-anchor="middle" dominant-baseline="central" fill="#85B7EB" font-size="11">按需出现</text>
<rect x="384" y="48" width="130" height="88" rx="8" fill="#633806" stroke="#EF9F27" stroke-width="0.5"/>
<text x="449" y="74" text-anchor="middle" dominant-baseline="central" fill="#FAC775" font-size="13" font-weight="500">掩码密钥</text>
<text x="449" y="98" text-anchor="middle" dominant-baseline="central" fill="#EF9F27" font-size="11">32 位随机数</text>
<text x="449" y="118" text-anchor="middle" dominant-baseline="central" fill="#EF9F27" font-size="11">客户端必带</text>
<rect x="526" y="48" width="114" height="88" rx="8" fill="#0C447C" stroke="#85B7EB" stroke-width="0.5"/>
<text x="583" y="74" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4" font-size="13" font-weight="500">载荷</text>
<text x="583" y="98" text-anchor="middle" dominant-baseline="central" fill="#85B7EB" font-size="11">应用数据</text>
<text x="583" y="118" text-anchor="middle" dominant-baseline="central" fill="#85B7EB" font-size="11">长度由头部给</text>
<text x="40" y="166" dominant-baseline="central" fill="#888780" font-size="12">长度 ≤125 直接写；126 → 后接 16 位；127 → 后接 64 位</text>
<text x="40" y="190" dominant-baseline="central" fill="#EF9F27" font-size="12">客户端 → 服务端必须掩码；服务端 → 客户端必须不掩码</text>
</svg>

TCP 只提供「有序的字节流」，**它没有消息边界**——你发了两个消息，对端可能一次 `read` 就收到粘在一起的两个，也可能被拆成三块。WS 的帧就是来补这个缺口的：`FIN` 标记消息结束，`opcode` 标记这条消息是什么类型，于是「消息」这个概念第一次在协议层被定义出来。这是 WS 相对裸 TCP 最核心的增值（其次是它能穿过 80/443 的代理和防火墙）。

它的报文形式：**​WS 的消息是二进制帧**

ws客户端的文本帧字节可能长这样：

<svg viewBox="0 0 680 326" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>WebSocket 客户端文本帧的字节结构</title>
<desc>一条 26 字节的客户端文本帧由 2 字节帧头、4 字节掩码键和 20 字节掩码后的载荷组成，载荷与掩码键逐字节异或可还原原文。</desc>
<text x="40" y="32" font-size="14" font-weight="500" fill="var(--color-text-primary)" font-family="ui-sans-serif, system-ui, sans-serif">客户端 → 服务端：一条文本帧，共 26 字节</text>
<rect x="60" y="58" width="44" height="30" rx="6" fill="#0C447C" stroke="#85B7EB" stroke-width="0.5"/>
<text x="82" y="73" font-size="12" text-anchor="middle" dominant-baseline="central" fill="#E6F1FB" font-family="ui-monospace, Consolas, monospace">81</text>
<text x="116" y="73" font-size="12" dominant-baseline="central" fill="var(--color-text-secondary)" font-family="ui-sans-serif, system-ui, sans-serif">FIN=1 + opcode=0x1 → text 帧</text>
<rect x="60" y="96" width="44" height="30" rx="6" fill="#0C447C" stroke="#85B7EB" stroke-width="0.5"/>
<text x="82" y="111" font-size="12" text-anchor="middle" dominant-baseline="central" fill="#E6F1FB" font-family="ui-monospace, Consolas, monospace">94</text>
<text x="116" y="111" font-size="12" dominant-baseline="central" fill="var(--color-text-secondary)" font-family="ui-sans-serif, system-ui, sans-serif">MASK=1 + payload 长度 20</text>
<rect x="60" y="134" width="110" height="30" rx="6" fill="#633806" stroke="#EF9F27" stroke-width="0.5"/>
<text x="74" y="149" font-size="12" dominant-baseline="central" fill="#FAC775" font-family="ui-monospace, Consolas, monospace">37 FA 21 3D</text>
<text x="182" y="149" font-size="12" dominant-baseline="central" fill="var(--color-text-secondary)" font-family="ui-sans-serif, system-ui, sans-serif">masking-key：4 字节，明文写在帧里</text>
<rect x="60" y="172" width="216" height="30" rx="6" fill="#444441" stroke="#B4B2A9" stroke-width="0.5"/>
<text x="74" y="187" font-size="12" dominant-baseline="central" fill="#D3D1C7" font-family="ui-monospace, Consolas, monospace">4C D8 55 44 47 9F 03 07 …</text>
<text x="288" y="187" font-size="12" dominant-baseline="central" fill="var(--color-text-secondary)" font-family="ui-sans-serif, system-ui, sans-serif">掩码后的载荷（20 字节）</text>
<text x="60" y="232" font-size="13" dominant-baseline="central" fill="var(--color-text-secondary)" font-family="ui-sans-serif, system-ui, sans-serif">把载荷逐字节 XOR 掩码键，还原出原文：</text>
<rect x="60" y="248" width="340" height="40" rx="8" fill="#27500A" stroke="#97C459" stroke-width="0.5"/>
<text x="78" y="268" font-size="13" dominant-baseline="central" fill="#C0DD97" font-family="ui-monospace, Consolas, monospace">{"type":"heartbeat"}</text>
<text x="40" y="306" font-size="11" dominant-baseline="central" fill="var(--color-text-tertiary)" font-family="ui-monospace, Consolas, monospace">完整字节：81 94 37 FA 21 3D 4C D8 55 44 47 9F 03 07 15 92 44 5C 45 8E 43 58 56 8E 03 40</text>
</svg>

这个和 SSE 的差别一眼就能看出来：**SSE 是一行行文本，WS 是一串字节**。帧头就两个字节，拆开看全是位域。

然后我们来说说实战中ws会话的维护吧，连接得放哪条线程上？SignalR、socket.io 的客户端都能跑在 Worker 里，也默认可跑在主线程——Worker 不是库的特性，是你自己选的运行位置。我们主流上认为主线程是默认的，比如socket.io中的io()还是vueuse的useWebsocket、SignalR的new HubConnectionBuilder()。Worker只有具体理由的时候才会上，而worker有实打实的代价，真正算得上值得上的只有一种情况，那就是需要 跨标签页共享一条连接的时候，那个时候我们可能需要的是SharedWorker，而不是Dedicated Worker。

>SharedWorker是Web Worker的一种，区别在于它支持多个浏览上下文，读个标签页和iframe，共享同一个worker实例

SharedWorker 的通信需要先获取 `port`，再通过它收发消息：

```ts
const myWorker = new SharedWorker("worker.js");
myWorker.port.start(); // 手动启动端口
myWorker.port.postMessage([1, 2]);
myWorker.port.onmessage = (e) => console.log(e.data);

// 在 Worker 内部，通过 `onconnect` 事件获取连接的端口，然后监听消息
self.onconnect = (e) => {
  const port = e.ports[0];
  port.onmessage = (event) => {
    port.postMessage(event.data[0] * event.data[1]);
  };
};
```

为什么我说还是主线程是默认呢，因为 `postMessage` 是结构化克隆，每条消息跨线程深拷贝一遍（除非用 Transferable / SharedArrayBuffer）。所以"放进 Worker"并不天然更快——消息量大、载荷小的时候，拷贝成本可能比解析成本还高。Worker 收到 → `postMessage` → 主线程派发，比主线程直接收多一个 tick。对超低延迟（撮合、行情）反而是负担。

具体的我放专项的文档里再说吧，毕竟本章讲解的是TCP协议。



