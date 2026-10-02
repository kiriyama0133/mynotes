```ts
const sum = ref(0)
const dogList = reactive(['https://images.dog.ceo/breeds/pembroke/n02113023_6269.jpg'])
// 方法：改变 sum 值
async function change() {
  const result = await axios.get('https://dog.ceo/api/breed/pembroke/images/random')
  console.log(result.data)
  dogList.push(result.data.message)
}

```

以上可以完成一个简单的get请求

## Rest API

​	REST 是 **"Representational State Transfer"** 的缩写，中文翻译为“表述性状态转移”。它是一种用于构建 **Web 服务** 的架构风格，由 Roy Fielding 在他的博士论文中首次提出。REST API 是基于 REST 架构风格设计的 **应用程序接口（API）**，它允许客户端和服务器通过 HTTP 协议进行通信。REST API 的核心思想是将系统中的资源（如用户、文章、商品等）通过 URL 表示，并使用标准的 HTTP 方法（如 GET、POST、PUT、DELETE）对这些资源进行操作。

## 客户端封装

#### **封装好的客户端的作用**

1. **简化 API 调用**：

   - 开发者无需手动构建 HTTP 请求（如设置 URL、请求头、处理响应等），只需调用封装好的方法即可。例如：

     

     ```javascript
     const articles = await $strapi.find('articles');
     ```

     这比手动使用fetch或axios更加简洁。

2. **内置功能**：

   - 封装好的客户端通常会内置一些常用功能，例如：
     - 自动处理认证（如附加 Token 到请求头）。
     - 自动解析响应数据。
     - 提供错误处理机制。

3. **提高开发效率**：

   - 开发者可以专注于业务逻辑，而无需关心底层的 API 调用细节。

### **Strapi 和 Pinia 的区别**

#### **1. Strapi 是一个后端 CMS（内容管理系统）**

- **作用**：
  - Strapi 是一个 **Headless CMS**，用于管理和存储内容（如文章、用户、商品等）。
  - 它提供了 REST API 或 GraphQL API，前端可以通过这些 API 获取或操作后端的数据。
  - Strapi 的主要职责是**作为后端服务**，处理**数据存储、验证、权限管理**等。
- **使用场景**：
  - 你需要一个后端来存储和管理数据。
  - 你需要通过 API 将数据提供给前端应用（如 Vue.js、React、Angular 等）。
  - 例如：构建博客、电子商务网站、内容管理平台等。
- **特点**：
  - 基于 Node.js，支持自定义扩展。
  - 提供直观的管理界面，方便非技术人员管理内容。
  - 支持用户认证和权限管理。

#### **2. Pinia 是一个前端状态管理工具**

- **作用**：
  - Pinia 是 Vue.js 的状态管理库，用于在前端应用中管理组件之间共享的状态。
  - 它的主要职责是存储和管理前端的状态（如用户登录信息、购物车数据等），并在组件之间实现状态共享和通信。
- **使用场景**：
  - 你需要在多个组件之间共享数据。
  - 你需要管理前端的状态（如用户登录状态、主题设置、购物车内容等）。
  - 例如：在一个电商网站中，购物车数据需要在多个页面之间共享。
- **特点**：
  - 轻量、简单，专为 Vue.js 设计。
  - 支持响应式数据和模块化设计。
  - 可以与 Vue DevTools 集成，方便调试。