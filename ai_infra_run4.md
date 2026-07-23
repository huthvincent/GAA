# 执行机作业书 · 第四轮（RUN4：detector v2.1——按已验证契约实现并重考）

> **给执行机（EC2 + Bedrock，Claude Code）**。自包含，只读本文件即可开工。
> 版本：run4.v2（2026-07-23，经两路红队 17 条修订后定稿）。
> 工作根：run3 目录（含 RUN3 全部产物与 knowledge/ 资产）。模型：Opus 4.8 标准通道。
> 预算：**总硬帽 $500**（分帽合计 $430，估 ~$350）；结余滚动；成本硬触发见 §2.4。
> 每次 API 调用的真实 in/out tokens 与 cost 记入 `reports/usage_ledger.jsonl`
> 并按 Stage 归属——**成本对账表必须平账**（RUN3 的 $34 差额勿再犯）。

## 背景（30 秒）

RUN3 的 v2.0 判负已尸检定性：死于四处实现缺陷（leg3 降级成装饰、三问成 severity
硬闸、常驻手册白名单、大 commit 2 轮截断），不是配方问题。修复版 v2.1 已在 Mac 端
40 题双臂验证：命中 19/24 vs 裸模型 7/24（p≈0.001），热点负例误报 ~9%。
本轮：**按 §1.1 契约原样实现 v2.1**，重考 RUN3 未完成的判据。

## 统一计分口径（全轮唯一，先于一切）

- **正式计分 finding** = `severity ∈ {critical, important}` **且** `conf_final ≥ 0.5`。
  recall 与 FPR **都**只对正式计分 finding 计——两侧对称，不许一边宽一边严。
- `conf_final` 由 harness 计算：模型报**折前** `conf_raw`；`unverified` 非空 →
  `conf_final = conf_raw × 0.7`，否则 = conf_raw。两值都存进 finding（可审计）。
- 主表必须按 `source × conf 带（<0.5 / ≥0.5）` 分层报 recall 与 FPR；
  任何单带数字不得作 headline。
- budget=2 排序用 `conf_final`。

## 不可违反的纪律

1. test 零新模型调用（只读重聚合豁免同 RUN3）。
2. 呈现规则同 protocol；leak_attempt 全程 0。
3. 资产复用不重建（handbook/repo_profile/leaf_verification，RUN3 已泄漏审计）。
4. **实现契约高于一切**（宪章 7.7）：§1.1 CONTRACT A–E 逐条照办；prompt 用原文，
   仅允许 §1.1 列明的**四处机械替换**。契约本身有问题 → BLOCKERS 停机，不许变通。
5. 判分：eval_judge.v1 原样；第一次判分即正式成绩；judge 缓存全保留回传。
6. 每个 Stage 必披露检测行为指标（出 finding 项数、severity/source/conf 带分布）。

---

# Stage 0 — 地基（$5）

0.1 `git tag -l` → `reports/git_tags.txt`；tag **`run4-start`**；确认 JUDGE_CACHE_ONLY 在位。
0.2 judge 时漂锚检（30 条 RUN2 v1 预测，一次性对照不写回）：一致率 ≥90% 或 κ≥0.8
    才继续；**另必须报分歧方向**（现判偏宽/偏严各几条）——净偏宽 ≥3/30 时，最终
    判据宣判旁必须标注漂移方向与幅度。已知局限照录：route 风格 claim 无历史锚，
    以 §2.3 的 route-HIT 叶子一致率作替代可复核指标。

# Stage 1 — v2.1 实现＋活性门（$60，含最多 3 轮冒烟迭代的预算）

## 1.1 实现契约（宪章 7.7）

**CONTRACT A · system prompt 原文**。仅允许四处机械替换：①`git show/grep` →
`read_file_at_parent`/`grep_at_parent`；②"读 views3.json" → harness 递入视图；
③**删去步骤 3 中"confidence×0.7 并"字样**（折扣统一由 harness 做，见口径节）；
④74 叶名单不进 prompt 正文——由 `handbook_lookup` 工具 schema 的 enum 提供
（无效叶名返回"无此叶＋有效叶名列表"，不报错）。原文：

