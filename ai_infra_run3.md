# 执行机作业书 · 第三轮（RUN3：检测器 v2 ＋ 叶子验证表 ＋ 考题框架检验）

> **给执行机（EC2 + Bedrock，Claude Code）**。自包含：只读本文件即可开工。
> 版本：run3.v2（2026-07-19，经两路红队审查修订 25 处后定稿）。
> 工作根：RUN2 的目录（`ai_infra_run2/ai_infra_v011_work/`）。
> 模型纪律：一切 LLM 调用 = **Opus 4.8 标准通道**（Batch 不支持 Opus 4.x，勿试）。
> 预算：**总硬帽 $450**（分帽合计 $410，估算 ~$300）。各 Stage 结余可滚动给 Stage 4/5。
> 触帽 → 先走本文预授权的降级阶梯（见 Stage 4.6）；阶梯用尽才停机留痕等批准（宪章 6.2）。
> 成本一律从 API usage 实测记账。

## 本轮背景（30 秒）

RUN2 已证明 v1 的 leg3 独撑召回、leg1/leg2/对抗层在 test 净负。Mac 端本地小样已验证
v2 配方（**手册常驻＋仓库画像＋父快照上下文核验**）：引入 commit 场景 2/6 vs 裸模型
0/6（含 1 个 low 可检），误报 0/4；并发现疑似考题框架伪影（regfix case 用修复 commit
提问 → 方向错位 MISS）。本轮：把 v2 做成正式实现在 dev 全量量化、建 74 叶验证表、
用配对检验验证框架伪影。

## 不可违反的纪律（违反即本轮作废）

1. **test 本轮不做任何新的模型调用**（检测或判分都不行）。
   **唯一豁免**：对 RUN2 已有 test 产物（预测文件＋judge 缓存＋卡片元数据）做
   **只读重聚合**（Stage 0/1 需要）——必须在 cache-only 模式下进行（见 Stage 0.1），
   任何 cache miss 都不得触发实时判分。
2. 呈现规则同 `splits/protocol.md`：检测器输入 = PR-time 视图，后验字段屏蔽；
   `leak_attempt` 全程 0。
3. **一切资产（手册/画像/验证表/微基准冒烟）只准从 train 构建与验证**，
   dev 只用于调参与评测，案例 id 全部留痕以便回传核对 split 归属。
4. Rui 已明确：本轮不重跑 v1 与裸模型。参照系 = RUN2 dev 历史数字
   （`reports/detector_v1_dev.md`，同 dev_tune、同 judge 版本）：
   v1 冻结版 overall@2 21.3% / pair 13.3% / weighted FPR 12%；
   v1 最强变体（ablate_adversarial）overall@2 **25.3%** / pair 10.0% / FPR 19%。
   （baseline(b) 的 dev 数字在另一子集 dev_calib，仅量级参考，不进对比表。）
5. **判分规程（全轮统一）**：第一次判分即正式成绩，绝不回填修改；每个 Stage 的
   judge 调用记录模型 id 与日期；judge 缓存全程保留并回传。
6. 每个产物带 provenance（模型 id / prompt 版本 / 时间戳 / 成本 / 随机种子）。

---

# Stage 0 — 地基与欠账（$10）

0.1 **给 `eval_metrics.py` 加 cache-only 开关**（本轮第一件事）：
    环境变量 `JUDGE_CACHE_ONLY=1` 时 cache miss 直接抛错、绝不发起实时判分。
    Stage 0/1 的一切 test 相关重聚合必须在此模式下跑。
0.2 git 整备：先写 `.gitignore`（排除四仓 clone、大缓存、tmp；judge 缓存文件保留），
    再把 run2 工作目录纳入 git，tag **`run3-start`**；`git tag -l` → `reports/git_tags.txt`。
0.3 **从 RUN2 judge 缓存重建 `paper/tables/T1、T4`**（cache-only；禁止重新判分）：
    与 `baseline_results.md`/`detector_v1_results.md` 同源；miss 计数写入
    `reports/tables_rebuild_note.md`——**miss>0 → 该表标注不完整并写 BLOCKERS 等 Rui，
    不得自动补判**；缓存整体缺失 → 同样停在 BLOCKERS（Stage 1 降级为只做不需判分的
    统计）。同步修 `paper/claims.md` C4 的引用。
