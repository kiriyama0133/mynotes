---
date: 2025-11-01
tags:
  - plugin
  - keycloak
---
在使用keycloak的时候，为了能够清晰地了解客户端策略，我写了这篇笔记，用于阅读和了解细节。

| 名称                                                                                                                                                                         | 描述                                                                                                                                                                                                                                                                 |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [fapi-1-baseline 全局](https://localhost:18080/admin/master/console/#/web3d/realm-settings/client-policies/fapi-1-baseline/edit-profile)                                     | Client profile, which enforce clients to conform 'Financial-grade API Security Profile 1.0 - Part 1: Baseline' specification.                                                                                                                                      |
| [fapi-1-advanced 全局](https://localhost:18080/admin/master/console/#/web3d/realm-settings/client-policies/fapi-1-advanced/edit-profile)                                     | Client profile, which enforce clients to conform 'Financial-grade API Security Profile 1.0 - Part 2: Advanced' specification.                                                                                                                                      |
| [fapi-ciba 全局](https://localhost:18080/admin/master/console/#/web3d/realm-settings/client-policies/fapi-ciba/edit-profile)                                                 | Client profile, which enforce clients to conform 'Financial-grade API: Client Initiated Backchannel Authentication Profile' specification (Implementer's Draft ver1'). To satisfy FAPI-CIBA, both this profile and fapi-1-advanced global profile need to be used. |
| [fapi-2-security-profile 全局](https://localhost:18080/admin/master/console/#/web3d/realm-settings/client-policies/fapi-2-security-profile/edit-profile)                     | Client profile, which enforce clients to conform 'FAPI 2.0 Security Profile Final' specification.                                                                                                                                                                  |
| [fapi-2-message-signing 全局](https://localhost:18080/admin/master/console/#/web3d/realm-settings/client-policies/fapi-2-message-signing/edit-profile)                       | Client profile, which enforce clients to conform 'FAPI 2.0 Message Signing Final' specification.                                                                                                                                                                   |
| [oauth-2-1-for-confidential-client 全局](https://localhost:18080/admin/master/console/#/web3d/realm-settings/client-policies/oauth-2-1-for-confidential-client/edit-profile) | Client profile, which enforce confidential clients to conform 'OAuth 2.1' specification.                                                                                                                                                                           |
| [oauth-2-1-for-public-client 全局](https://localhost:18080/admin/master/console/#/web3d/realm-settings/client-policies/oauth-2-1-for-public-client/edit-profile)             | Client profile, which enforce public clients to conform 'OAuth 2.1' specification.                                                                                                                                                                                 |
| [fapi-2-dpop-security-profile 全局](https://localhost:18080/admin/master/console/#/web3d/realm-settings/client-policies/fapi-2-dpop-security-profile/edit-profile)           | Client profile, which enforce clients to conform 'FAPI 2.0 Security Profile Final' using DPoP specification.                                                                                                                                                       |
| [fapi-2-dpop-message-signing 全局](https://localhost:18080/admin/master/console/#/web3d/realm-settings/client-policies/fapi-2-dpop-message-signing/edit-profile)             | Client profile, which enforce clients to conform 'FAPI 2.0 Message Signing Final' with DPoP specification.                                                                                                                                                         |
| [saml-security-profile 全局](https://localhost:18080/admin/master/console/#/web3d/realm-settings/client-policies/saml-security-profile/edit-profile)                         | Client profile that enforces SAML clients to be secure.                                                                                                                                                                                                            |

## fapi-1 / fapi-2

FAPI（Financial-grade API）是一套基于 OAuth 2.0 和 OpenID Connect 构建的**高级安全规范**，专为金融、医疗等高敏感数据场景设计。它在普通 OIDC 的基础上增加了更严苛的防护，核心目标是确保“就算令牌被截获，也无法被滥用”。

fapi2引入了多项强制性安全要求，最关键的是**发送者约束令牌**，传统的bearer token缺陷是谁拿到令牌谁就可以用，一旦泄露就可能被攻击者冒充，fapi2的做法是要求令牌里必须与发情请求的客户端进行密码学绑定，即便令牌被窃取，攻击者也无法在自己的环境里使用。这种绑定可以用两种技术实现，mTLS、DPoP。

mtls叫做双向tls，在标准tls的基础上，客户端也出示自己的**X.509 证书**，让双方在建立连接前互相验明正身。客户端和服务器各自持有由收信任CA签发的证书，握手的时候双方互相验证证书的有效性，是否由可信的CA签发，是否过期，用途是否正确，都验证过后，建立加密通道，这个比较成熟，B2B，企业级金融接口对合规和审计要求比较严格的场景会去使用。

FAPI2利用了这个特性，在颁发令牌的时候将其与客户端的证书指纹绑定，后续请求必须使用同一张证书，令牌才有效。

DPOP也是绑定，方式是客户端生成密钥对，在应用层签发dpop proof，这个比较新。

### 实践

在 Nuxt (Nitro) 环境中实践 mTLS，核心思路和在任何 Node.js 后端服务中一样：**通过 Node.js 原生的 `https.Agent` 来加载客户端证书**。Nuxt 的 `$fetch` 底层基于 `ofetch`，它完全支持传递自定义的 `agent` 选项，因此你不需要一个特殊的“mTLS 库”。

将你的客户端证书 (`client.crt`)、私钥 (`client.key`) 以及用于验证服务端的 CA 证书 (`ca.crt`) 放在安全的位置（如 `server/certs/`）。

在 Nitro 的服务端工具函数中，创建一个配置好 mTLS 的 `https.Agent` 实例，在需要调用 YARP 或 Keycloak 的地方，将这个 `agent` 传递给 `$fetch`。


```ts
// server/utils/mtls.ts
import { Agent } from 'node:https'
import { readFileSync } from 'node:fs'
import { join } from 'node:path'

// 创建一个全局复用的 mTLS Agent，避免每次请求都重新加载证书
export const mtlsAgent = new Agent({
  cert: readFileSync(join(process.cwd(), 'server/certs/client.crt')),
  key: readFileSync(join(process.cwd(), 'server/certs/client.key')),
  ca: readFileSync(join(process.cwd(), 'server/certs/ca.crt')),
  // 生产环境务必设为 true，确保证书校验严格
  rejectUnauthorized: true, 
})
```

然后在你的 Nitro 路由中，像这样使用：

```ts
// server/api/protected.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  // 调用一个需要 mTLS 的下游服务（例如 YARP）
  const data = await $fetch(`${config.yarpBaseUrl}/api/some-resource`, {
    // 关键：将 mTLS agent 传递给请求
    agent: mtlsAgent, 
    headers: {
      // 依然可以携带 Bearer Token
      Authorization: `Bearer ${...}`,
    },
  })
  return data
})
```

继续更新...


## oauth-2-1-for-confidential-client

这是keycloak的内置的一套安全强制标准，它封装了 OAuth 2.1 草案中对**机密客户端**的最佳安全实践，启用它，就相当于一层保护措施。

这个策略通过内置的“执行器”自动落实以下规则：

- **强制使用 PKCE**：即使对于有机密客户端，OAuth 2.1 也强制要求使用 PKCE，以防止授权码拦截攻击。[](https://docs.redhat.com/zh-cn/documentation/red_hat_build_of_keycloak/26.2/html/server_administration_guide/executor)
    
- **禁用不安全的授权流程**：会自动**强制禁用隐式流（Implicit Flow）和混合流（Hybrid Flow）**，这些都是已被淘汰的不安全模式。[](https://docs.redhat.com/zh-cn/documentation/red_hat_build_of_keycloak/26.2/html/server_administration_guide/executor)
    
- **强制使用 HTTPS 重定向 URI**：要求所有 `redirect_uri` 必须以 `https` 开头，且**禁止使用通配符**，从根源上杜绝开放重定向漏洞。[](https://docs.redhat.com/zh-cn/documentation/red_hat_build_of_keycloak/26.2/html/server_administration_guide/executor)
    
- **强制更安全的客户端认证方式**：会要求客户端使用更安全的认证方法，如 `private_key_jwt` 或 `client_secret_jwt`，而不是简单的 `client_secret_basic`。

启用了这项措施之后，无需手动逐项检查PKCE，重定向URI等配置是否正确，keycloak会去自动强制执行，提前符合 OAuth 2.1 规范，能有效防御授权码注入、开放重定向等已知攻击向量。还能将安全规则集中定义在 Keycloak 侧，避免了在多处应用配置中遗漏安全设置的风险。

继续更新...

## 业务方面

### Organization

从 Keycloak 26.6 开始，`Organizations` 功能内置了 **`Organization Groups`**，专门用来解决担心的命名冲突问题。如果我们使用领域级别的groups来建立`/A/Engineering` 和 `/B/Engineering`，这两个路径必须是唯一的，容易冲突。而 `Organization Groups` 是**隔离**的：组织 A 的 `/Engineering/Backend` 和组织 B 的同名路径是完全独立的两套组，成员和标识互不影响。

这个可以容易的实现多租户和多层级的场景，这个对于ERP来说是必要的，我们可以在在 Keycloak 中为每个业务组织创建一个 `Organization`（例如 `org-a`），进入 Organization，在它的 **Groups** 标签页下创建内部层级（如 `/Engineering/Backend`）。

然后将用户分配到对应的组织及其内部组中。用户登录后，Token 的 `organization` 声明里会携带这些组信息。

创建组织的时候会让你填写名称、别名、域、重定向URL、描述。名称是给用户看的企业名称、别名是系统内部唯一标识符，必须是英文、数字、中划线比如kuayuan-zaowu。域是企业邮箱的后缀，比如说你可以填写kuayuan.com等，如果你开启自动加入功能，当用户用 `xxx@kuayuan.com` 注册时，Keycloak 会自动把他拉进这个组织，不用你手动邀请。

重定向URL是邀请成员时候的跳转地址，这个可以填写nuxt前端的首页或者特定页面，描述就随便写了，可以写什么核心3D打印之类的。

创建好了之后