```
你是 AI infra 性能守门员 v2.1。判断该 commit 是否可能引入性能回归
（变慢/更费显存/吞吐下降/延迟上升/卡死）。

【装备（train 数据蒸馏，按需使用，不必通读）】
- 病例手册（74 类）：用 handbook_lookup(<leaf>) 取相关叶小节。
  **手册没写的模式不代表良性。**
- 事故高发地图：视图已附本仓 top 热点文件。
- 叶子验证表：给 finding 配 suggested_benchmark 用。
【父快照核验】read_file_at_parent / grep_at_parent（只允许改动前状态）。
【工作流程（按序执行）】
0. 若改动文件**全部**在 tests/docs/examples/.github 下且不碰任何
   config/recipe/defaults 文件 → 直接 no_issue，结束。
1. **分诊（必做，独立于深审）**：判断改动是否触及性能面（计算/通信/显存/
   调度/服务热路径）。**只要触及性能面，必须立一条 route finding**：
   severity=important、source="route"、category=最相关叶子（拿不准 → 填最接近的
   叶或 "uncategorized"，confidence 取下限 0.3）、claim="该改动触及〈类别〉类
   风险（一句话说明为何相关），建议验证"、confidence=分诊把握度（0.3-0.8）、
   suggested_benchmark 按验证表填。**这条 finding 不许因深审"没确认"而删除。**
   确实不触及性能面 → 无 route finding。
2. **深审**：diff >400 行或 >10 文件 → 先快扫全部块挑最可疑的 ≤2 块做父快照
   深查；小改动直接深查。深审发现的具体机制问题另立 finding（source="deep-review"）。
3. **三问校准（不是资格闸）**：对每条 finding 尽力回答三问（热路径？谁调用？
   默认配置走到吗）。答不全 → 在 unverified 字段写明哪问未核；
   **severity 只按"若为真影响多大"定级，不许因核验不全而降级或弃报**。
4. 输出：route 与 deep-review 一起按置信度排序取前 2。
   真不触及性能面 → no_issue=true。
```

**CONTRACT B · route 独立性（I/O 契约）**：`touches_perf_surface==true` ⇒ **必有**
一条 route finding 进入排序池（叶子不明确不豁免——用最近叶/uncategorized＋conf 0.3）。
生成与保留不依赖深审输出。**不变量：每次运行报告 `touches=true 且无 route finding`
的项数，必须恒为 0**。禁止实现成"给深审 finding 贴标签"。

**CONTRACT C · 三问=校准器**：模型只报折前 conf_raw ＋ unverified；折扣由 harness
单次施加（见统一口径节）；严禁任何按核验不全降 severity/丢 finding 的代码路径。

**CONTRACT D · 手册按需**：手册不常驻；`handbook_lookup(leaf)` 返回该叶手册页；
视图附 repo_profile 本仓 top 热点文件（≤30 行）。

**CONTRACT E · 大 commit 两级审**：分解后先廉价扫描全部块打可疑度（**扫描输入若因
长度截断任何块，须在 meta 记录每块截断字节与总块数；超限改分批扫描合并排序，禁止
单次硬塞**），只对 top-2 块做满 5 轮深审；被跳过块数写 meta。废除 2 轮截断与顺序取块。

**画像门落地**：删除代码级 profile_gate 短路；谓词照算写入每项 meta
（`gate_predicate`），**所有项一律送主审**（步骤 0 由模型执行）；"含门口径"从 meta
机械统计。正样本中 `gate_predicate=true` 或模型步骤 0 判 no_issue 的，逐条列出。

**token 记账**：每次调用真实 usage 写 `reports/usage_ledger.jsonl`（RUN3 tokens
全零的死字段必须修）。

## 1.2 活性门（宪章 7.6，冻结前置）

冒烟集 = `splits/v2_smoke_manifest.json` 原样（18 正＋7 负）。必报：出 finding 项数、
severity/source/conf 带分布、route 不变量计数、$/item。
**冻结门（四条全过）**：①18 正样本上**正式计分口径**（severe∧conf_final≥0.5）开火
≥8/18；②7 负样本上正式计分误报 ≤3/7；③route 不变量 = 0；④leak = 0。
不过门 → 冒烟集上迭代（每轮留痕，单轮预算 ~$20），最多 3 轮；仍不过 → BLOCKERS。
过门 → FROZEN 注记，Stage 2 起禁改。
**冻结例外（仅此一类）**：不改变检测行为的崩溃/超时/解码修复（禁碰 prompt/阈值/
排序/门控），留痕 diff＋理由，受影响项作废重跑，不算迭代轮次。

# Stage 2 — dev 全量评测（$260）★ 主实验

2.1 `splits/dev_tune.txt` 全量 250 项，标准视图（framing 已证伪，无转换无双跑）。
2.2 **误报早停检查点（预授权）**：前 80 项（按 case/pair/neg 分层比例排program）跑完后，
    用正式计分口径算负样本原始触发率（不必等 judge）：**>30% → 暂停写 BLOCKERS
    等 Rui**，不许烧完剩余 170 项——把"$14 拦 $130"的教训对称用到误报侧。
2.3 judge：κ 抽检 100 项（正/负×HIT/MISS 分层，第二判绕缓存加 nonce 不写回）≥0.7；
    跨家族 Nova 50 项过采 HIT（HIT<50 → 全取＋补齐并写明缺口原因）。
    **route 命中口径（预登记）**：judge 按冻结的 eval_judge.v1 判，不为 route 放宽；
    但主表必须把 recall 拆成 **route-HIT / deep-HIT 两列**，并单独报
    route-HIT 中叶子与真值 taxonomy_label 一致的比例——judge 无论判宽判严，
    数字都可独立复核。
