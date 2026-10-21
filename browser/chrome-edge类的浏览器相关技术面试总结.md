---
date: 2026-10-02
tags:
  - browser
---
我这里整理了面试中可能会考到的浏览器的部分知识和理论，后续会更新一些新的知识上去

### localStorage加密邪教技术

cookie和LocalStorage是web开发中最常用的两个存储方式，cookie主要保存在内存中，生命周期和浏览器相关，浏览器关闭之后直接消失（会话cookie），持久cookie是保存在硬盘的特定文件里，生命周期是设置的过期时间。

在Chrome这种浏览器中，持久化cookie存储在用户数据目录里面的cookies SQLITE数据库里，例如 C:\Users\用户名\AppData\Local\Google\Chrome\User Data\Default\cookies

然后看看localStorage，这个始终是存储在硬盘里的独立区域的，Chrome呢，存储在User Data\Default\Local Storage\目录里， **Firefox**：存储在 `webappsstore.sqlite` 文件中。

cookie的原理是http状态管理，服务器在http响应头里可以加Set-Cookie，浏览器收到后就会保存，每一次向同源服务器发请求的时候浏览器都会自动把Cookie塞进Cookie请求头里，每个Cookie最大4KB，每次请求都会携带，太大会拖慢网络，他有安全属性，HttpOnly让JS读不到，Secure只走HTTPS，SameSite可以控制跨站发送。

localStorage的原理是Web Storage API，是纯客户端，数据只会在浏览器本地，永远不会自动发给服务器，也是按源来隔离的，提供了一些同步API，可以通过setItem、getItem、removeItem等方法直接读写键值对。容量更大，通常为5MB左右，远大于Cookie的4KB，只存字符串，存对象需要先JSON.stringify，取出来再JSON.parse。

聊完了之前技术，有一个好玩的东西，有些库`secure-ls`、`@cyberadityacode/secure-ls` 等可以让我们在localSotrage中存储加密数据，用起来和普通的localStorage差不多，原理是在存入前加密，库会封装setItem和getItem，传入的对象会被AES之类的算法加密成密文，再存进localStorage中，读取的时候自动解密，给我们原始对象。

```ts
// 普通 localStorage
localStorage.setItem('user', JSON.stringify(user))

// 加密库的用法（API 类似，但底层自动加解密）
await storage.setItem('user', user)  // 存进去的是密文
const user = await storage.getItem('user')  // 取出来是明文对象
```

### Service Worker

HTML5的离线存储，也叫做Application Cache，也在已经废弃，继任者叫做Service Worker，ServiceWorker是一个在浏览器后台运行的独立脚本，充当网络请求的代理，可以拦截页面所有请求，并决定是返回缓存资源还是去网络获取，使用分为三步：

注册、安装于缓存、拦截和响应。

注册会在js中注册一个service worker的脚本，安装和缓存会在serviceWorker的install事件中，将需要离线使用的核心资源比如html css js加入到缓存，拦截和响应会在fetch事件中，拦截页面的网络请求，可以制定策略，比如说缓存优先，看看有没有资源，没有再去请求网络，或者网络优先，先请求，失败再回退缓存。

相比 AppCache，Service Worker 提供了**更精细、更灵活的控制能力**。你可以精确管理缓存的版本、更新策略，并且它完全通过 JavaScript 编程实现，不再依赖那个容易出错的 manifest 文件。

service worker不会阻塞主线程，也不能访问dom，靠事件唤醒，空闲时候被终止，能够拦截fetch，再决定怎么响应。注册之后可以长期存在，页面关了也在，必须https或者localhost，只能控制注册路径和子路径。

有service worker之后可以决定

```
页面 → 浏览器 → Service Worker（拦截）→ 决定：
                    ├─ 用缓存响应
                    ├─ 走网络
                    └─ 自定义响应
```

Service Worker 的生命周期和普通脚本完全不同，分几个阶段：

首先是注册：

```js
// 主线程
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js')
        .then(reg => console.log('注册成功', reg))
        .catch(err => console.log('注册失败', err));
}
```
注册的时候文件不会加载，不会立即生效，然后install的时候才会。

```js
// sw.js
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open('v1').then(cache => {
            return cache.addAll([
                '/',
                '/index.html',
                '/style.css',
                '/app.js'
            ]);
        })
    );
});
```

install事件在sw首次注册的时候或者文件内容变化的时候触发，通常在预缓存静态资源。 `event.waitUntil(promise)`让浏览器"等这个 Promise 完成再算安装成功"。然后是activate激活阶段：

