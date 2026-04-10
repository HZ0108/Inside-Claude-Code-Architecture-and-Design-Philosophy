# Inside Claude Code: Architecture & Design Philosophy

> An in-depth reverse-engineering and architectural analysis of Claude Code CLI.

---

**This is the English version.** For the Chinese version, please refer to [README_ZH.md](./README_ZH.md).

---

## Table of Contents

- [Inside Claude Code: Architecture & Design Philosophy](#inside-claude-code-architecture--design-philosophy)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Core Architectural Pillars](#core-architectural-pillars)
    - [1. Context & Token Management](#1-context--token-management)
    - [2. Multi-Tier Memory System](#2-multi-tier-memory-system)
    - [3. Agent Algorithm & Tool Orchestration](#3-agent-algorithm--tool-orchestration)
    - [4. Context Dynamic Injection](#4-context-dynamic-injection)
    - [5. Query Reliability Mechanism](#5-query-reliability-mechanism)
  - [Document Catalog](#document-catalog)
  - [Key Discoveries](#key-discoveries)
  - [Contributing](#contributing)
  - [Disclaimer](#disclaimer)

---

## Introduction

Claude Code is more than just a wrapper around the Claude API — it is a sophisticated, multi-agent system designed to operate autonomously within local filesystems and remote environments. By analyzing its source map, this project extracts the underlying TypeScript structures to reveal how Anthropic handles the hardest problems in AI agent development: **context window limits, long-term memory retention, concurrent tool execution, and reliability under partial failures.**

## Core Architectural Pillars

### 1. Context & Token Management

Claude Code's context management is built on a philosophy of **zero intentional information loss**. It operates through a multi-layered fallback mechanism to keep the LLM within its token budget while preserving critical task details:

- **Microcompact (Layer 1):** Clears stale tool results and utilizes API prompt caching (`cache_edits`) to manipulate context without breaking cache hits.
- **History Snip (Layer 2):** Selectively truncates older, irrelevant historical spans.
- **Session Memory Compaction (Layer 3):** Summarizes ongoing tasks incrementally into a markdown state, replacing raw history with a highly structured AI-generated summary when token limits hit ~90%.
- **Context Collapse (Experimental):** An advanced, paging-like strategy that archives chunks of conversation into metadata placeholders to prevent "prompt too long" (413) errors.

📄 **Full Report:** [EN/Context Management.pdf](./EN/Context%20Management.pdf)

---

### 2. Multi-Tier Memory System

Unlike traditional agents that rely on vector databases, Claude Code uses a pure **filesystem-based approach** (Markdown + YAML Frontmatter) to ensure absolute alignment with Git workflows and human readability.

- **Memory Types:** Categorized strictly into `user`, `feedback`, `project`, and `reference`.
- **Scopes & Synchronization:** Memories are divided into Private (`~/.claude`), Project (`.claude/`), and Team scopes. The Team Memory features a pull-push delta sync mechanism via API, complete with ETag optimistic locking and client-side secret scanning.
- **Staleness Tracking:** Memories are evaluated based on their `mtimeMs`. Information older than a specific threshold triggers automatic `<system-reminder>` warnings to the LLM to verify facts against current code before asserting them.

📄 **Full Report:** [EN/Memory System.pdf](./EN/Memory%20System.pdf)

---

### 3. Agent Algorithm & Tool Orchestration

The execution engine is driven by a powerful `AsyncGenerator` query loop and a dynamic Sub-Agent architecture.

- **Query Loop & Generators:** State machines built on `yield*` delegates that seamlessly stream text, tool uses, and telemetry (like TTFT) back to the UI layer.
- **StreamingToolExecutor:** A concurrency engine that evaluates if a tool is safe to run in parallel (e.g., `Read`, `Glob`) or requires exclusive execution (e.g., `Bash`, `Write`).
- **Sub-Agent Topologies:**
  - *LocalAgentTask:* Isolated background sub-processes with their own hierarchical `AbortControllers`.
  - *RemoteAgentTask:* Agents executing in remote environments (CCR) via MCP.
  - *Coordinator Mode:* A meta-agent pattern where a Coordinator manages multiple parallel worker agents for complex research and implementation pipelines.

📄 **Full Report:** [EN/Agent Algorithm Flow.pdf](./EN/Agent%20Algorithm%20Flow.pdf)

---

### 4. Context Dynamic Injection

A per-turn injection mechanism that feeds dynamic context — such as staleness warnings or malware-read alerts — into the conversation without permanently bloating the static system prompt. This allows runtime information to influence the model's behavior on a case-by-case basis while keeping the base prompt stable.

📄 **Full Report:** [EN/Context Dynamic Injection.pdf](./EN/Context%20Dynamic%20Injection.pdf)

---

### 5. Query Reliability Mechanism

Strategies for maintaining agent reliability under conditions of partial failure, ambiguous responses, or API-level issues. Covers retry logic, timeout handling, graceful degradation, and how Claude Code ensures progress even when individual queries fail or return unexpected results.

📄 **Full Report:** [EN/Query Reliability Mechanism.pdf](./EN/Query%20Reliability%20Mechanism.pdf)

---

## Document Catalog

| # | Topic | English | 中文 |
|---|-------|---------|------|
| 1 | Context & Token Management | [EN/Context Management.pdf](./EN/Context%20Management.pdf) | [上下文管理.pdf](./ZH/上下文管理.pdf) |
| 2 | Multi-Tier Memory System | [EN/Memory System.pdf](./EN/Memory%20System.pdf) | [记忆系统.pdf](./ZH/记忆系统.pdf) |
| 3 | Agent Algorithm & Tool Orchestration | [EN/Agent Algorithm Flow.pdf](./EN/Agent%20Algorithm%20Flow.pdf) | [Agent算法流程.pdf](./ZH/Agent算法流程.pdf) |
| 4 | Context Dynamic Injection | [EN/Context Dynamic Injection.pdf](./EN/Context%20Dynamic%20Injection.pdf) | [上下文动态注入方案.pdf](./ZH/上下文动态注入方案.pdf) |
| 5 | Query Reliability Mechanism | [EN/Query Reliability Mechanism.pdf](./EN/Query%20Reliability%20Mechanism.pdf) | [query可靠性机制.pdf](./ZH/query可靠性机制.pdf) |

---

## Key Discoveries

- **Prompt Caching Obsession:** Every layer of the system — from forking sub-agents to share the parent's prompt prefix, to Cached Microcompact — is aggressively optimized for Anthropic's Prompt Caching mechanism to reduce latency and cost.
- **The `<system-reminder>` Tag:** An invisible, per-turn injection mechanism used to feed dynamic context (like staleness warnings or malware-read warnings) without permanently bloating the static system prompt.
- **Token Economics:** Hardcoded token budgets for compression recovery (e.g., allocating strictly 5,000 tokens for restoring recently read files after a full compaction) to balance context retention with API limits.
- **Concurrency-Aware Tool Executor:** Not all tools are equal — the executor distinguishes between read-only parallelizable tools and stateful exclusive tools, preventing race conditions without serializing everything.
- **Filesystem-First Memory:** Choosing plain Markdown + YAML over a vector database was a deliberate trade-off: human-readable, Git-friendly, and zero additional infrastructure.

---

## Contributing

Contributions, insights, and corrections are welcome! If you've discovered an interesting pattern in the codebase or want to translate a section, please open a PR or Issue.

---

## Disclaimer

This repository is an independent, educational analysis. It is not affiliated with, officially supported by, or endorsed by Anthropic. All intellectual property regarding Claude Code belongs to Anthropic.
