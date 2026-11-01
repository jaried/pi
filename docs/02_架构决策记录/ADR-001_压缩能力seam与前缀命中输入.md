# ADR-001 压缩能力 seam 与前缀命中输入

- 状态：草案（待随承接 Issue 实施生效）
- 日期：2026-11-01
- 承接 Issue：S1-01（原始需求 `docs/01_Sprint记录/Sprint01/S1-01/S1-01_原始需求.md`）
- 设计前身：`docs/01_Sprint记录/Sprint01/WF-S1-03_压缩请求前缀不变式设计.md`
- 方案决策：`docs/01_Sprint记录/Sprint01/S1-01/S1-01_方案决策.md`

## 背景

pi 的自动压缩把历史序列化成一段文本，配上独立的 summarizer system prompt 再请求模型。实测（DeepSeek `deepseek-flash`，20129 字符前缀）：

| 组 | 命中 / 未命中 |
|---|---|
| 冷启动 | 0 / 5292 |
| 逐字节重放 + 尾部指令 | 5120 / 192（96.4%） |
| 前缀中部改一个词 | 2432 / 2883（截断到改动点） |

现行实现命中率恒为 0。DeepSeek 1M 窗口、`v4-pro` 高峰价下一次压缩的输入约 8.6 元；命中价下同一次压缩的输入约 0.30 元。

同时发现两处机制事实：

- DeepSeek 无服务端压缩：Responses API `context_management`、`previous_response_id`、`store` 均不受支持；Anthropic 有 `compaction: { type: "summarize" }` beta。
- `deepseek-flash` 静默忽略中段 system 消息（实测答 `GREEN` 而非 `BANANA`），只有末条 system 生效；`deepseek-v4-pro` 支持中段 system。因此 flash 必须把 prompt 补丁折叠进首条 system，这是它的系统性全价 miss 来源，与压缩无关。

## 决策

1. 压缩能力是一个 seam，两个 Adapter：`provider`（原生远程压缩）与 `prefix`（重放前缀 + 尾部指令）。选择依据是 catalog 静态声明，fail-closed：声明了但未实现返回 `unsupported`，不静默降级。
2. `prefix` Adapter 遵守两侧不变式：摘要请求既是上一次实际请求的前缀，也是当前上下文的同一前缀；任一不成立即抛 `PrefixUnavailable`。
3. wire body 只由 `buildChatBody()` 产出，coding-agent 不重新序列化、不截断历史。
4. 触发点 = `min(contextWindow × threshold, contextWindow − maxTokens)`，`threshold` 默认 0.95，全模型一致；provider context-overflow 作为反应式兜底。
5. 摘要请求沿用模型默认参数，不额外设置 `max_tokens` 或 reasoning 覆盖。
6. 上次请求体只在进程内保留，不持久化。

## 后果

- DeepSeek 上压缩输入从全价降到命中价（约 27–35 倍），质量不变。
- 触发点因「模型限额先到」而在 DS 1M/384K 上落在 616K（61.6%），压缩次数增加，已接受。
- 声明原生压缩的 provider 在 `provider` Adapter 交付前会得到 `unsupported`，属显式失败而非降级。
- 小窗口模型（32K）在 95% 下只剩约 1.6K 输出空间，长回答可能被 provider 拒绝，记为待观察项。
- 不引入 tool-result pruner：命中价下裁剪换来的一次全量 miss 远贵于节省的输入。

## 备选与被否决

- 每 provider 一套 summarizer 接口（各自暴露 signature、beta 头、工具变更）：接口宽度与实现相同，属浅模块，否决。
- 只做客户端前缀、不建 seam：在引入原生压缩需求后不成立，两套机制都是真实存在的。
- 摘要 `maxTokens` 收紧到 1200：实测撞顶导致 `length` 硬失败（4/6 段），否决。
- 摘要关闭 thinking：实测省 1.2 倍 token，但用户要求沿用模型默认，未采纳。