0.4 **judge 时漂锚检**（~$3）：从 RUN2 judge 缓存随机抽 30 条 v1 预测，用今天的
    judge（同 prompt 同模型 id）重判**一次性对照**（结果不写回主缓存）：一致率 ≥90%
    或 κ≥0.8 → 继续；否则停机写 BLOCKERS（跨轮对比的可比性需 Rui 裁决）。
    锚检结果写进 `reports/judge_anchor.md`。

# Stage 1 — 考题框架检验（$20）★ 证据整理，不下最终结论

**假设**：regfix 的 case 侧（修复 commit 提问"是否引入回归"）因方向错位被系统性压分。

1.1 用 pairs 文件的 `case_id` 字段做 **case↔pair 映射**（test 116 对的 fix-case 应
    100% 在 test case 集内——先验证并报告重合率）。对重合部分做 **McNemar 配对检验**
    （同一回归、修复侧 vs 引入侧两种问法，RUN2 全部 5 个选手各做一次，cache-only）。
    非重合部分只做描述性对比并声明混杂。
1.2 备择解释逐条检验（至少三项）：case/pair 两组的 static_detectability 分布、
    repo 分布、diff 大小分布——排除"框架效应其实是构成差异"。
1.3 对 dev 的 regfix case 中 `inducing_commit_traceable ∈ {direct, likely}` 且
    inducing sha 可在本地 clone 解析的卡，构建**引入侧 PR-time 视图**
    → `splits/dev_regfix_inducing_views.jsonl`；报告转换率与 converted 子集构成
    （n、detectability、repo）。
1.4 **修正口径的采纳规则（现在预登记）**：仅当 McNemar 在 ≥3/5 个选手上
    p<0.05 且方向一致（引入侧显著更易命中）时，Stage 4 才以修正口径为主叙事；
    否则旧口径为主、修正口径只作附表。
1.5 产出 `reports/framing_report.md`：定位为**证据整理**——分解表、配对检验、
    备择解释检验、转换率；「历史分数应如何重新解读」与北极星口径是否修改
    **留给 Rui 终审**（写入 Stage 6 等待项），本报告不下最终结论。
    另：注明「261 分母对同一回归双计」问题，作为 v0.3 修正建议。

# Stage 2 — v2 三件套资产（$80，全部 train-only）

## 2.1 全量叶子手册 `knowledge/handbook.v1.md`（$50）

74 叶每叶一页：典型反模式（2–4 条）/ 显现条件 top3 / 历史量级样本 / 检测时该查什么
（父快照核验要点）/ 高发文件 top5。
**防泄漏三件套**：①蒸馏 prompt 明文禁止引用输入 train 卡之外的任何具体案例、
commit sha、issue/PR 号、日期（模型参数记忆里的"著名案例"也不许写）；
②脚本审计：正则扫描手册中全部 sha/issue 号/PR 号/日期，逐一核对属于 train 卡集合，
外源引用即删并留痕 → `reports/handbook_leak_audit.md`（repo_profile 与
leaf_verification 同样过审）；③随机抽 10 叶第二次独立蒸馏对读：要点集合差异
过半 → 重写；对读表加"外源内容检查"一栏。

## 2.2 Repo-Profiler v1 `knowledge/repo_profile.v1.json`（$10）

四仓各一节，确定性脚本为主：热点文件榜（train perf 卡 × commit files，top50，
含 top 叶子）/ 模块→易感叶子表 / 排除表（tests|docs|examples|.github）/
Megatron 节加 recipe 覆盖提示（LLM 依据 recipe 配置推断一次，人读留痕）。

## 2.3 叶子验证表 `knowledge/leaf_verification.v1.json`（$20）

74 叶每叶三选一（LLM 起草＋规则校验）：
- `route-recipe`：继承/修订 `detector/leg3_recipe_map.json`（recipe＋配置提示＋观察指标）。
  **设计定位：这类测试永远由仓库方 CI 执行（NVIDIA CI 或 fork Actions），不由我们跑**。
- `microbench`：适合单机显现的叶——生成可执行模板
  `knowledge/microbench_templates/<leaf>.py`（pytest 风格，参数化{目标函数,输入,
  重复 N,固定种子}，核心=父快照与新代码各计时 N 次报中位数比值）。