```js
self.addEventListener('activate', (event) => {
    event.waitUntil(
        caches.keys().then(keys => {
            return Promise.all(
                keys.filter(k => k !== 'v1')
                    .map(k => caches.delete(k))   // 清理旧缓存
            );
        })
    );
});
```

`activate` 在安装成功后、SW 正式接管页面前触发。通常在这里**清理旧版本缓存**。最后是接管：

```js
self.addEventListener('activate', (event) => {
    event.waitUntil(self.clients.claim());   // 立即接管
});
```
默认情况下，sw安装不会立即开始接管已经打开的窗口，需要等待页面刷新，使用这个情况我们可以在这里激活后直接接管。
之后service worker就可以帮我们来完成一系列工作了，比如说拦截请求：

```js
self.addEventListener('fetch', (event) => {
    event.respondWith(
        caches.match(event.request).then(cached => {
            // 缓存命中 → 用缓存
            if (cached) return cached;
            // 没命中 → 走网络
            return fetch(event.request);
        })
    );
});
```

我们可以在这里设置一个缓存优先策略/网络优先策略，这个是fetch事件的常见写法，适合需要最新数据但是也要离线兜底的接口。

```js
self.addEventListener('fetch', (event) => {
    event.respondWith(
        caches.match(event.request).then(cached => {
            return cached || fetch(event.request);
        })
    );
}); // 缓存

self.addEventListener('fetch', (event) => {
    event.respondWith(
        fetch(event.request)
            .then(response => {
                // 网络成功，更新缓存
                const clone = response.clone();
                caches.open('v1').then(cache => cache.put(event.request, clone));
                return response;
            })
            .catch(() => caches.match(event.request))   // 网络失败用缓存
    );
});
```

如果是能够短暂接受旧数据，但是还是想尽快响应的，场景，可以用缓存优先+后台更新：

```js
self.addEventListener('fetch', (event) => {
    event.respondWith(
        caches.match(event.request).then(cached => {
            const fetchPromise = fetch(event.request).then(response => {
                const clone = response.clone();
                caches.open('v1').then(cache => cache.put(event.request, clone));
                return response;
            });
            return cached || fetchPromise;   // 立即返回缓存，后台更新
        })
    );
});
```

sw里面可以用CacheStorage来管理缓存：

```js
// 打开/创建缓存
const cache = await caches.open('v1');

// 添加
await cache.add('/index.html');
await cache.addAll(['/a.js', '/b.css']);
await cache.put(request, response);   // 手动放入

// 查询
const response = await cache.match('/index.html');
const responses = await cache.matchAll();

// 删除
await cache.delete('/index.html');

// 列出所有缓存
const keys = await caches.keys();

// 删除整个缓存
await caches.delete('v1');
```

sw和页面不能直接通信，需要使用postMessage：

```js
// 主线程
navigator.serviceWorker.controller.postMessage({
    type: 'SKIP_WAITING'
});

navigator.serviceWorker.addEventListener('message', (event) => {
    console.log('收到 SW 消息', event.data);
});
```

这个架构设计其实普遍常见，可以看做后端架构里的Worker，比如说Cloudflare Worker或者 Web Workers、后台任务队列：

- **独立执行单元**
    
- **事件驱动**
    
- **无状态或短生命周期**
    
- **通过消息通信**
    
- **不直接访问主进程资源

**Service Worker 完全符合这个模型**。

提一嘴，`CacheStorage` 存在**磁盘**上，和 `IndexedDB`、`localStorage` 一样是持久化存储。

在vite项目中，我们可以使用vue专用sw包装的插件vite-plugin-pwa，它通过一个虚拟模块 `virtual:pwa-register/vue`，提供了一个专门给 Vue 3 用的 `useRegisterSW` 函数：

```vue
<script setup lang="ts">
import { useRegisterSW } from 'virtual:pwa-register/vue'

const {
  offlineReady, // 应用准备好离线使用（Ref<boolean>）
  needRefresh,  // 有新版本可用（Ref<boolean>）
  updateServiceWorker, // 更新 SW 并刷新页面（函数）
} = useRegisterSW({
  onRegisteredSW(swUrl) {
    console.log(`SW 已注册: ${swUrl}`)
  },
})
</script>

<template>
  <div v-if="offlineReady || needRefresh" class="pwa-toast">
    <span v-if="offlineReady">应用已可离线使用</span>
    <span v-else>有新内容可用，点击刷新</span>
    <button v-if="needRefresh" @click="updateServiceWorker()">刷新</button>
  </div>
</template>
```

