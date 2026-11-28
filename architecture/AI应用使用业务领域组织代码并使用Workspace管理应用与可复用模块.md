---
date: 2026-11-23
tags:
  - architecture
  - ai
  - ddd
  - workspace
  - pnpm
---
现在前端发展之迅猛，Ai发展之快速，技术迭代之夸张，为了以后自己能够有口饭吃，我决定现在开坑前端ai应用开发这个坑，现在许多大厂的ai应用需求也比较大，agent时代之下，代码能力的差距被缩小，放大的是架构设计的经验，因此，我们笔记也应当重点讲解架构，而代码为辅。

对于一个AI应用来说，它的核心原则应该是：模块化、关注点分离、可扩展、易于维护、按功能领域组织代码，而非按技术类型。所以说，我们在设计AI应用的时候，workspace应当管理多个应用和可复用的模块，首先应当做到领域分离，解决代码应该如何组织、业务边界如何划分的问题、workspace使用monorepo的方式，解决多个应用、共享模块、依赖和构建任务统一管理的问题。

两者结合，比单纯按照 `components/`、`hooks/`、`utils/`、`services/` 这种技术类型划分更适合中大型 AI 应用。

一个所谓的AI应用具有许多的复杂业务概念，比如说对话、智能体、工作流、工具、知识库、模型、记忆、执行过程、评估等，这些领域存在着业务关系，但是又具有相对独立的生命周期，例如Agent可以使用多个Tool，也可以执行Workflow，workflow也可以调用Agent，也可以调用普通tool。

我们组织文件的时候，倾向于通过领域来组织：

```

ai-platform/
│
├── apps/
│   ├── web/                    # Web 应用
│   ├── desktop/                # Electron 桌面应用
│   ├── api/                    # API 服务
│   └── worker/                 # 后台任务执行器
│
├── packages/
│   ├── domains/
│   │   ├── conversation/       # 对话领域
│   │   ├── agent/               # Agent 领域
│   │   ├── workflow/            # 工作流领域
│   │   ├── knowledge/           # 知识库领域
│   │   ├── model/               # 模型管理领域
│   │   └── execution/           # 任务执行领域
│   │
│   ├── ai-core/
│   │   ├── llm/                 # LLM 抽象
│   │   ├── providers/           # 模型供应商适配
│   │   ├── tools/               # Tool 系统
│   │   ├── memory/              # Memory
│   │   └── orchestration/       # Agent 编排
│   │
│   ├── infrastructure/
│   │   ├── database/
│   │   ├── event-bus/
│   │   ├── observability/
│   │   └── storage/
│   │
│   ├── contracts/               # API DTO、事件契约
│   ├── ui/                      # 共享 UI 组件
│   ├── config/                  # 共享配置
│   └── utils/                   # 通用工具
│
├── tooling/
│   ├── eslint/
│   ├── typescript/
│   └── scripts/
│
├── pnpm-workspace.yaml
├── package.json
└── pnpm-lock.yaml
  
```

让一个业务领域相关的代码尽可能聚集在一起。这里有一个重要的设计原则，apps放可运行的应用，packages方可以被独立依赖的模块。

例如：

- `apps/web` 是 Web 应用。
    
- `apps/desktop` 是 Electron 应用。
    
- `packages/domains/agent` 是 Agent 领域模块。
    
- `packages/ai-core/llm` 是模型调用基础能力。
    
- `packages/contracts` 是前后端共享的数据契约。

现在的有个概念，一个AI平台通常包含我们上面说的产品模块，但是ai应用不是都趋向于微前端，因为**前端是一种前端组织和部署策略，不是 AI 应用架构的必然形态。**

假设我们要构建一个综合型 AI Platform，我会考虑下面的结构：

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1795523777476-dyymbbkl.webp)

微前端的划分尽量以业务领域划分，而不是对应的技术组件划分。
- Agent Studio 是一个完整业务应用。
- Workflow Studio 是一个完整业务应用。
- Button、Input、Modal 则应该是共享 UI 组件，不应该各自成为微前端。

这样才能让每个微前端有独立的业务边界。文件组织应该如下：

```
apps/
└── workflow-studio/
    ├── src/
    │   ├── app/
    │   ├── features/
    │   │   ├── canvas/
    │   │   ├── nodes/
    │   │   ├── execution/
    │   │   └── debugging/
    │   ├── components/
    │   ├── services/
    │   └── main.ts
    │
    ├── package.json
    ├── vite.config.ts
    └── tsconfig.json
```