- `not-verifiable-yet`：如实标注＋一句原因。
**冒烟纪律**：先 `nvidia-smi` 探测。有 GPU → 挑 3 个模板对 **train 卡**的已知案例
真实冒烟；无 GPU → 只对 CPU 可验叶冒烟 ≥2 个。**冒烟案例只准取 train 卡**（id 入
冒烟记录）。历史版本装不上（CUDA 依赖等）→ 允许降级为"不 import 目标仓库的合成
微基准"验证模板骨架本身可跑，记录标注 `repo-install-blocked, template-only smoke`，
不算违规。其余模板标 `untested`。**本轮只交表＋模板＋冒烟，GPU 全量执行属后续。**

# Stage 3 — detector_v2 实现（$40）

`scripts/detector_v2.py` + `detector/detector_v2.config.json`（冻结留痕）：

3.1 **画像门**：仅当改动文件**全部**落在排除目录（tests|docs|examples|.github）
    **且** diff 不触及任何 recipe/配置文件路径时才直接 no_issue。
    **冻结前必做**：在全部 train 正样本上跑排除规则，误杀率 >2% → 收窄规则；
    误杀率与被误杀样本清单留痕 `reports/gate_killrate.md`。
3.2 **大 commit 分解**：diff>400 行或 >10 文件 → 按文件/hunk 分组分别审、合并排序。
3.3 **主审**：手册常驻（按画像/初分类裁剪到相关 10–20 叶）＋DET_TOOLS 父快照核验，
    prompt 写死三问（热路径？调用方？默认配置启用？），宁缺毋滥。
3.4 **leg3 路由保留**：分类→叶子→查 leaf_verification，finding 附
    `suggested_benchmark` 或模板名。
3.5 无对抗层（RUN2 消融依据）；budget=2 按置信排序。
3.6 **冒烟集（写死）**：dev_calib 以种子 20260720 抽 18 个 case ＋ dev_tune 负样本
    同种子抽 7 个，共 25 项冒烟迭代 prompt（dev 专用允许）；这 7 个负样本在 Stage 4
    主表**照常计入**，报告脚注标明曾用于冒烟。定稿写 FROZEN 注记。
3.7 **成本投影检查点**：用冒烟 25 项实测 $/item 外推 Stage 4 总成本，写进
    `reports/cost_projection.md`；若外推超 Stage 4 分帽 → 直接启用 4.6 降级阶梯，
    不需要停机。

# Stage 4 — dev 全量评测（$200）★ 主实验

4.1 **逐项运行矩阵（写死）**：
    - 非转换项（普通 case＋负样本）：跑 1 次（原视图），双口径共用；
    - pair（30）：跑 1 次（引入侧，本来如此），双口径共用；
    - Stage 1.3 转换成功的 regfix case：**跑 2 次**——原视图（入旧口径）＋引入侧视图
      （入修正口径），预测分开存 `predictions/detector_v2/dev_tune.jsonl` 与
      `dev_tune_inducing.jsonl`；
    - 修正口径合分前按 (repo, inducing_sha) 检测 converted-case 与 pair 的碰撞：
      碰撞项只计一次，碰撞数进主表脚注，并给去重后 overall 作敏感性分析。
4.2 judge = `eval_judge.v1` 原样，缓存保存。**κ 抽检**：100 项按 正/负 × HIT/MISS
    分层抽样，第二次判分**绕过缓存**（user prompt 加一次性 nonce，结果不写回主缓存），
    κ≥0.7 才继续（<0.7 两轮修不好 → 停等 Rui）。**跨家族抽检**：50 项（刻意过采
    v2 的 HIT 项）用 Nova Pro 独立判，一致率入报告——本轮 Opus 全链路
    （手册/检测/判分同族）的自我偏好风险必须有此对照。
4.3 **预登记消融**（两项，子集 = 按 static_detectability × kind 分层随机抽
    50 正＋30 负，种子 20260720，**剔除与 3.6 冒烟集重叠项**）：
    (i) v2 − 手册；(ii) v2 − 父快照工具（画像门保留）。消融表强制带 Wilson 95% CI，
    CI 重叠 → 结论写"不可区分"。报告设"归因边界"一节：画像门与大 commit 分解的
    贡献本轮未隔离。
