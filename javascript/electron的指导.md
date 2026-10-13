---
date: 2026-12-10
tags:
  - javascript
  - electron
---
本笔记用于指导electron的理论和实践

## 架构

Electron 的 IPC是它的核心架构，主进程是一个node环境，用于管理窗口、声明周期、系统api，能够访问完整的nodejs api，比如说fs、path、child_process，不能直接操作dom，渲染进程是chromium环境，每个窗口一个，能够操作dom、跑前端代码，默认不能访问nodejs。

每个渲染进程在启动前，会加载一个预加载脚本，我们一般叫做preload.js，用于定义能访问的部分nodejs api，是主进程和渲染进程的bridge。

它提供的有三种模式，分别是渲染进程->主进程，有invoke和handle，渲染进程请求、主进程响应：


```js
// 主进程
const { ipcMain } = require('electron');

ipcMain.handle('read-file', async (event, filePath) => {
    const content = await fs.promises.readFile(filePath, 'utf-8');
    return content;   // 返回给渲染进程
});

// 预加载脚本
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('api', {
    readFile: (path) => ipcRenderer.invoke('read-file', path)
});

// 渲染进程
const content = await window.api.readFile('/path/to/file');
```

它的特点是promise风格，异步有返回值。

另一种是渲染进程->主进程，使用send和on完成，是单向的，渲染进程发，主进程收，但是不返回。


```js
// 主进程
ipcMain.on('log', (event, message) => {
    console.log('渲染进程说:', message);
    // 不返回
});

// 预加载
contextBridge.exposeInMainWorld('api', {
    log: (msg) => ipcRenderer.send('log', msg)
});

// 渲染进程
window.api.log('hello');
```

主进程单向到渲染进程：

```js
// 主进程
mainWindow.webContents.send('update', { version: '1.0.0' });
// 预加载
contextBridge.exposeInMainWorld('api', {
    onUpdate: (callback) => ipcRenderer.on('update', (event, data) => callback(data))
});
// 渲染进程
window.api.onUpdate((data) => {
    console.log('收到更新:', data);
});
```

ipc的底层是chromium的mojo ipc，mojo是chromium的ipc框架，管理二进制序列化、异步、跨进程，electron封装了mojo，能够提供ipcRenderer、ipcMain。

mojo在win32平台的底层传输是基于命名管道实现的，函数在 Windows 下会调用 `CreateNamedPipeW` 来创建管道。一个专门抓取 Chromium IPC 的工具也明确指出，它捕获的就是 **Named Pipe** 的流量

值得一提的是，从 Chrome 112 开始，Chromium 逐渐用新的 **ipcz** 协议替代了 Mojo Core。ipcz 为了优化性能，在 Windows 上可能会更多地使用**共享内存**来传递数据，而不是完全依赖命名管道，但这属于更底层的优化实现。

安全方面：

contextIsolation是默认开启的，渲染进程和预加载脚本上下文隔离。nodeInteration也是默认关闭的，渲染进程肯定不能访问node。sandbox沙箱默认开启，渲染进程在沙箱里。webSecurit默认开启，同源策略。

生命周期方面：

app.whenReady()是app初始化完成的时候，创建窗口。
window-all-closed是所有窗口都关闭，macos不退出。
before-quit: 退出前，可以拦截
will-quit:即将推出
quit:已退出
ready:app就绪
second-instance:第二个实例启动

窗口管理：

BrowserWindow:创建窗口
webPreferences:安全配置preload、contextisloation、nodeintergration
loadURL/loadFile：加载远程URL/本地文件
ready-to-show:窗口准备好显示，避免白屏
frame:false:无边框窗口，自定义标题栏
show:false + read-to-show 先隐藏，准备好在显示，可以消除白屏
transparent:true透明窗口
alwaysOnTop：置顶
webContents:窗口的web内容，可以`send`、`openDevTools`

preload脚本是渲染进程启动前，DOM加载前执行，能够访问`contextBridge`、`ipcRenderer`、部分 Node API。

继续更新
