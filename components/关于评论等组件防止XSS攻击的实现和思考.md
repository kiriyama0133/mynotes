---
date: 2026-10-06
tags:
  - components
  - xss
---
跨站攻击脚本的防御中，有一个名言，叫做永远不要相信用户的输入，但是在评论组件里，如果在前端做过滤，就要知道一个现实，纯前端的字符过滤是防不住的，因为可以用postman调用，正所谓前端防君子，后端防小人。一个完整的xss业务流程必须是前端防呆、后端兜底、渲染层隔离的三层防御体系。

前端上，我们一般会去拦截普通用户的误操作或者低级的恶意输入来降低服务器压力，比如说限制评论最大字符数500字，防止超长payload攻击。然后是敏感词/黑名单过滤，这里可以使用正则或者敏感词库进行简单的过滤。

针对纯文本的评论，我们只允许纯文本，直接在前端把<和>转义了，或者剥离所有标签。

后端这里是防御xss的主场，前端传过来的任意数据，后端必须当做带毒来处理，如果允许用户发图文，比如说markdown渲染成html，后端不能用黑名单比如替换script，因为迪克可以用`<img src=x onerror=alert(1)>` 绕过。后端必须使用类似 `js-xss`（Node.js）或 `HTMLPurifier`（PHP）这种库，只允许特定的安全标签如 `<b>`, `<i>`, `<a>`, `<p>`，其他都删除。

**安全存储**：入库时，可以存储原始代码或清洗后的代码（视业务而定），但一定要确保存储的数据是经过验证的。

## 内容清洗

在前端渲染阶段，为了防止浏览器把数据当做代码去执行，所以前端渲染的时候决定会不会触发XSS，如果我们使用的是vue/react，那么就不要使用v-html或者dangerouslySetInnerHTML。vue可以使用`{{ comment.content }}`（或者 `v-text`）。Vue 的模板引擎也会自动进行 HTML 实体转义。

如果说我们的评论支持markdown或者富文本，必须渲染html标签，那么可以在后端返回的时候进行清洗，后端在返回API数据前，把HTML过滤干净，前端直接信任后端，用v-html或 `dangerouslySetInnerHTML` 渲染。

或者是让前端来清洗，后端没做清洗，或者是双保险，前端需要引入一个叫做DOMPurify的库：

```ts
import DOMPurify from 'dompurify';

// 假设这是用户输入的恶意评论
const dirtyInput = `这是一条评论 <img src="x" onerror="alert('你被XSS攻击了！')">`;

// 用 DOMPurify 清洗（它会剥离掉危险的 onerror 属性）
const cleanHtml = DOMPurify.sanitize(dirtyInput);

// 此时再交给 React 渲染就绝对安全了
<div dangerouslySetInnerHTML={{ __html: cleanHtml }} />
```

## CSP内容安全策略

除了代码层面的过滤，你还可以在服务器返回 HTML 的 HTTP 响应头中，配置 **CSP (Content Security Policy)**。这样，就只会允许加载当前域名下的脚本，禁止加载任何内敛脚本scrip或者onclick。及时用户把 `<script>alert(1)</script>` 插进去了，浏览器也会因为 CSP 规则直接拒绝执行。