4.4 **成功判据（现在写死，全部旧口径＝与 RUN2 完全同视图同 judge 版本）**：
    - v2 dev overall@2 **≥ 25.3%**（v1 最强变体，detector_v1_dev.md ablate_adversarial 行）
    - v2 pair@2 **≥ 13.3%**（取 v1 两变体中更高者，从严）
    - weighted FPR **≤ 15%**（与 v1 的 12% 同定义同分层口径；此为本轮研究门，
      **产品红线仍是宪章 2.1 的 10%**，主表两条线并排标注）
    - 主表必须并报分层 FPR（尤其 hard-negative）与"被画像门直接判 no_issue 的
      正样本数及叶子分布"（>0 则逐条列出）
    修正口径的分数单独成节（含 converted/未转换子集分列），**不得**与 25.3% 直接比较；
    不达标 → 如实报告＋按消融归因，不放宽判据不改口径。
4.5 产出 `reports/detector_v2_dev.md`：主表（旧口径为主）、修正口径节、与 RUN2
    历史数字并排、消融表、κ/跨家族/锚检汇总、成本/item、失败案例 top10。
4.6 **预授权降级阶梯**（成本超投影时依序执行，每步留痕，不必停机）：
    ①砍消融 (ii) → ②消融子集减半 → ③负样本减到 50（分层等比）→
    ④仍不够才停机留痕等批准。

# Stage 5 — 前瞻对决（$45）

5.1 对 RUN2 前瞻的**同一批 61 个 commit**（`prospective/unseen_commits.json`）跑 v2：
    报**两个触发率**（含画像门/不含门），与 v1 的 26 个触发做交并差——被门直接
    排除的 commit 单列一类；双方独有触发 top10 清单（格式同 RUN2 top-20，标注
    是否受门影响），供 Rui 人工核验。
5.2 `git fetch` 四仓，若 2026-07-19 后 Megatron 新 commit ≥20 → 补一窗；否则如实记录。
5.3 产出 `reports/prospective_v2.md`。无标签 → 定性＋对比性结论，leak 纪律同前。

# Stage 6 — 报告与回传（$15）

- `reports/run3_report.md` 总览（各 Stage 关键数字＋实测花费）；更新 `report.md`；
  tag **`run3-complete`**。
- **等待 Rui 终审事项（列表写进 run3_report）**：framing 结论与北极星口径是否修改；
  修正口径是否升级为 v0.3 标准。
- **回传 zip 必含**（宪章 7.4）：全部 judge 缓存（含本轮）、`reports/git_tags.txt`、
  **`paper/` 全目录**（重建后的 T1/T4＋claims）、knowledge/ 三件套＋
  microbench_templates/＋泄漏审计、detector_v2 全套、predictions/（本轮全部，含
  dev_tune_inducing）、splits/ 新增文件、全部 reports/。

## 完成判据（全打勾才算完）

- [ ] S0：cache-only 开关落地；tags 归档；T1/T4 同源重建（miss 处理合规）；锚检通过
- [ ] S1：case↔pair 映射＋重合率；McNemar×5 选手；备择解释三项；引入侧视图＋转换率；
      framing_report（证据整理定位）；期间对 test 零新模型调用
- [ ] S2：handbook 74 叶＋泄漏审计＋10 叶对读；repo_profile 4 仓；leaf_verification
      74 叶全覆盖＋模板＋train-only 冒烟记录
- [ ] S3：画像门 train 误杀率留痕（≤2%）；冒烟集按规格；冻结注记；成本投影
- [ ] S4：运行矩阵合规（converted 双跑）；κ 绕缓存抽检 ≥0.7；跨家族 50 项；两项消融
      带 CI；成功判据按旧口径宣判；碰撞去重敏感性
- [ ] S5：61 commit 对决（双触发率＋交并差＋top10）；新窗口跑或如实记录
- [ ] S6：run3_report＋tag＋Rui 终审事项清单；回传清单逐项核对（含 paper/）
- [ ] 全程 leak_attempt=0；总花费 ≤$450；降级阶梯之外的超帽有当时书面留痕

## 异常处理

- 锚检失败 / judge κ<0.7 两轮修不好 / T1-T4 缓存 miss>0 → 停，写 BLOCKERS 等 Rui
- 某叶无 train 卡或资产构建失败 → 如实标注跳过，不编造
- Stage 4 前发现 v2 配置要改 → 允许（dev 专用），重跑 3.6 冒烟并记录改动

## 跑完之后

停下等 Rui。届时应有：v2 的 dev 定量成绩（旧口径硬对比＋修正口径附表）、增益归因、
74 叶验证表＋模板、框架效应的配对检验证据、与 v1 的前瞻对决。
下一步（test 级终验走新前瞻窗口、单卡 GPU 微基准执行、NVIDIA 接触）由 Rui 决定。
