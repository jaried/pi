---
id: WF-S1-01
label: wayfinder:research
parent: ../wayfinder-map.md
status: closed
assignee: tony
blockedBy: []
blocks: [WF-S1-03]
sprint: Sprint01
created: 2026-11-01
closed: 2026-11-01
---

# 研究：pi 的分段系统提示补丁会不会破坏字节前缀

## Question

pi 的哪些运行时机制会让「本次请求体」不再是「上一次 routed 请求体」的字节前缀？列出触发条件，并给出每条在源码中的定位。

具体要回答：

1. `before_agent_start` 返回的 `systemPromptOptions` 改动，最终以什么形式进入请求（替换首条 system，还是追加一条中途 system 消息）？追加时前缀是否仍逐字节一致？
2. `context_edit` 的 omission、`forceSystemPrompt`、工具集增删，各自在请求体的哪个位置制造分叉？
3. 上一次请求的 api key、headers、beta 列表变化是否影响被缓存的 token 前缀（还是只影响计费）？

## Context

- 入口：`packages/coding-agent/src/core/compaction/compaction.ts`（`buildSummarizationContext`、`completeSummarization`）
- 文档：`packages/coding-agent/docs/extensions.md` 的 `before_agent_start` 与 `systemPromptOptions` 段
- 相邻事实：`packages/ai/src/api/anthropic-messages.ts` 与 `openai-completions.ts` 如何序列化 system / tools
- 结论形态：一张「机制 → 是否破坏前缀 → 源码行号」的表，附一句每条的可规避方式

## Resolution

结论：DeepSeek 的两个模型行为不同，必须分开陈述。两种情况下「工具集变化」都必然全量失效，「消息内容变化」都从变化点起失效。

| 机制 | deepseek-v4-pro | deepseek-flash | 定位 |
|---|---|---|---|
| `before_agent_start` 的 section 补丁 | 追加为**中途** system 消息，前缀保持 | 折叠进**首条** system，token 0 改变 → 整段失效 | `types.ts:552-575`；`transcript.ts:105`（`collapseSystemMessages`）、`transcript.ts:115`（`resolveTranscript`）；`openai-completions.ts:1184` |
| 工具增删或定义变更 | 顶层 `tools` 变 → token 0 前失效 | 同上 | `transcript.ts:252-271`（`resolveTranscriptTools`，`support: "none"` 时 `requestTools` 取当前全集）；`openai-completions.ts:1216-1238` |
| `context_edit` omission / replacement | 从被改条目起失效 | 同上 | `session-manager.ts:172-182`（`ContextEditEntry`，`replacement: null` 为 omitted） |
| 压缩 checkpoint 替换一段 | 从 span 起点失效 | 同上 | `compaction.ts` 的 `firstKeptEntryId` 语义 |
| apiKey / headers / beta 列表 | 不影响被缓存的 token 前缀 | 同上 | `openai-completions.ts` 仅用它们构造请求头，不进入消息体 |

根因：`supportsMidConvoSystemMessages` 是折叠与否的开关（`openai-completions.ts:1184`），而 `packages/ai/catalog/providers/deepseek.ts` 的 `finalize` 只给 `deepseek-v4-pro` 打开它，`deepseek-flash` 走折叠路径。

对设计的三条硬约束：

1. `deepseek-flash` 上任何 section 补丁都让下一次请求全量 miss（system 是 token 0）。这不只影响压缩，影响每一轮；压缩请求只能如实继承这个代价，不能自行规避。
2. 重放必须基于**上一次实际发出的 payload**，而不是重新走一遍投影。投影是确定性的，但只要期间落了一次 section 补丁或工具变更，重新投影出的首条 system 就与上次不同。
3. 尾部指令作为最后一条 user 消息追加是安全的：DeepSeek 的 compat 未开 `requiresAssistantAfterToolResult`，toolResult 之后直接跟 user 消息不会被改写。

未验证项：真实命中率（`prompt_cache_hit_tokens`）由 WF-S1-02 实测，本 ticket 只给出机制层面的判定。
