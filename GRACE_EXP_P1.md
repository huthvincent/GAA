# GRACE Exp-P1 — 接地准入自我进化的第一个正结果实验

> 本文档是你要跑的实验的**完整、自足的指令**。照做即可。
> **纪律**：① 所有产物**只存本地**。② 预算硬帽 **$500**，触顶即停。③ 每完成一个阶段/一轮，落盘对应输出文件（§7 格式）。④ 全程英文代码、产物文件名按本文档规定。

---

## 1. 背景与目标（一句话）

前序实验已锁定：文本经验库**有用**（oracle 库 +9.4pp），但**文字裁判把关的自我进化产不出好库**（+0.0pp，且放行教作弊的毒条目）；能持续 work 的自我进化全部骑在**执行级接地信号**上。本实验验证 GRACE 的核心假设：

> **把准入信号从"文字裁判"换成"执行级验证器"，同一套自我进化循环就能在 held-out 上产生真实正增益。**

环境换成 **AppWorld**（代码调 API 的 agent，自带隐藏单元测试式验证器——检查 DB 状态变更是否全部达成且无副作用）。这是"接地信号在优化时合法可用"的 Tier-1 域。

## 2. 三条臂（唯一差异 = 准入信号）

| 臂 | 自我进化 | 准入信号 | 意义 |
|---|---|---|---|
| **G（GRACE 接地准入）** | 有 | **AppWorld 官方验证器**在 OPT（train）任务上的执行结果 | 核心主张 |
| **T（文字裁判准入）** | 有 | Sonnet **只读 transcript** 判成败（不许看验证器/执行结果） | 关键消融=复现 CAGE-v1 失败模式 |
| **N（无库 baseline）** | 无（空库） | — | 底线（即 held-out 上空库的成绩，可与 G/T 共用评测 rollout 的 seed 配对） |

**成功判据（正结果）**：held-out 上 **G > N 且 G > T**（配对统计，目标 G−N ≥ +5pp）。预期副产品：T ≈ N 或更差、T 臂放行的毒/无用条目率显著高于 G。

## 3. 环境与模型

```bash
pip install appworld && appworld install && appworld download data
# agent 用 AppWorld 文档的 ReAct 式代码执行循环（他们有示例 scaffold），单集交互上限 ~40 步
```
- **agent = Sonnet**（`global.anthropic.claude-sonnet-...`）。不要用 Haiku——Supp-B 已证 Haiku 吃不动好库（能力天花板），而 Sonnet 能（oracle +9.4pp 就是 Sonnet 吃出来的）。
- 蒸馏器 = Sonnet；T 臂文字裁判 = Sonnet。温度 0。
- **冒烟自检（≤$20，必做）**：train 里 3 个任务跑通"agent rollout → 官方 evaluator 打分 → 库注入 system prompt 且确实改变行为 → T 臂文字判卷"全链路；**实测单集成本**，外推总预算（§8）。

## 4. 数据划分（写死进 config，seed=42）

- **OPT 池**：train split 抽 **40 个任务**（跨 app 分层抽样，别集中在单一 app）。自我进化只在这发生；验证器结果在 OPT 上**对 G 臂合法可见**（这是 Tier-1 设定的核心：优化用信号本来就是环境提供的）。
- **HELD-OUT 评测集**：dev split 抽 **40 个任务**（同样分层）。**官方 evaluator 只在测量时使用；held-out 任务在优化期间对 agent/蒸馏器/裁判全程不可见。**
- 冒烟时确认 train/dev 本地 evaluator 可跑；若 dev 不可用，从 train 另切互斥的 40 个当 held-out 并在 config 注明。

## 5. 自我进化循环（G/T 两臂完全相同，只换准入信号；4 轮）

每轮，对每臂：
1. **尝试**：agent 带当前库在 OPT 池跑 **15 个任务**（轮转覆盖，避免每轮同一批）；
2. **蒸馏**（防自我确认）：**独立蒸馏器**（非 agent 自我总结）对比本轮成功/失败轨迹，写 **≤3 条候选条目**——通用流程经验（如"改状态前先查当前状态""API 分页要遍历完"），**禁止任务专属实体**（具体 task id/人名/值）；每条必须附**溯源**：引用它derive自的具体轨迹与关键 API 调用；
3. **配对 A/B 验证**：每候选在 **5 个匹配 OPT 任务**上跑 with vs without（同 seed 配对）+ **3 个无害对照**（该臂已通过的任务）：
   - **G 臂打分 = AppWorld 官方验证器**（执行级，pass/fail）；
   - **T 臂打分 = Sonnet 读 transcript**（**严禁**接触验证器结果——这是本实验的全部意义所在，实现时用断言锁死）；