它应该具备自己的开发服务器、构建配置、路由入口、业务状态管理、API Client、测试体系、CI/CD 管线、部署产物。比如说能够使用下面的应用：

```bash
# 独立启动
pnpm --filter @app/workflow-studio dev

# 独立构建
pnpm --filter @app/workflow-studio build

# 独立测试
pnpm --filter @app/workflow-studio test
```

这才是真正意义上的独立开发和交付。独立构建、独立部署不意味着必须使用独立 Git 仓库。single-spa 的文档也明确介绍了 Monorepo 中多个 SPA 独立构建和部署的方案。由于我个人常用的构建工具链是vite，所以我们可以应该去研究最主流的module federation，通过它我们可以管理前端模块的组合，我们甚至还可以通过microsoft agent framework来管理ai任务执行，两者通过稳定api和事件契约连接，做到非侵入式的实现。

前 Module Federation 已经提供 Vite 集成，官方推荐的 Vite 插件是 `@module-federation/vite`。它支持 Host / Remote、共享依赖和运行时加载。

我们要先理解Host和Remote，Module Federation 的核心。Host是主应用，管理路由和导航，加载Remote，管理着全局认证，提供共享依赖。Remote A 是 Agent Studio，B是 Workflow Studio，他们都可以独立构建和部署，Host 在运行时加载它，而不必重新构建整个平台。

monorepo的结构应当如下：

```
ai-platform/
│
├── apps/
│   ├── shell/                  # Host
│   ├── chat/                   # Remote
│   ├── agent-studio/           # Remote
│   ├── workflow-studio/        # Remote
│   ├── knowledge-studio/       # Remote
│   └── model-center/           # Remote
│
├── packages/
│   ├── design-system/
│   ├── auth-client/
│   ├── api-client/
│   ├── contracts/
│   ├── telemetry/
│   └── shared-config/
│
├── services/
│   ├── platform-api/
│   ├── agent-runtime/
│   └── workflow-runtime/
│
├── pnpm-workspace.yaml
└── package.json
```

这里 `apps/*` 每一个都是可以单独运行和构建的 Vite 项目。一个remote的配置示例如下：

```bash
pnpm add -D @module-federation/vite
```

Agent Studio 的 `vite.config.ts`：

```ts
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";
import { federation } from "@module-federation/vite";

export default defineConfig({
  plugins: [
    vue(),

    federation({
      name: "agent_studio",

      filename: "remoteEntry.js",

      exposes: {
        "./App": "./src/App.vue",
        "./routes": "./src/routes/index.ts",
      },

      shared: {
        vue: {
          singleton: true,
        },
        "vue-router": {
          singleton: true,
        },
        pinia: {
          singleton: true,
        },
      },
    }),
  ],
});
```

Module Federation 的 `shared` 用来协调共享依赖版本。Vue、Vue Router、Pinia 这类具有全局运行时状态的依赖尤其值得考虑单例共享；但不应该把所有依赖都设置成 singleton。

一个优秀的开发者，应当正确学习到这些内容：

|顺序|内容|重要程度|
|---|---|---|
|1|Host / Remote / Exposes / Remotes|必须|
|2|Shared Dependencies / Singleton|必须|
|3|Remote Manifest / Runtime Loading|必须|
|4|Dynamic Remote Registration|必须|
|5|Remote Version Management|必须|
|6|独立部署与回滚|必须|
|7|类型声明与契约管理|必须|
|8|SSR、跨框架集成|后续|

特别是 Remote Manifest 和动态注册。

如果需要，我们甚至还可以用Microsoft Agent Framework，来实现Agent和工作流，官方将其核心能力分成 Agents、Harness Agent、Workflows 和 Integrations。
https://learn.microsoft.com/en-us/agent-framework/overview/?utm_source=chatgpt.com

**Microsoft Agent Framework 的学习模块** ：

**Agent**

模型调用、指令、Tool Calling、会话和上下文管理。

**Tools / MCP**

函数工具、外部服务、MCP Server、工具权限和审批。

**Workflow**

Executor、Edge、Graph、条件分支、Fan-out / Fan-in。

**State / Persistence**

会话状态、工作流状态、Checkpoint、恢复执行。

**Human-in-the-loop**

人工确认、审批、暂停和继续。

**Observability**

执行事件、Tracing、日志、Token 和耗时统计。

我们可以把Agent Framework给集成到微前端平台，我们可以构建一个独立的agent studio微前端，并通过**ASP.NET Core 后端集成 Microsoft Agent Framework，让用户能够创建、配置、运行和调试 Agent。**

 