### Web Worker

刚才提到的service worker常用于网络代理层，web worker常用于后台计算线程，能够把耗时计算移出主线程，生命周期是随着页面的创建和销毁。这个是html5提供的一个api，允许我们在后台线程运行js代码，从而避免主线程，它是浏览器实现多线程的标准方式。

因为Js是单线程，主线程会负责执行js代码，处理dom渲染，响应用户交互。如果主线程执行一个耗时任务比如大量计算、处理大数组，页面就会卡死，用户点击没反应、动画掉帧。Web Worker 把耗时任务放到**独立线程**，主线程继续响应用户，两者互不阻塞。

```js
// 1. 创建 Worker
const worker = new Worker('worker.js');

// 2. 发送数据给 Worker
worker.postMessage({ type: 'compute', data: [1, 2, 3, 4, 5] });

// 3. 接收 Worker 的结果
worker.onmessage = (e) => {
    console.log('结果:', e.data);
};

// 4. 监听错误
worker.onerror = (err) => {
    console.error('Worker 错误:', err.message);
};

// 5. 终止 Worker
// worker.terminate();

// worker.js
// 接收主线程的消息
self.onmessage = (e) => {
    const { type, data } = e.data;

    if (type === 'compute') {
        // 执行耗时计算
        const result = data.reduce((sum, n) => sum + n * n, 0);

        // 把结果发回主线程
        self.postMessage({ type: 'result', value: result });
    }
};
```


`postMessage` 支持两种传输方式，分别是结构化克隆、转移所有权。默认是结构化克隆：

```js
worker.postMessage({
    num: 1,
    str: 'hello',
    arr: [1, 2, 3],
    obj: { a: 1 },
    map: new Map([['k', 'v']]),
    set: new Set([1, 2]),
    date: new Date(),
    buffer: new ArrayBuffer(8)
});
```

不支持dom节点、error对象（部分）、函数。

转移所有权有点像c++的移动move概念。对于大块数据比如ArrayBuffer，可以用转移替代复制，实现零拷贝：

```js
const buffer = new ArrayBuffer(1024 * 1024 * 100); // 100MB

// 转移所有权，主线程不再拥有该 buffer
worker.postMessage({ buffer }, [buffer]);

// 此时 buffer 在主线程已不可用
console.log(buffer.byteLength); // 0
```

处理大数组、图像数据、音视频时，转移比复制快得多。典型的使用场景就有wasm，我们一般就是在worker中运行wasm模块的。可以用于图像处理、像素操作、canvas离屏渲染。还能用于轮询和长连接，后台定时请求、websocket心跳，这也是为了主线程不会去做大json解析、二进制处理处理占用主线程。

### IndexedDB

这个是浏览器的嵌入式数据库，用kv存储，支持索引、事务、游标。能存储任意结构化数据，localStorage只能存储5-10MB，但是这个可以存储上GB的信息量。一个indexedDB可以创建多个数据库：

```
const request = indexedDB.open('myDB', 1);   // 名字 + 版本号
```

他可以使用对象仓库，类似于表：

```
db.createObjectStore('users', { keyPath: 'id' });
```

可以使用索引加速查询：

```
store.createIndex('email', 'email', { unique: true });
store.createIndex('age', 'age');
store.createIndex('name_age', ['name', 'age']);   // 复合索引
```

支持事务，能确保原子性和自动提交，出错能自动回滚：

```
const tx = db.transaction('users', 'readwrite');
const store = tx.objectStore('users');
store.add({ id: 1, name: 'Alice' });
tx.oncomplete = () => console.log('提交成功');
tx.onerror = () => console.log('失败');
```

### Ajax技术

Ajax是一组技术的组合，用于在不刷新页面的前提下用js异步向服务器请求数据并且更新页面的方法。

我们已知，HTML/CSS用于展示内容，DOM动态更新页面，JS用于逻辑控制，那么XMLHttpRequest可以用于一步请求，XML/JSON可以用于数据展示。

我们都知道现代ajax技术经过了发展从XMLHttpRequest变成了fetch，又变成了axios。XMLHttpRequest是1999发明，es6之后，发布了fetch。

### PWA

**Progressive Web App** 是一组能力的集合，核心是Service Worker + Web App Manifest。他可以做到离线可用，断网也可以打开，可安装，桌面/主屏有图标，独立窗口。可以推送通知，可以后台同步。

service worker之前讲过，是一个跑在浏览器后台独立线程的脚本，可以拦截页面的网络请求、缓存资源、离线响应。


