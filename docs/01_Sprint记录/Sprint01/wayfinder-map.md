---
labels: [wayfinder:map]
sprint: Sprint01
status: closed
created: 2026-11-01
closed: 2026-11-01
---

# 压缩成本：从本地摘要到前缀命中

## Destination

锁定 pi 压缩请求构造的深化设计：不变式是「压缩请求必须是上一次实际发出的 payload + 一条尾部 user 指令」，在 `deepseek-v4-pro` 与 `deepseek-flash` 两种投影下都成立。交付物是设计文档加钉住字节前缀的失败测试，不含生产实现。不引入 summarizer 抽象层。

## Notes

- 领域：pi harness 的上下文压缩成本（`packages/coding-agent/src/core/compaction/`），对照实现是 DSH 的 `packages/compaction/compaction-basic`。
- 每次会话开工前读：`packages/coding-agent/docs/compaction.md`、`packages/ai/src/api/openai-completions.ts`、`packages/ai/src/utils/transcript.ts`、`packages/ai/src/types.ts`。
- 通用词汇用 `codebase-design`；设计题先 `grilling`，术语落 `domain-modeling`。
- **范围：只做 DeepSeek**（`deepseek-v4-pro` 与 `deepseek-flash` 的差异必须都在测试面里）。
- 既定立场（codebase-design）：单 provider 只有一个实现，**不引入 summarizer seam / adapter**。深度来自「请求构造」这一处的不变式与它的测试，不来自抽象层。
- 本仓库没有 GitHub Issue 流转，`docs/` 是唯一 tracker。

## Decisions so far

<!-- 一行一个已关闭 ticket，点击链接读细节 -->

- [研究：pi 的分段系统提示补丁会不会破坏字节前缀](tickets/WF-S1-01-prefix-stability.md)：v4-pro 把补丁追加为中段 system（前缀保住），flash 折叠进首条 system（token 0 失效）；工具变更两种情况都全量失效；重放必须基于上次实际 payload。
- [任务：实测 DeepSeek 前缀重放的真实命中率](tickets/WF-S1-02-ds-cache-probe.md)：逐字节重放 + 尾部指令命中 96.4%（5120/5312），前缀中部改一个词只命中到改动点（2432）；尾部指令永不命中。
- [Grilling：降级路径的触发条件与错误码](tickets/WF-S1-03-degradation-decisions.md)：只新增 `PrefixUnavailable`；serialize 只做冷启动 fallback；不做可压/不可压分类。设计见 [压缩请求前缀不变式](WF-S1-03_压缩请求前缀不变式设计.md)。

## Not yet specified

_（空）_。设计路线已清晰，剩余未定项都是执行期验证，不是设计缺口。

## Out of scope

- **Anthropic 原生 `compaction` beta 及其 `signature` / `tool_changes` 回放**：不在本次范围。它压整段对话而非 span，且需要 provider 私有状态持久化。
- 其它 provider（OpenAI Responses、Gemini、Moonshot）的适配：单 provider 优先。
- 在 pi 里实现跨 provider 迁移或 `encrypted_content` 持久化。
- Sprint01 之外的生产实现排期与发布。
- tool-result pruner（DSH 有、pi 无）：它要改写历史 tool result，与前缀不变式直接冲突，属于另一次设计。
- 会话中途在 `deepseek-v4-pro` 与 `deepseek-flash` 之间切模型：缓存按模型分区，切了就冷启动，不是设计缺口。

## Tickets

Open tickets 是本目录下的 `tickets/` 子文档，按 `blockedBy` 判前线；本 map 不重复列出它们的正文。
