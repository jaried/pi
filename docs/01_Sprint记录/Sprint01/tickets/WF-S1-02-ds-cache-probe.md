---
id: WF-S1-02
label: wayfinder:task
parent: ../wayfinder-map.md
status: closed
assignee: tony
blockedBy: []
blocks: [WF-S1-03]
sprint: Sprint01
created: 2026-11-01
closed: 2026-11-01
---

# 任务：实测 DeepSeek 前缀重放的真实命中率

## Question

把「原样重放上一轮对话 + 尾部追加一条摘要指令」发给 DeepSeek，`prompt_cache_hit_tokens` 实际是多少？重放前缀与上次请求逐字节相同时命中多少，仅差一个字符时又命中多少？

这张任务存在的唯一目的是解阻塞：不拿到真实命中数字，降级策略（WF-S1-03）无从判断该不该保留 serialize 回退路径。

## Context

- 凭证：`~/.pi/agent/auth.json` 已有 `deepseek` 条目
- 端点：`https://api.deepseek.com`（OpenAI 格式）与 `https://api.deepseek.com/anthropic`
- 计费口径：`https://api-docs.deepseek.com/zh-cn/quick_start/pricing/`（元/百万 tokens，闲时与高峰两档，缓存写入不单独计费）
- 命中规则：`https://api-docs.deepseek.com/guides/kv_cache` —— 必须完整匹配一个已持久化的 cache prefix unit；unit 在请求边界、跨请求公共前缀、长输入固定间隔处持久化
- 做法要求：至少三组对照 —— 完全一致、尾部只加指令、前缀中段改一个字符。每组记录 `prompt_cache_hit_tokens` / `prompt_cache_miss_tokens` / 耗时
- 成本：三组小样本预计远低于 0.01 元，实际发起前须得到用户确认

## Resolution

已实测。凭证来自 `~/.pi/agent/auth.json`，脚本与原始输出留在 `../.scratch/证据/`。

| 组 | 内容 | 命中 / 未命中 | 结论 |
|---|---|---|---|
| A | 冷启动基线 | 0 / 5292 | 首次无缓存 |
| B | 逐字节重放 A 的 payload + 尾部 `assistant:"OK"` + 摘要指令 | **5120 / 192** | 命中 96.4%，尾部约 190 token 按未命中计 |
| C | 原样重复 B | 5120 / 192 | 尾部段仍不命中：持久化的 unit 边界落在基础请求末次输入处 |
| D | 前缀第 110 行改一个词 | 2432 / 2883 | 命中截断到改动点，之后全 miss：严格前缀语义 |

三条结论：

1. 不变式成立，而且收益是量级性的：当前 pi 的实现命中 0%，重放可命中 96%+。
2. 前缀任何位置改动都让该点之后全部失效，所以必须重放**上次实际发出的 payload**，不能重新投影。
3. 尾部指令、`assistant` 衔接消息永不命中，所以指令要短（设计取 300 token 上限），且不要把摘要指令写长。
