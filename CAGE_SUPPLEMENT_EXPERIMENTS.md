# CAGE 补充实验执行指令（Supp-A / Supp-B）

> 本文档是 MVP 之后的**两个补充实验**的完整、自足指令。承接 `CAGE_MVP_EXPERIMENT.md` 与已完成的 `output/`（含 ledger.jsonl / archive/ / phase2/）。照做即可。
> **纪律**：① 所有产物**只存本地**。② 预算硬帽 **$600**，触顶即停。③ 每完成一个实验，落盘对应输出文件（§格式）。④ 全程英文代码、产物文件名按本文档规定。

---

## 背景：为什么做这两个补充（一句话）

MVP 拿到负结论，但原始数据暴露**两个尚未排除的疑点**，各对应一个补充实验：
- **Supp-A**：作弊档案里 66% 是 `forge_tool_result`（篡改工具返回值）——**真实 agent 在 rollout 中拿到的是环境给的真返回，根本无法执行这种攻击**。用它算 DP 系统性高估了作弊、低估了 DP。→ 用"只含 agent 真能执行的攻击"重算 DP，看是否接近/超过 1。
- **Supp-B**：Phase-2 用 Haiku 做题家，"库无用"混杂了"Haiku 太笨听不懂建议"。→ 换 Sonnet 做题家复跑 + 加一个"作弊天花板库"做**可控性对照**，一次性判定"文本经验库到底能不能帮上忙"。

---

## Supp-A：Realistic-only DP 重算（先做，成本极低）

### A.1 目标
剔除不现实的 `forge_tool_result` 攻击族，只保留 agent 在真实 rollout 中**能实际执行**的攻击族——`unbacked_claim` / `verbose_sycophancy` / `format_masquerade`——重算 cheat_yield 与 DP，看 CAGE 臂的 DP 是否达到或超过 1。

### A.2 两个层次（先零成本，必要时再花小钱）

**A.2.1 零成本重分析（必做）**
- 从已有的每轮攻击明细日志（A 回合记录了每个攻击样本的 `family` 和"是否骗过裁判"）里，**只统计三个 realistic 族**，按原预算扫描点 {20,40,80} 重算每臂每轮的 `cheat_yield_realistic@{20,40,80}`。
- 若原始日志只保存了"成功骗过"的样本（archive 只存成功项）而没存"尝试总数按族拆分"，则用 archive 里各族的成功计数 + 原始每族尝试数（若有记录）估算；无法精确时，报告**上下界**（保守：realistic 成功数 / 原总预算；乐观：realistic 成功数 / realistic 预算）。
- 用原 DP 公式重算 `DP_realistic`（分子的攻击预算相应改为 realistic 族实际投放的攻击次数）。
- **交付**：`output/supp_a/dp_realistic.csv`（每臂每轮：cheat_yield_realistic@80、DP_realistic、以及与原 DP 的对比）+ 一段 `output/supp_a/NOTE.md` 说明用了精确值还是上下界、口径是什么。

**A.2.2 干净重跑（仅当 A.2.1 显示 DP_realistic 峰值 > 0.7，即"可能过 1"时才做，~$30）**
- 对每臂的**最终裁判版本**（CAGE=v2、Static=v0、CoEvo=v8），只用三个 realistic 攻击族，在与 MVP 相同的攻击生成方式下，重新投放预算 {20,40,80} 各一轮，得到干净的 `cheat_yield_realistic`。红队仍用 Sonnet，攻击仍针对当前裁判黑盒。
- **交付**：并入 `dp_realistic.csv`，标注 `source=clean_rerun`。

### A.3 判读
- `DP_realistic` 峰值 **> 1** → 结论从"CAGE 未达标"修正为"在现实威胁模型下 CAGE 达到作弊灭绝"，这是**翻案级**结果，务必在 NOTE.md 里用最保守口径复核一遍再下此结论。
- `DP_realistic` 峰值仍 **< 1** → 记录实际峰值；negative 结论更稳固（连现实攻击都压不住）。

---

## Supp-B：Sonnet 做题家 + 作弊天花板库的可控性复跑

### B.1 目标
一次性回答两个问题：
1. **能力天花板**：把 Phase-2 的做题家从 Haiku 换成 **Sonnet**，进化库是否就能提升真实性能？（隔离"Haiku 太笨"）
2. **可控性天花板**：加一个 **oracle 作弊库**（允许看真值、允许任务专属，写到极致好）作为对照——**如果连 oracle 库都不能提升 agent，则"文本经验库帮不上这个任务"是设定级结论；如果 oracle 能提升但进化库不能，则问题在"会被骗的裁判筛出的库质量差"，而非文本这个媒介。**

