# Inside Claude Code：架构与设计哲学

> 对 Anthropic Claude Code CLI 的深度逆向工程与架构分析。

---

## 目录

- [Inside Claude Code：架构与设计哲学](#inside-claude-code架构与设计哲学)
  - [目录](#目录)
  - [引言](#引言)
  - [核心架构支柱](#核心架构支柱)
    - [1. 上下文与 Token 管理](#1-上下文与-token-管理)
    - [2. 多层记忆系统](#2-多层记忆系统)
    - [3. Agent 算法与工具编排](#3-agent-算法与工具编排)
    - [4. 上下文动态注入](#4-上下文动态注入)
    - [5. Query 可靠性机制](#5-query-可靠性机制)
  - [文档目录](#文档目录)
  - [核心发现](#核心发现)
  - [贡献指南](#贡献指南)
  - [免责声明](#免责声明)

---

## 引言

Claude Code 不仅仅是对 Claude API 的简单封装——它是一个精密的多 Agent 系统，设计用于在本地文件系统和远程环境中自主运行。通过分析其源代码映射（source map），本项目提取底层的 TypeScript 结构，揭示 Anthropic 如何解决 AI Agent 开发中最棘手的问题：**上下文窗口限制、长期记忆保留、并发工具执行，以及部分失败场景下的可靠性保障。**

---

## 核心架构支柱

### 1. 上下文与 Token 管理

Claude Code 的上下文管理系统建立在**零主动信息丢失**的哲学之上。它通过多层降级机制，在将 LLM 保持在 Token 预算内的同时，保留关键任务细节：

- **微压缩（Layer 1）：** 清除过时的工具结果，利用 API 提示缓存（`cache_edits`）操作上下文，同时不破坏缓存命中。
- **历史裁剪（Layer 2）：** 选择性截断较早的、无关的历史片段。
- **会话记忆压缩（Layer 3）：** 当 Token 限额达到约 90% 时，将正在进行的任务增量摘要为 Markdown 状态，用高度结构化的 AI 生成摘要替换原始历史记录。
- **上下文折叠（实验性）：** 一种高级的分页策略，将对话块归档到元数据占位符中，防止"prompt too long"（413）错误。

📄 **完整报告：** [上下文管理.pdf](./ZH/上下文管理.pdf)

---

### 2. 多层记忆系统

与依赖向量数据库的传统 Agent 不同，Claude Code 采用纯**文件系统方案**（Markdown + YAML Frontmatter），确保与 Git 工作流和人类可读性的绝对对齐。

- **记忆类型：** 严格分类为 `user`、`feedback`、`project` 和 `reference` 四种。
- **作用域与同步：** 记忆分为私有（`~/.claude`）、项目级（`.claude/`）和团队级作用域。团队记忆通过 API 实现拉取-推送增量同步机制，包含 ETag 乐观锁和客户端密钥扫描。
- **时效追踪：** 记忆基于 `mtimeMs` 进行评估。超过特定阈值的历史信息会触发自动的 `<system-reminder>` 警告，提示 LLM 在断言前对照当前代码验证事实。

📄 **完整报告：** [记忆系统.pdf](./ZH/记忆系统.pdf)

---

### 3. Agent 算法与工具编排

执行引擎由强大的 `AsyncGenerator` 查询循环和动态子 Agent 架构驱动。

- **查询循环与生成器：** 基于 `yield*` 委托构建的状态机，无缝地将文本、工具调用和遥测数据（如 TTFT）流式传回 UI 层。
- **StreamingToolExecutor：** 一个并发引擎，评估工具是否可安全并行运行（如 `Read`、`Glob`），或需要独占执行（如 `Bash`、`Write`）。
- **子 Agent 拓扑：**
  - *LocalAgentTask：* 拥有独立层级 `AbortControllers` 的隔离后台子进程。
  - *RemoteAgentTask：* 通过 MCP 在远程环境（CCR）中执行的 Agent。
  - *Coordinator Mode：* 一种元 Agent 模式，由 Coordinator 管理多个并行工作 Agent，用于复杂的研究和实施流水线。

📄 **完整报告：** [Agent算法流程.pdf](./ZH/Agent算法流程.pdf)

---

### 4. 上下文动态注入

一种逐轮注入机制，将动态上下文——如时效警告或恶意文件读取警告——注入到对话中，而不永久撑大静态系统提示。这允许运行时信息在逐案基础上影响模型行为，同时保持基础提示的稳定性。

📄 **完整报告：** [上下文动态注入方案.pdf](./ZH/上下文动态注入方案.pdf)

---

### 5. Query 可靠性机制

在部分失败、响应模糊或 API 级问题条件下维持 Agent 可靠性的策略。涵盖重试逻辑、超时处理、优雅降级，以及 Claude Code 如何在单个查询失败或返回意外结果时仍确保任务推进。

📄 **完整报告：** [query可靠性机制.pdf](./ZH/query可靠性机制.pdf)

---

## 文档目录

| # | 主题 | English | 中文 |
|---|------|---------|------|
| 1 | 上下文与 Token 管理 | [Context Management.pdf](./EN/Context%20Management.pdf) | [上下文管理.pdf](./ZH/上下文管理.pdf) |
| 2 | 多层记忆系统 | [Memory System.pdf](./EN/Memory%20System.pdf) | [记忆系统.pdf](./ZH/记忆系统.pdf) |
| 3 | Agent 算法与工具编排 | [Agent Algorithm Flow.pdf](./EN/Agent%20Algorithm%20Flow.pdf) | [Agent算法流程.pdf](./ZH/Agent算法流程.pdf) |
| 4 | 上下文动态注入 | [Context Dynamic Injection.pdf](./EN/Context%20Dynamic%20Injection.pdf) | [上下文动态注入方案.pdf](./ZH/上下文动态注入方案.pdf) |
| 5 | Query 可靠性机制 | [Query Reliability Mechanism.pdf](./EN/Query%20Reliability%20Mechanism.pdf) | [query可靠性机制.pdf](./ZH/query可靠性机制.pdf) |

---

## 核心发现

- **对 Prompt Caching 的极致追求：** 系统的每一层——从 fork 子 Agent 共享父级提示前缀，到缓存感知的微压缩——都针对 Anthropic 的 Prompt Caching 机制进行了激进优化，以降低延迟和成本。
- **`<system-reminder>` 标签：** 一种无形的逐轮注入机制，用于向 LLM 传递动态上下文（如时效警告或恶意读取警告），而不永久增加静态系统提示的体积。
- **Token 经济学：** 压缩恢复时的 Token 预算硬编码（例如，全量压缩后用于恢复最近读取文件的 Token 严格限定为 5,000 个），以平衡上下文保留与 API 限额。
- **并发感知的工具执行器：** 并非所有工具都是平等的——执行器区分只读可并行工具与有状态的独占工具，在不序列化所有操作的前提下防止竞态条件。
- **文件系统优先的记忆方案：** 选择纯 Markdown + YAML 而非向量数据库是一种深思熟虑的权衡：人类可读、Git 友好、无需额外基础设施。

---

## 贡献指南

欢迎提交贡献、见解和修正！如果你在代码库中发现了有趣的模式，或想要翻译某个章节，请提交 PR 或 Issue。

---

## 免责声明

本仓库为独立的、教育性质的分析作品。它与 Anthropic 没有关联，未获得 Anthropic 的官方支持或认可。Claude Code 相关的所有知识产权均归 Anthropic 所有。
