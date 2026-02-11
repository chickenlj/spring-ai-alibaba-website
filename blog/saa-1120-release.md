---
title: 支持 Agent Skills 和 Multi-agent Patterns，Spring AI Alibaba 1.1.2.0 版本发布！
tags: [Spring AI, Spring AI Alibaba, 正式版, "1.1.2", Agent Skills, 多智能体, 技能]
description: Spring AI Alibaba 1.1.2.0 正式发布，带来 Agent Skills 技能框架、多智能体并行执行、Graph 并行边与聚合策略等核心能力升级，推荐用于生产环境。
authors: [chickenlj]
date: "2026-02-03"
slug: saa-1120-release
category: article
---

Spring AI Alibaba **1.1.2.0** 正式发布。本版本将底层 Spring AI 升级至 **1.1.2**，并为 Agent 框架带来 **Agent Skills** 支持与**多智能体并行执行**等核心能力升级；Graph 支持**并行条件边**、**并行聚合策略**（AllOf / AnyOf）、异步工具执行与 `returnDirect` 等增强；同时包含序列化与 MergeStrategy 修复、Admin/UI 与文档更新。

<!-- truncate -->

## 核心特性概览

- **Agent**：ReactAgent 集成 **Agent Skills**，支持技能的渐进式披露；内置完善的智能体模式最佳实践（如 Routing、Supervisor、Handoffs、Custom Workflow 等），支持**并行子智能体执行**；新增 Flow Agent 统一 Hooks、`streamMessages` API、工具 `returnDirect`、ToolContextHelper 等。
- **Graph**：**并行条件边**、**并行分支聚合策略**（AllOf / AnyOf）、批量 `addEdge`、`interruptAfter` Hook、AgentToolNode **异步工具执行**、流式节点完整输出等。
- **Sandbox & Studio**：新增 **SAA Sandbox** 安全沙箱模块、内置 RenderTemplate、Studio CORS 可选配置。
- **质量与体验**：序列化与 MergeStrategy 修复、Agent 指令与 systemPrompt 等多项修复、Admin 包重命名为 `@spark-ai/flow`、CustomInputsControl 支持 `variableList` 等。

下文将重点介绍 **Agent Skills** 与 **多智能体模式** 两大核心功能，并给出 Release Notes 与版本分布表。

---

## Agent Skills（技能）

1.1.2.0 中，ReactAgent 集成了 **Agent Skills** 能力，支持以「技能」为单位做可复用指令与上下文的**渐进式披露**，在相关任务时由智能体自动发现并按需加载，从而降低 token 消耗、扩展能力规模。

### 核心概念

- **渐进式披露**：系统提示中先只注入技能列表（name、description、skillPath）；模型在需要某技能时调用 `read_skill(skill_name)` 加载完整 SKILL.md；再按需访问技能目录下的资源或使用与该技能绑定的工具。
- **Skill 目录结构**：每个技能一个子目录，必须包含 `SKILL.md`，可选 `references/`、`examples/`、`scripts/` 等。
- **SKILL.md**：YAML front matter 中需提供 `name`、`description`，正文为功能说明、使用方法与可用资源列表。

### 在 Agent 中使用

通过 **SkillRegistry**（如 `FileSystemSkillRegistry` 或 `ClasspathSkillRegistry`）管理技能，再使用 **SkillsAgentHook** 注册 `read_skill` 工具并将技能列表注入系统提示，即可在 ReactAgent 中使用：

```java
SkillRegistry registry = FileSystemSkillRegistry.builder()
    .projectSkillsDirectory(System.getProperty("user.dir") + "/skills")
    .build();

SkillsAgentHook hook = SkillsAgentHook.builder()
    .skillRegistry(registry)
    .build();

ReactAgent agent = ReactAgent.builder()
    .name("skills-agent")
    .model(chatModel)
    .saver(new MemorySaver())
    .hooks(List.of(hook))
    .build();

agent.call("请介绍你有哪些技能");
```

技能可与 Python、Shell 等工具配合：使用 **ShellToolAgentHook**、**PythonTool** 等，让 Agent 根据技能说明处理技能目录下的文件与脚本。通过 **groupedTools** 将工具与技能名绑定，可实现「仅当模型对该技能调用 `read_skill` 后，对应工具才加入当次请求」的渐进式工具披露。

更完整的 Skills 用法（含 ChatClient + SkillPromptAugmentAdvisor、生产环境自动重载、自定义系统提示模板等）请参阅文档：[Skills 技能](/docs/frameworks/agent-framework/tutorials/skills)。

---

## 多智能体模式

1.1.2.0 在工作流智能体上增强了**多智能体模式**能力：**LlmRouting**、**Supervisor** 等既可路由到单一子智能体，也可**一次选择多个子智能体并并行执行**，便于做多领域并行查询与结果汇总。下面简要归纳常见多智能体模式及适用场景。

### 为何需要 Multi-Agent

无论是单智能体还是多智能体，核心都在于**上下文管理**。多智能体在以下方面能带来明显收益：

