---
date: 2026-08-12
tags:
  - browser
  - blob
---
Blob 本质上是一个**不可变的、类文件的原始数据容器**。它不直接暴露底层字节，只提供 `size`、`type` 以及读取方法（`text()`、`arrayBuffer()`、`stream()` 等）。

它的存储不是单一机制，而是由浏览器引擎在内存和磁盘之间动态调度的，以chromium的实现为例，有一个BlobMemoryController专门来管理配额，当Blob数据量小、内存充足，优先放在内存里传输，当数据较大或者内存紧张的时会把Blob数据分页到磁盘，用文件来暂存。

有一个关键点，渲染进程再把数据交给浏览器进程之前，会临时持有这份数据，如果创建大量的blob而js侧不释放引用的话，渲染进程可能会因为等待传输而内存暴涨甚至可能OOM，所以Blob存在内存还是磁盘取决于数据大小和浏览器调度。但是从开发者的视角来说，他就是一个黑盒的二进制载体。

## mime类型设置文件类型

我们一般**通过构造函数的 `type` 选项来设置，但这个类型是“声明”的，不是“探测”的。**

`new Blob([data], { type: 'text/plain' })` 里的 `type` 会直接成为 `blob.type` 的值。如果未知，`blob.type` 返回空字符串。

当这个 Blob 被转成 `blob:` URL 并被请求时，浏览器返回的响应头 `Content-Type` 就会取自 `blob.type`。

需要警惕的是：**浏览器不会去读取字节流来验证你声明的类型**。你把一个 PNG 文件重命名为 `.txt` 再读进来，如果构造时传 `type: 'text/plain'`，`blob.type` 就是 `text/plain`，尽管内容其实是 PNG。所以 `blob.type` 只适合作为“提示”，不能当作安全校验依据。

## 用于触发下载和暂存文件

这两个是blob最常用的两个场景，触发下载是创建blob-> URL.createObject(blob)生成随机URL，创建一个隐藏的a标签，然后设置href为这个url，download属性告诉浏览器这个是要下载的，不是导航，必须revoke，不然blob数据会一直被url引用着无法回收。

暂存文件是更为宽泛的场景，你可以把任意二进制数据从fetch拿到的response.blob( )比如、用户选择文件得到的flie对象、或者自己拼的字符串ArrayBuffer，包成Blob，然后在内存里持有，切片slice、或者转成URL给img、video预览，它相当于一个浏览器内的临时文件句柄，不需要真正写入磁盘就可以被页面消费。

继续更新...