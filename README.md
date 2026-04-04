# Decoding Claude Code: Architecture & Design Philosophy> An in-depth reverse-engineering and architectural analysis of Anthropic's Claude Code CLI.

  
# Decoding Claude Code: Architecture & Design Philosophy

> An in-depth reverse-engineering and architectural analysis of Anthropic's Claude Code CLI.

This repository provides a comprehensive analysis of the internal mechanics, agent algorithms, context management strategies, and memory systems powering Claude Code. It aims to uncover the design philosophy behind building a production-ready, highly complex LLM-driven CLI tool.

## 📖 Table of Contents

- [Introduction](#introduction)
- [Core Architectural Pillars](#core-architectural-pillars)
  - [1. Context & Token Management](#1-context--token-management)
  - [2. Multi-Tier Memory System](#2-multi-tier-memory-system)
  - [3. Agent Algorithm & Tool Orchestration](#3-agent-algorithm--tool-orchestration)
- [Directory Structure](#directory-structure)
- [Key Discoveries](#key-discoveries)
- [Contributing](#contributing)
- [Disclaimer](#disclaimer)

## 💡 Introduction

Claude Code is more than just a wrapper around the Claude API; it is a sophisticated, multi-agent system designed to operate autonomously within local filesystems and remote environments. By analyzing its source map, this project extracts the underlying TypeScript structures to reveal how Anthropic handles the hardest problems in AI agent development: **context window limits, long-term memory retention, and concurrent tool execution.**

## 🏗 Core Architectural Pillars

### 1. Context & Token Management ([Read Full Report](./docs/context-management.md))
The context management system is built on a philosophy of *zero intentional information loss*. It operates through a multi-layered fallback mechanism to keep the LLM within its token budget while preserving critical task details:
- **Microcompact (Layer 1):** Clears stale tool results and utilizes API prompt caching (`cache_edits`) to manipulate context without breaking cache hits.
- **History Snip (Layer 2):** Selectively truncates older, irrelevant historical spans.
- **Session Memory Compaction (Layer 3):** Summarizes ongoing tasks incrementally into a markdown state, replacing raw history with a highly structured AI-generated summary when token limits hit ~90%.
- **Context Collapse (Experimental):** An advanced, paging-like strategy that archives chunks of conversation into metadata placeholders to prevent "prompt too long" (413) errors.

### 2. Multi-Tier Memory System ([Read Full Report](./docs/memory-system.md))
Unlike traditional agents that rely on vector databases, Claude Code uses a pure filesystem-based approach (Markdown + YAML Frontmatter) to ensure absolute alignment with Git workflows and human readability.
- **Memory Types:** Categorized strictly into `user`, `feedback`, `project`, and `reference`.
- **Scopes & Synchronization:** Memories are divided into Private (`~/.claude`), Project (`.claude/`), and Team scopes. The Team Memory features a pull-push delta sync mechanism via API, complete with ETag optimistic locking and client-side secret scanning.
- **Staleness Tracking:** Memories are evaluated based on their `mtimeMs`. Information older than a specific threshold triggers automatic `<system-reminder>` warnings to the LLM to verify facts against current code before asserting them.

### 3. Agent Algorithm & Tool Orchestration ([Read Full Report](./docs/agent-algorithm.md))
The execution engine is driven by a powerful `AsyncGenerator` query loop and a dynamic Sub-Agent architecture.
- **Query Loop & Generators:** State machines built on `yield*` delegates that seamlessly stream text, tool uses, and telemetry (like TTFT) back to the UI layer.
- **StreamingToolExecutor:** A concurrency engine that evaluates if a tool is safe to run in parallel (e.g., `Read`, `Glob`) or requires exclusive execution (e.g., `Bash`, `Write`).
- **Sub-Agent Topologies:**
  - *LocalAgentTask:* Isolated background sub-processes with their own hierarchical `AbortControllers`.
  - *RemoteAgentTask:* Agents executing in remote environments (CCR) via MCP.
  - *Coordinator Mode:* A meta-agent pattern where a Coordinator manages multiple parallel worker agents for complex research and implementation pipelines.

## 📂 Directory Structure

```text
decoding-claude-code/
├── docs/
│   ├── context-management.md    # Deep dive into Token/Context Compaction
│   ├── memory-system.md         # Analysis of persistent and session memory
│   └── agent-algorithm.md       # Breakdown of the Query Loop and Tool Executor
├── diagrams/                    # Architectural flowcharts and state diagrams
└── README.md
```


🚀 Key Discoveries

Prompt Caching Obsession: Every layer of the system—from forking sub-agents to share the parent's prompt prefix, to Cached Microcompact—is aggressively optimized for Anthropic's Prompt Caching mechanism to reduce latency and cost.

The <system-reminder> Tag: An invisible, per-turn injection mechanism used to feed dynamic context (like staleness warnings or malware read warnings) without permanently bloating the static system prompt.

Token Economics: Hardcoded token budgets for compression recovery (e.g., allocating strictly 5,000 tokens for restoring recently read files after a full compaction) to balance context retention with API limits.

🤝 Contributing

Contributions, insights, and corrections are welcome! If you've discovered an interesting pattern in the codebase or want to translate a section, please open a PR or Issue.

⚠️ Disclaimer

This repository is an independent, educational analysis. It is not affiliated with, officially supported by, or endorsed by Anthropic. All intellectual property regarding Claude Code belongs to Anthropic.