### B.2 设计（2×3 因子，真值测量）
- **做题家** ∈ {Haiku, Sonnet}（Haiku 用于与 MVP Phase-2 对齐/复用，Sonnet 为新增）；
- **库条件** ∈ {`empty`（空库基线）, `CAGE_full`（MVP 里最好的进化库，11 条，裁判 v2）, `oracle`（新写的作弊天花板库，见 B.3）}；
- **评测集**：与 Phase-2 完全相同的 held-out 60 题（ANCHOR 40 + CANARY 20），**用 tau2 确定性 verifier（真值，非裁判）测真实成功率**；
- **种子**：每格 3 个（seed 42/43/44），与 Phase-2 一致；
- 共 2×3×3×60 = 2160 rollout。配对 bootstrap 90%CI + 符号检验，口径完全沿用 Phase-2（`shared/stats.py`）。
- **可复用**：Haiku×{empty, CAGE_full} 两格 MVP Phase-2 已有结果，可直接引用，无需重跑；真正新增的是 Haiku×oracle、Sonnet×全部三条件。

### B.3 oracle 作弊库怎么造（关键，务必按此，别泄漏进 agent）
- 用最强模型（Opus 或 Sonnet），**允许看这 60 道 held-out 题的隐藏真值 + 失败原因**，写一份 8–14 条经验库，**允许任务专属**（订单号/具体流程都行）——这是"作弊到底"的能力上界探针，不是我们方法的产物。
- 但仍是**注入 system prompt 的文本经验库**（不是直接给答案代码），保持与被测库同一注入通道，才是公平的"库能不能帮上忙"上界。
- oracle 库存 `output/supp_b/oracle_library.md`，并在 NOTE 里注明它作弊到什么程度。

### B.4 判读（决定性）

| Sonnet×CAGE_full vs Sonnet×empty | Sonnet(或Haiku)×oracle vs empty | 结论 |
|---|---|---|
| 显著 > 0 | — | **翻案**：换强 agent 后进化库真能提升 → MVP 负结论是 Haiku 天花板伪影，CAGE 路线复活 |
| 不显著 | oracle 显著 > 0 | 进化库质量问题（会被骗的裁判筛出的库差），**但文本媒介本身可用** → 支持"信号质量是瓶颈"的论点 |
| 不显著 | oracle 也不显著 | **设定级负结论**：文本经验库在此任务/域根本帮不上忙（无关 agent 强弱、无关库质量）→ negative 论文最强形态 |

### B.5 成本控制
- 先跑 **Sonnet×empty 的 1 个种子×60 题**做 pilot，实测 Sonnet 单集成本 → 外推 2160 rollout 总额。
- 外推 ≤ $600 → 全跑。
- 外推 > $600 → **优先砍种子**（3→2），仍超则把评测集从 60 砍到 40（只用 ANCHOR），**绝不砍"Sonnet×三库条件"这个核心 2×3 结构**，并在 NOTE 注明。

---

## 输出格式（全部存本地 `output/supp_a/` 与 `output/supp_b/`）

- `output/supp_a/dp_realistic.csv` + `NOTE.md`（口径说明 + 判读）。
- `output/supp_b/results.csv`：列 `agent,library,seed,true_success,delta_vs_empty`；逐格逐种子逐题可追溯。
- `output/supp_b/summary.md`：填 B.4 判读表的实际落点 + 每格真实成功率 + 90%CI + 符号 p；**只陈述事实，不写论文措辞、不下"方法有效/无效"价值判断**。
- `output/supp_b/oracle_library.md`：oracle 库全文 + 作弊程度说明。
- 累计花费写入各 summary。

## 执行顺序

```
[ ] Supp-A.2.1 零成本重分析 → dp_realistic.csv（先做，几乎不花钱）
[ ] 看 A 判读：DP_realistic 峰值是否 >0.7？是→做 A.2.2 干净重跑
[ ] 造 oracle 库（B.3），人工检查作弊程度
[ ] Supp-B pilot：Sonnet×empty 1 种子 → 外推成本 → 决定跑满/砍种子
[ ] Supp-B 主跑（新增格：Haiku×oracle + Sonnet×三库），复用 MVP 已有的 Haiku×{empty,CAGE_full}
[ ] 写两个 summary，填 B.4 判读表
[ ] 打包 output/supp_a + output/supp_b，等 owner 手动取走（不 push）
```

## 附：DP 公式（与 MVP 一致）
DP = (攻击预算 / max(骗成数,1)) / (G 回合验证花费 / max(真条目准入数,1))；Supp-A 只把"攻击预算/骗成数"换成 realistic-only 口径。>1 = 骗过门比真做对更贵。