- **上下文管理**：通过专职子智能体缩小单次上下文、降低延迟与 token 消耗；子智能体拥有隔离的上下文空间。
- **子智能体并行执行**：路由或主管模型可一次选出多个子智能体，并行执行后再做聚合。

**适合引入多智能体的典型场景**：单个 Agent 工具过多、部分任务需要专属知识/提示词或工具、需要按条件开启某子智能体能力等。

### 常见模式概览

| 模式 | 核心特点 | 典型场景 |
|------|----------|----------|
| **Supervisor（主管）** | 主智能体协调子智能体（Agent As Tool）；路由决策集中；子智能体无状态、可并行；上下文隔离 | 问题涉及多领域（如邮件、数据库）；集中式工作流；需并行多任务 |
| **Handoffs（交接）** | 通过状态变量（如 `current_step`）动态切换 Agent；多轮间状态持久化；每步可直接响应用户 | 多步骤对话、客服（先收集保修再退款）等 |
| **Skills（技能）** | 能力以「技能」形式按需披露；Prompt 驱动、轻量；适合单智能体扩展多能力 | 单智能体需多种技能、团队分技能开发维护 |
| **Routing（路由）** | 按领域拆解问题并分派；可单路或并行多子 Agent；需汇总结果 | 多领域问答、并行查多领域知识并综合 |
| **Sequential/Parallel/Loop（管道）** | 如Sequential模式是指前一节点输出即下一节点输入；可任意组合多种模式 | 固定流程、RAG 管道等 |
| **Custom Workflow** | 自定义图结构（顺序、分支、循环、并行）；可混合确定性逻辑与智能体节点 | 以上模式不满足时的完全自编排 |

1.1.2.0 中，**Supervisor** 与 **LlmRouting** 已支持「一次选择多个子智能体并并行执行」，配合 Graph 的**并行条件边**与**聚合策略（AllOf / AnyOf）**，可以更自然地实现多路并行与结果汇总。

更详细的模式说明、代码示例与最佳实践请参阅：[多智能体](/docs/frameworks/agent-framework/advanced/multi-agent) 及相关文档。

---

## Release Notes

* 完整 Release Notes 请参考：https://github.com/alibaba/spring-ai-alibaba/releases/tag/v1.1.2.0
* 版本详情与使用建议，请参考：https://java2ai.com/docs/versions

### 本版本核心升级摘要

**Agent**

- Agent Skills 集成于 ReactAgent（#3975），支持技能与工具的渐进式披露。
- 工作流智能体（LlmRouting、Supervisor）支持并行子智能体执行。
- Flow Agent 统一 Hooks 支持（#3983）。
- 新增 `streamMessages` API、带额外参数的 `invoke`/`call` 重载（#4031）。
- 工具 `returnDirect`（#4139）、ToolContextHelper（#4163）等。

**Graph**

- 并行条件边（#4027）、并行分支聚合策略 AllOf / AnyOf（#4028）。
- 批量 `addEdge`（#3938）、`interruptAfter` Hook（#3864）。
- AgentToolNode 异步工具执行（#4037/#3988）、流式节点完整输出（#3941）。

**Sandbox & Studio**

- SAA Sandbox 模块（#4069）、内置 RenderTemplate（#3935）、Studio CORS 可选（#4154）。

**修复与依赖**

- 序列化（#3969）、MergeStrategy（#4129）；Agent 指令、systemPrompt、selectorModel、jumpTo、NacosReactAgentBuilder 等多项修复；Admin 包重命名为 `@spark-ai/flow`（#3977）。
- **Spring AI**：1.1.2；**Spring AI Alibaba Extensions**：1.1.2.1。

**升级注意**：若使用 Admin Flow UI，请将依赖由 `@agentscope-ai/flow` 改为 `@spark-ai/flow`。

### Spring AI Alibaba 版本分布

| 版本 | 说明 |
|------|------|
| **1.1.2.0** | Agent Skills、多智能体并行、Graph 并行边与聚合策略、SAA Sandbox、多项修复与体验优化（当前推荐生产使用） |
| 1.1.x | 基于 Spring AI 1.1.x 的系列功能与修复 |
| 1.0.x | Spring AI Alibaba 1.0 GA 及后续 1.0 系列 |

---

## 总结

Spring AI Alibaba **1.1.2.0** 在保持与 Spring AI 1.1.2 对齐的基础上，重点增强了 **Agent Skills** 与 **多智能体模式** 两大方向：通过 Skills 实现可复用、渐进式披露的技能体系，通过 Supervisor/LlmRouting 的并行子智能体与 Graph 的并行边、聚合策略，为开发者提供多智能体开发最佳实践。同时，SAA Sandbox、序列化与 MergeStrategy 等修复以及 Admin/UI 的调整，进一步提升了生产可用性。

欢迎在项目中升级至 1.1.2.0，并参考官网文档与示例仓库进行开发与反馈。

- **官网**：[https://java2ai.com](https://java2ai.com)
- **GitHub**：[https://github.com/alibaba/spring-ai-alibaba](https://github.com/alibaba/spring-ai-alibaba)
- **示例仓库**：[https://github.com/springaialibaba/spring-ai-alibaba-examples](https://github.com/springaialibaba/spring-ai-alibaba-examples)
