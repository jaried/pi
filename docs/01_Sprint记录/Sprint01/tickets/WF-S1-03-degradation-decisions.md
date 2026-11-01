---
id: WF-S1-03
label: wayfinder:grilling
parent: ../wayfinder-map.md
status: closed
assignee: tony
blockedBy: [WF-S1-01, WF-S1-02]
blocks: []
sprint: Sprint01
created: 2026-11-01
closed: 2026-11-01
---

# Grilling：降级路径的触发条件与错误码

## Question

当字节前缀无法成立时，summarizer 该返回什么，调用方该做什么？需要定下的决定：

1. 触发条件：分段补丁（v4-pro 安全 / flash 全量失效）、工具集变化、换模型、上次请求未落库 —— 这几类里哪些判为「不可压缩」、哪些判为「可以用旧 serialize 路径继续」？
2. 错误码：`unavailable` / `prefix-mismatch` / `unsupported` 三者语义边界在哪，是否需要区分「前缀不可用」与「摘要失败」？
3. 旧 serialize 路径的去留：保留为兜底（多一套始终要测的路径），还是删掉、前缀不成立就不压（少一套路径，代价是某些 provider 彻底不能压）？
4. 计费可见性：走兜底路径时是否必须在 UI 上标注「本次摘要未命中缓存」？

## Context

- 该 ticket 依赖两张事实 ticket 的结论，不允许在事实缺失时定案
- 设计词汇用 `codebase-design`：先定不变式，再定错误模式的形状，最后才谈实现
- pi 现有错误形态：`docs/compaction.md` 的 `session_compact_failed` 事件与 `ManualCompactionErrorCode`
- DSH 对照：`packages/compaction/compaction-basic/README.md` 的失败处理与 KV cache effect 段
- 已完成事实：WF-S1-01

## Resolution

四个决定，已锁定进 [`WF-S1-03_压缩请求前缀不变式设计.md`](../WF-S1-03_压缩请求前缀不变式设计.md)。

1. **触发条件**：不区分。只要 `previous` 仍是当前 surface 的连续前缀就可用；一旦更早条目被改（`context_edit`、工具集变化、flash 上的 section 补丁折叠）就判 `PrefixUnavailable`。不做“可压/不可压”分类，因为分类本身就是两套代码。
2. **错误码**：只新增一个 `PrefixUnavailable`。摘要本身失败继续用现有的 `session_compact_failed`。不区分「前缀不可用」与「摘要失败」两套码，因为二者对调用方的动作不同：前者重选路径，后者不重试。
3. **旧 serialize 路径**：保留，但**只**作为 `PrefixUnavailable` 的冷启动分支（会话从磁盘恢复）。一个入口两条分支，不升成接缝。
4. **计费可见性**：走 fallback 时必须在 UI 标注本次摘要未命中缓存，复用现有 `showCacheMissNotices`。

观测到的现状缺陷：运行 `.scratch/prefix-invariant.test.ts` 时，无 `previous` 的情形并没有声明不可用，而是直接发出了一个全价请求（返回 401）——这正是本条要修的语义。

codebase-design 自检：单 provider 只有一个实现，**不引入 summarizer 接缝**（删除测试：删了它复杂度只回到同一个文件）；不变式写进接口文档，测试只跨接口。
