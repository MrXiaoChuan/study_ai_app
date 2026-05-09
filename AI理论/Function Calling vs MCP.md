---
title: Function Calling vs MCP
aliases:
  - 函数调用 vs MCP
  - Tool Calling vs MCP
tags:
  - AI/LLM
  - AI/Agent
  - Protocol
  - Tooling
status: draft
created: 2026-05-09
---

# 概览
- **Function Calling（函数调用/工具调用）**：在一次模型推理中，把“可用工具”以结构化方式（常见为 JSON Schema）提供给模型，让模型返回“要调用哪个函数 + 参数”，由宿主程序执行并把结果回填给模型继续推理。
- **MCP（Model Context Protocol）**：一种把“工具/资源/提示词模板”等能力以统一协议暴露为服务的标准，让宿主/客户端以一致的方式发现、调用、鉴权并把结果提供给模型使用。

一句话区分：Function Calling 更像“模型输出结构化调用指令”；MCP 更像“工具与上下文能力的标准化总线/适配层”。

> [!note] 关系
> 两者不是互斥的：常见组合是 **模型用 Function Calling 选择工具**，而工具本身通过 **MCP Server** 提供（宿主端用 MCP Client 去调）。

# 核心差异
## 关注点不同
- Function Calling 解决“**模型怎么表达要做的动作**”（哪个函数、哪些参数、何时调用）。
- MCP 解决“**工具能力怎么被标准化提供与连接**”（发现、连接、权限、数据形态、生命周期、跨工具一致性）。

## 对比表
| 维度 | Function Calling | MCP |
| --- | --- | --- |
| 主要对象 | 模型输出格式与工具选择 | 工具/资源能力的协议化暴露与连接 |
| 提供方 | 通常由 LLM Provider/SDK 定义（实现细节有差异） | 协议标准 + MCP Server 实现 |
| 工具发现 | 多为“请求里直接给工具列表” | 可通过协议获取 server 提供的 tools/resources/prompts |
| 状态/会话 | 往往偏“单次请求/宿主自管状态” | 更自然支持长连接/会话与能力边界（依实现而定） |
| 扩展性 | 工具多时上下文膨胀、schema 管理成本高 | 通过 server 侧拆分能力、统一协议降低接入成本 |
| 权限/隔离 | 依赖宿主实现（白名单、沙箱、审计） | 协议层可承载鉴权/权限模型（仍需宿主落实执行） |
| 典型产物 | `{"name": "...", "arguments": {...}}` | tools/resources/prompts 的标准化接口与调用结果 |

# 典型架构
```mermaid
flowchart LR
  U[User] --> H[Host / Orchestrator]
  H --> M[LLM]
  M -->|Function Call: tool + args| H
  H -->|Execute| T[Local Tools]
  H -->|MCP Client| S[MCP Server(s)]
  S --> EXT[(DB / SaaS / FS / APIs)]
  T --> H
  S --> H
  H --> M
  M --> H
```

# 什么时候用哪个
> [!tip] 快速选择
> - 只需要“让模型调用少量自定义函数”，不想引入额外协议与服务：优先 **Function Calling**。
> - 你有很多工具/资源/提示词，需要可复用接入、统一鉴权、跨项目复用、团队协作：优先 **MCP**（或把工具逐步迁到 MCP）。
> - 你在做 Agent 平台：通常 **两者都用**（Function Calling 做决策输出；MCP 做工具生态与连接层）。

## 更适合 Function Calling 的场景
- 工具很少（1～10 个）且稳定，参数简单，调用频率不高。
- 工具执行环境完全在你可控的后端（比如同一个服务进程内的函数）。
- 你主要挑战在“模型怎么填参数/怎么选工具”，而不是“工具怎么接入/怎么治理”。

## 更适合 MCP 的场景
- 工具很多、异构（数据库/工单/Jira/GitHub/文件系统/内部服务），需要统一接入与复用。
- 需要清晰的能力边界：谁能访问哪些资源、审计、限流、隔离、最小权限。
- 多模型/多应用共享同一套工具与资源能力，希望降低重复开发。

# 实践要点（踩坑清单）
> [!warning] 常见问题
> - **工具爆炸**：把所有工具都塞进一次调用上下文会造成 token 浪费与选择困难；用 MCP/分组路由/按需检索工具清单更稳。
> - **Schema 漂移**：函数签名变更导致历史提示词或调用失败；要做版本化（例如 `tool_v2`）或兼容层。
> - **权限错配**：模型“能提议”调用不等于“应当执行”；宿主必须做显式授权与校验（参数校验、资源范围、审批）。
> - **结果不可控**：工具返回过长/噪声大影响后续推理；要做摘要、结构化与大小限制。

# 最小心智模型
- Function Calling = “把模型输出变成可执行的结构化指令（函数名 + 参数）”
- MCP = “把外部能力包装成标准化可插拔模块（工具/资源/提示词）供宿主连接”

# 示例（简化）
## Function Calling 输出（示意）
```json
{
  "name": "search_docs",
  "arguments": {
    "query": "MCP authentication",
    "top_k": 5
  }
}
```

## MCP 调用（概念步骤）
1. 宿主连接某个 MCP Server（本地或远端）
2. 获取其暴露的 tools/resources/prompts 列表（可选）
3. 根据模型决策调用对应 tool，并把结果回填给模型

# 关联笔记
- [[Agent 架构]]
- [[工具调用（Tool Calling）]]
- [[RAG]]
- [[权限与最小权限原则]]