### 浏览器缓存技术

浏览器本身给我们提供了一些缓存技术，最基础的缓存是HTTP响应头机制，通过强缓存可以将带内容哈希的静态资源比如app.a1b2c3.js缓存。

Cache-Control: 这个头我们在缓存里最常用，可以设置max-age= 单位s来以高优先级设置缓存资源，也可以用immutable设置资源永不变，刷新也不请求。头部可以设置Expires来设置绝对时间过期，优先级比较低。

Cache-Control: public, max-age=31536000, immutable 比如这样。

浏览器可以发送请求询问服务器是否可用，这个叫做协商缓存：

`ETag` / `If-None-Match` 可以基于内容哈希，精度比较高，`Last-Modified` / `If-Modified-Since`基于时间戳，精度比较低。响应为304 Not Modified，不传响应体，比较节省带宽。 ETag是服务器为了某个资源生成的唯一标识符，通常基于文件内容计算比如哈希值，这样客户端在下次请求的时候就会通知服务器本地存有这个版本，并判断是否资源最新的。

这是协商的完整过程：

```
第一次请求：
  客户端 ──GET /index.html──→ 服务器
  客户端 ←──200 OK────────── 服务器
              ETag: "abc123"
              Cache-Control: no-cache
// - `no-cache` 表示**每次都要向服务器确认**，不能直接用本地缓存。

第二次请求：
  客户端 ──GET /index.html──→ 服务器
              If-None-Match: "abc123"
  客户端 ←──304 Not Modified── 服务器
              （无响应体）
// 如果没变，返回 `304`，**不传响应体**，节省带宽

如果内容变了：
  客户端 ←──200 OK────────── 服务器
              ETag: "def456"
              （返回新内容）
```


适用于HTML文件、API响应等需要及时更新的资源：

```
Cache-Control: no-cache
ETag: "abc123"
```

vary: Origin表示响应内容依赖请求的Origin头，如果同一个URL被不同 `Origin` 请求，浏览器会分别缓存。这在开发环境下常见，因为 Vite 的 dev server 需要处理 CORS。生产环境下，如果 CDN 配置不当，`vary: Origin` 可能导致缓存命中率下降。

缓存策略组合：

HTML这种资源推荐使用no-cache+ ETag每次协商，js/css推荐用max-age和immutable进行缓存。图片和字体也可以通过max-age+ETag进行缓存。API响应可以通过no-store或者短max-age + ETag进行缓存。

内存缓存是浏览器自己自动管理的，同一页面重复请求同一资源的时候就会直接从内存读取，关闭标签页就会失效。浏览器在收到一个资源的时候，会根据响应头决定存储在哪里。内存缓存的存储位置是RAM，而非DISK，读取速度很快，像js css 图片资源等高频使用的资源可以存，生命周期是当前标签页会话，关闭就失效。磁盘缓存的位置是硬盘，生命周期是由Cache-Control / Expires 决定，读取速度较快，像大文件，不常变动的资源可以缓存。service worker缓存存储的位置是Cache API，独立于HTTP缓存，需要手动控制生命周期，像PWA、离线资源可以存。

```
1. 内存缓存命中？
   ├─ 是 → 直接用（from memory cache）
   └─ 否 → 下一步

2. Service Worker 缓存命中？
   ├─ 是 → 用 SW 缓存
   └─ 否 → 下一步

3. 磁盘缓存命中？
   ├─ 强缓存有效？
   │   ├─ 是 → 直接用（from disk cache）
   │   └─ 否 → 协商缓存（发请求带 If-None-Match）
   │              ├─ 304 → 用本地
   │              └─ 200 → 下载新资源
   └─ 否 → 发请求下载
```


public, max-age=2592000, immutable 会让我们强缓存30天永不变，cf-cache-status: HIT表示这是Cloudflare CDN边缘节点已缓存。etag是协商的缓存标识，last-modified是最后的修改时间。这个组合会让浏览器把图片存入磁盘缓存，下次请求同一URL时候浏览器直接读磁盘，不发网络请求，状态码会显示 `200 (from disk cache)`。

### 标签页和v8

每个标签页其实不会分配一个独立的v8实例，v8实例是绑定在渲染进程RendererProcess上的，而不是直接绑定在标签页上。正确的关系是，一个v8实例可能服务多个标签页和iframe。在早期，确实一个标签页一个进程，现在策略已经变了，如果我们打开两个标签都访问同一个网站比如example.com，chrome可能让它们共享一个渲染进程，也就是共享一个v8实例，用来节省内存。如果网站嵌入了跨站iframe，chrome会为了跨站ifame单独创建一个渲染进程也就是OOPIF，跨进程iframe，一个标签页内部可能对于好几个渲染进程，有好几个v8实例。