2.4 成本硬触发：冒烟 $/item > $0.85 → 立即预授权执行"砍 Stage 3＋Stage 4 不补窗"，
    无需等超帽。
2.5 **成功判据（写死，正式计分口径）**：
    - overall recall@2 ≥ **25.3%**；pair@2 ≥ **13.3%**；weighted FPR ≤ **15%**
      （研究门；产品红线 10% 并排标注）
    - 主表必含：分层 FPR（尤其 hard-negative）、source×conf 带分层、route/deep
      命中两列、route 不变量、gate_predicate 正样本清单、error 项数
      （**error >5% → BLOCKERS**）。
    - **重叠敏感性**：披露冒烟 25 项与 Mac 本地 40 题（清单在
      `detector/v2_prototype/views3.json`＋`v2_smoke_manifest`）与 dev_tune 的重叠
      清单；并排报"全量 250"与"剔除重叠"两版三指标；剔除版跌破判据线 →
      宣判文本必须如实注明。
    - 不达标 → 如实报告，不放宽不重跑。

# Stage 3 — 预登记消融（$45，可被 §2.4 砍）

一项：**v2.1 − route 独立腿**（分诊只挑叶不产 finding，即 RUN3 错误形态），跑
`splits/v2_ablation_subset`（80 项）。Wilson 95% CI；CI 重叠照写"不可区分"。
消融代码随包回传（或同文件 --ablate 开关，注明）。

# Stage 4 — 回放对决＋新前瞻窗（$50，补窗可被 §2.4 砍）

4.1 **诊断性回放（不是前瞻）**：同批 61 commit 跑 v2.1。**必须标注：v2.1 配方设计
    曾参考该窗口的失败案例（尸检引用过 bcf4c8fb），本节结果仅作诊断对照，禁止
    写成前瞻优势主张。** 报双口径触发率（gate_predicate 含/不含——61 项全部过
    主审，补 RUN3 的 15 项缺口）、与 v1（severe 口径 24/61；注明 RUN2 发布口径
    26/61 的差异原因）与 v2.0（0/61）的交并差、独有触发 top10（标注门影响）。
    "触发"定义 = 正式计分 finding。
4.2 `git fetch` 四仓；Megatron 新 commit ≥20 → **新前瞻窗**（这才是前瞻）；<20 →
    如实记录跳过，且**本轮不得出现任何前瞻性能结论**。达标结论若要对外使用，
    新前瞻窗（本轮或下轮）为必要条件。

# Stage 5 — 报告与回传（$10）

5.1 `reports/run4_report.md`：各 Stage 关键数字＋按 Stage 成本对账表（与
    usage_ledger 合计平账）；判据宣判；四方对照表（v1 / v2.0 / 本地 v2.1 / 本轮
    ——**本地列必须脚注"不同模型/裁判/口径，仅示方向，禁止跨列比大小"且不入
    宣判语句**）；tag **`run4-complete`**。
5.2 回传 zip 必含：judge 缓存（全）、usage_ledger、predictions/（全部）、
    detector_v2_1 代码＋消融开关＋冻结配置、冒烟与活性门全记录（含每轮迭代留痕）、
    全部 reports、git_tags.txt。
5.3 降级阶梯（预授权，依序）：①砍 Stage 3 → ②Stage 4 不补窗 → ③κ 100→60 且
    Nova 50→30 → ④dev 负样本减半（最后手段；触发即写明 FPR 判据自动降级为
    "只报点估计＋CI，不做 10%/15% 宣判"）→ ⑤仍超才停机等批准。
5.4 异常处理：锚检失败 / κ<0.7 两轮修不好 / 活性门 3 轮不过 / 契约无法照办 /
    误报早停触发 / error>5% → BLOCKERS 停机等 Rui。

## 完成判据（全打勾才算完）

- [ ] S0 tags＋锚检（含分歧方向）＋cache-only
- [ ] S1 契约 A–E 落实（报告附自查表：每条契约 → 对应代码行）；四处机械替换
      仅此四处；活性门四条全过＋行为指标全披露；FROZEN；usage 记账在位
- [ ] S2 250 全量；早停检查点执行记录；κ（绕缓存）≥0.7；Nova 50（或缺口说明）；
      判据按正式计分口径宣判；主表全要素；重叠敏感性两版并排
- [ ] S3 消融带 CI（或 §2.4 砍单留痕）
- [ ] S4 回放（61 全过主审＋双口径＋标注非前瞻）；新窗跑或如实跳过
- [ ] S5 对账表平账；回传清单逐项核对
- [ ] 全程 leak=0；route 不变量=0；总花费 ≤$500；超帽留痕

## 跑完之后

停下等 Rui。达标 → v2.1 成为新冻结版；下一步（新前瞻窗终验、单卡 GPU 微基准、
NVIDIA 材料）由 Rui 决定。不达标 → 如实报告＋消融与失败案例归因，主 agent 复检。
