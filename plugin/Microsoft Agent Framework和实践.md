---
date: 2026-11-25
tags:
  - maf
  - agent
  - plugin
---
maf是 semantic kernel 和 autogen 的继任者，由微软的同一批团队打造，于 2026 年 4 月正式 GA。它并非从零开始的新框架，而是将 AutoGen 的**简洁 Agent 抽象**与 SK 的**企业级工程底座**（会话状态管理、类型安全、执行过滤器、遥测等）深度融合的产物。

MAF的官方文档将其能力归纳为了四个主要领域，分别是Agent，个体智能体，负责处理输入、工具调用包括MCP服务器，生成响应，MAF支持多种模型提供商，并抽象出统一的agent基类。
harness agent，这是一个开箱即用的，有主见的agent，内置了规划与任务追踪、上下文压缩、文件访问与记忆、工具审批等能力。
workflows，基于图的工作流，用于以显示执行路径连接多个Agent和函数，管理顺序、并发、群聊和交接等编排模式。
integrations，连接模型提供商、工具、中间件、评估服务等一切。

maf本身支持了c#和python，官方的入门显示，python需要3.10+，.net则需要受支持版本。maf的核心设计原则是统一，所有的agent都继承自同一个基类，无论底层使用openai、azure openai、anthropoic还是本地ollama模型，我们的重点都会放在agent抽象接口上，而不是纠结于特定模型提供商的sdk细节。