可以查看chromium.org内看文档：离线处理架构OOPIFs：

```
这种技术允许页面的子框架由与父框架不同的进程来渲染。采用独立进程内嵌框架的做法主要是出于安全考虑，比如 Site Isolation 项目所提出的解决方案——通过这种方式，即使存在跨域内嵌框架，也能让某个渲染进程专注于处理某个特定的网站内容。不过，独立进程内嵌框架也是一种通用的机制，其应用范围不仅限于安全相关场景，还可以用于其他功能，例如 Chrome 应用程序中的<webview>标签。
```

OOPIF 要求 Chromium 的架构从“以标签页为中心”转向“**以 frame 为中心**”，因为一个 frame 在其生命周期内可能被渲染到不同进程中。浏览器进程现在直接追踪**完整的 frame 树**。`WebContents` 持有一个 `FrameTreeNode` 树，镜像当前页面的 frame 结构。每个 `FrameTreeNode` 包含该 frame 的元信息（名称、origin 等），其 `RenderFrameHostManager` 负责处理该 frame 的跨进程导航，并支持状态复制和消息路由

```
OOPIF 的渲染面临一个核心安全问题：**能合成一个纹理，就意味着能读取它的像素**（Cross Site Texture Stealing）。因此 OOPIF 的完整安全优势依赖于 **Übercompositor** 架构[](https://www.chromium.org/developers/design-documents/oop-iframes/oop-iframes-rendering/)。

在 Übercompositor 下：

- 每个渲染进程的子合成器生成 **CompositorFrame**，包含 quads 和纹理引用。
    
- 渲染进程通过 **IPC 将 CompositorFrame 发给浏览器进程**，而不是直接发给 GPU。
    
- 浏览器进程的父合成器与 GPU 进程交互，完成最终合成[](https://www.chromium.org/developers/design-documents/oop-iframes/oop-iframes-rendering/)。
    

纹理在不同进程间通过 **TextureMailbox** 传递所有权：一个进程用 `glProduceTextureCHROMIUM` 生成纹理，另一个进程用 `glConsumeTextureCHROMIUM` 消费它[](https://www.chromium.org/developers/design-documents/oop-iframes/oop-iframes-rendering/)。
```

这地方了解着玩玩就行，不用记忆太深，quad是合成器提交给gpu的最小绘制单元，可以理解为一个带纹理矩形，描述了哪块的纹理，用什么变换，绘制到屏幕哪个位置。渲染进程的合成器不直接操作 GPU，而是把一帧画面**拆解成一组 quad**，打包成 `CompositorFrame`，通过 IPC 发给浏览器进程或 GPU 进程。GPU 再按顺序把这些 quad 绘制出来，合成最终画面。

再深入挖掘就是架构图形栈知识了，不讲不讲...

### viz进程架构

viz是chromium渲染管线现代化的核心，主要为了解决GPU进程职责过重，站点隔离下的性能和安全瓶颈，以及图形栈的服务化需求。

在 Spectre 等侧信道攻击的背景下，跨站 iframe 需要严格的进程隔离（OOPIF）。VIZ将核心的合成和光栅化集中到独立进程中，使得来自不同渲染进程的合成器帧可以在一个安全的、收信任环境下统一聚合，从而支持“无额外性能代价的站点隔离”。Viz 是 Chromium 向 Mojo 服务化架构转型的一部分。它将合成、GL、命中测试、媒体等能力封装为标准服务，通过 Mojo 接口对外提供，使得 Browser 进程不再是唯一的核心枢纽，架构上更清晰、可维护性更强。

skia是chromium的底层2D图形库，负责我们几乎我们所有看到的像素绘制，从文字到图形、形状、路径、渐变、阴影、图层混合等。

```
Chromium 图形栈（从上层到下层）：

┌─────────────────────────────────┐
│  Blink（渲染引擎）              │  ← HTML/CSS 解析、布局、绘制指令
├─────────────────────────────────┤
│  cc / viz（合成器）             │  ← 图层合成、光栅化调度、GPU 提交
├─────────────────────────────────┤
│  Skia                           │  ← 2D 绘制：路径、文字、图片、混合
├─────────────────────────────────┤
│  GPU 驱动 / Vulkan / Metal / GL │  ← 硬件加速
└─────────────────────────────────┘
```