4. **统计准入门**（两臂同规）：配对差 Agresti-Caffo 单侧 95% LCB > 0 **且**无害对照不掉分（**禁 Wald**；n<8 用 Agresti-Caffo）；
5. 库容硬帽 **8 条**；落盘本轮库快照 + 每条准入/拒绝记录（含 A/B 明细）。

## 6. 测量

- **held-out 评测**：**round 0（=N 臂基线）、round 2 后、round 4 后**，G/T 两臂各用当轮库在 40 个 held-out 任务 × **2 seeds** 跑，**官方 evaluator 判分**（这是 gold，测量专用）。三臂共用同批 task×seed 配对。
- **主指标**：held-out 验证成功率（AppWorld 的 TGC/任务级 pass）。配对 bootstrap 90%CI + 符号检验（同 Supp-B 口径）。
- **副指标**（讲故事的证据）：
  - 每臂每轮准入条目数、被拒数；
  - **T 臂的"判卷-验证器背离"**：T 臂准入时裁判说"提升"的条目，事后用验证器复核其 A/B 真实效果（这个复核**只做记录、不回流优化**），量化"文字裁判放进了多少实际无益/有害的条目"；
  - 条目文本人工可读快照（毒条目定性证据，对照 MVP 那个"编确认号"物证）。

## 7. 输出格式（存本地 `output/exp_p1/`）

- `config.json`：models、task_splits（OPT 40 + HELDOUT 40 的 task_id 列表）、rounds、cap、seed、appworld 版本;
- `ledger.jsonl`：每臂每轮一行：`{arm, round, opt_attempt_success_rate, candidates, admitted, rejected, library_size, cost_usd_cumulative}`;
- `library_G_r{1..4}.json` / `library_T_r{1..4}.json`：库快照（条目全文 + 溯源 + 准入 A/B 数字）;
- `heldout_results.csv`：`arm, checkpoint(r0/r2/r4), seed, task_id, verified_success`（逐题逐种子，可追溯）;
- `divergence_T.csv`：T 臂每条准入条目的"裁判判定效果 vs 验证器复核效果";
- `summary.md`：一页客观事实（只报数，不下"方法有效"价值判断）：三臂 held-out 成绩表（均值+CI+符号 p）、G−N 与 G−T 差值、T 臂背离率、总花费、异常如实记录。

## 8. 预算与止损

- 硬帽 **$500**；每次 LLM 调用计费累计。冒烟后先跑 **G 臂第 1 轮**实测单轮成本，外推三臂全程（注意 AppWorld 单集比 tau2 长得多，Sonnet 单集可能 $0.3-0.6）：
  1. 外推 ≤$500 → 按上述规模跑满；
  2. 超 → 依次砍：**中间 checkpoint（r2 评测）→ 轮数 4→3 → OPT 尝试 15→10/轮 → held-out 2 seeds→1**；**绝不砍**：三臂结构、held-out 最终评测、T 臂与 G 臂同规模（否则对照不公平）；
  3. 所有裁剪写进 config.json 与 summary.md。

## 9. 红线（违反则实验作废）

1. **T 臂的裁判绝不接触验证器结果 / 执行判分**（代码断言锁死；它只能看 transcript）——两臂对照的全部意义；
2. **held-out 的验证器结果绝不回流优化**（G 臂的接地信号只来自 OPT 任务）；
3. 蒸馏条目**禁止任务专属实体**（防 OPT 死记硬背——held-out 本身也会惩罚背题，但源头就要拦）；
4. G/T 两臂除准入信号外**逐行代码相同**（同蒸馏器、同 A/B 规模、同统计门、同库容）。

## 10. 判读

| 结果 | 含义 | 下一步 |
|---|---|---|
| **G>N 且 G>T**（目标 G−N≥+5pp） | **正结果成立**：接地准入让同一循环从无效变有效 | 建完整 GRACE（tau2 Tier-2 约束核查、红队硬化、对齐 ACE/SkillOpt baseline） |
| G≈N | 接地准入也推不动（与 ACE +17% 矛盾） | 先查实现（库真的注入了？蒸馏质量？agent 读库了？），报诊断，别急着下结论 |
| G>N 但 T≈G | 文字裁判在代码域意外够用 | 如实报告；说明域差异，故事需修正 |

## 11. 执行顺序

```
[ ] AppWorld 环境搭建 + ReAct agent scaffold
[ ] 冒烟（3 题全链路 + 单集成本实测，≤$20）
[ ] config.json（OPT/HELDOUT 划分写死）
[ ] round 0 held-out 基线（=N 臂，2 seeds）
[ ] G 臂 r1 → 外推成本 → 定裁剪档
[ ] G/T 两臂 4 轮循环（每轮落盘库快照+ledger）
[ ] r2、r4 held-out 评测（2 seeds，官方 evaluator）
[ ] T 臂背离复核（divergence_T.csv）
[ ] summary.md + 打包 output/exp_p1/，等 owner 取走（不 push）
```
