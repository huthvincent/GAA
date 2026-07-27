# 执行机作业书 · 第五轮（RUN5：轻量轮——工作点定档＋工具修补＋滚动前瞻）

> **给执行机（EC2 + Bedrock，Claude Code）**。自包含。
> 工作根：run4 目录。模型：Opus 4.8 标准通道。
> **版本 run5.v3（2026-07-26，七条复核勘误后修订；前身 run5.v2 红队定稿）**。
> 预算：**总硬帽 $100，估 ~$50**。usage 台账逐调用记账（含 judge，见 S2.1）。

> ## ⚠️ run5.v3 相对 v2 的变更（开工前必读）
> Mac 端 2026-07-26 做了一轮七路交叉核验，发现 **1 个会直接污染本轮 S2.2 产出的真 bug**
> ＋ 6 处表述问题。新增 **S2.7（修字段名地雷）** 与 **S2.8（修成本漏计）**，且 S2.2
> 的实现铁律追加一条"**禁止走 `eval_metrics.cmd_score`**"。其余 Stage 不变。
> 若你已经开始跑 v2：S1/S3 成果可留，**S2.2 若已产出分带表必须按 S2.7 修复后重生成**。

## 与你同步：RUN4 的终审结果（先读这节）

1. **验收**：四轮最干净——契约逐字、全部关键数字独立重算吻合、κ 非空卷、账平到分。
   判据宣判维持你的原判（recall/pair ✓、FPR 23% ✗）。
2. **FPR 处置（已裁决）**：主 agent 用你回传的预测＋judge 缓存做了展示阈值扫描
   （150/150 键重建映射）：**工作点 conf_raw≥0.6 → recall 43.3% / pair 13.3% /
   FPR 2%——三轴帕累托超 v1，产品红线 10% 达标**。不需要重跑任何东西。
3. **哨兵 +15.7pp 复判＝良好校准，非博弈**：命中 finding conf_raw 中位 0.60 vs
   负样本 0.45，分离度 0.866。你的标注按规则打得对，规则本身要改（见 S2.3）。
4. 你的五个待决事项全部已裁：FPR→展示门；哨兵→校准解读；叶一致 61%→接受、
   分列表述；judge 接 ledger→批准（本轮做）；新窗 11/22→采纳为可用性证据起点。
5. **Mac 端 A/B 已否决 v2.2 的 prompt 改动**（佐证压分规则全局伤召回、深审强化
   信号太弱）——**因此本轮检测器 prompt/权重/配置一个字节都不改**。
6. **七条复核勘误（2026-07-26，run5.v3 新增同步）**：Mac 端七路交叉核验修正了
   RUN4 报告的 6 处表述 ＋ 发现 1 个真 bug（→ S2.7）。与你本轮工作直接相关的是：
   - **"三轴帕累托超 v1"是错的**：pair 13.3% vs 13.3% 完全持平。报表里凡出现
     "三轴"字样一律改为"两轴显著改进（召回 21.3%→43.3%、误报 12%→2%）、pair 持平"。
   - **"deep 腿误报为零"不准**：有 2 条 deep finding 在 conf_raw=0.50 越线
     （neg:vllm@eea3024f43e0、neg:TransformerEngine@3275e1a0f4da），只是从未
     **单独**造成误报项（deep-only FP 项＝0）。分带表里请如实分列。
   - **"leak=0"与"route 不变量=0"都是结构性恒真**（`_guard_commitish` 零调用点；
     合成分支先置 `has_route=True` 使第二个 if 不可达）。报表禁止再把它们表述为
     "机制已验证"，改为结构性论证＋"模型 250 次全部自产 route finding、兜底合成 0 次"
     这个真事实（宪章新增 7.10）。
   - **展示门 0.6 是 dev 拟合、未见数据上未验证**。S3 的前瞻窗产出的是**报警率**，
     **不是精确率**——报表里必须写明"无真值标注，11/22 不代表 11 个真有问题"
     （宪章新增 7.11）。
   - **Mac-40 重叠已补算**：`views3.json` 40/40 全在 dev_tune 内（16 neg/14 pair/
     10 case）。剔除后 recall 103/126=81.7%、pair 4/16=25.0%、FPR 16/84=19.0%；
     并集剔除（52 项）后 81.0%/25.0%/19.5%，**判据结论不翻转**。请把这三行并入
     S2.2 的重叠敏感性表，替换原"Mac-40 清单未回传→无法评估"的注记。
   - 成本口径：只能引 ledger 的 $127.49，predictions 的 cost 求和漏计 scan-big（→ S2.8）。

## 纪律

test 零调用；dev 本轮也不重跑（没有检测器变更，重跑无意义）；leak=0；
detector prompt 与 RUN4 冻结版 byte-identical（禁改）；judge 缓存/台账随包回传。

# Stage 0 — 地基（$2）
tag `run5-start`；cache-only 开关确认；`git tag -l` → reports/git_tags.txt。

# Stage 1 — 工作点定档（$0）
新建 `detector/detector_v2_1_op.config.json` = RUN4 冻结配置 ＋
`"display_threshold": {"conf_raw": 0.6, "scope": "display-only"}`：
**展示层规则，不改检测、不改研究计分**（研究报表仍按 metrics.v1 全量报）。
注明依据：`RUN4_ACCEPTANCE_AND_OPERATING_POINT.md` 裁决。

# Stage 2 — 工具修补（$8，全部零模型调用或极少量）
2.1 judge 调用接入 usage_ledger（真实 tokens/cost；label=judge）。
2.2 报表生成器补两表：source×conf_raw 带分层 recall/FPR；对称口径附表
    （RUN4 完成判据欠账）。**实现铁律：两表一律从既有判分结果派生**——用已存
    `hit_finding_index` 对应的命中 finding 的 conf_raw 分带/过滤（对称 HIT ＝
    命中 finding conf_raw≥0.5），**禁止对任何过滤后的 findings 列表重新调 judge**
    （缓存键含完整列表，重判必 100% miss）。用已有缓存 cache-only 重生成 RUN4
    报表（miss>0 → 停）。**"无漂移"判定口径**：`detector_v2_1_summary.json` 的
    overall2/pair2/fpr/route_hit/deep_hit/route_leaf_agree/errors 各字段与 RUN4 版
    逐字段相等（sentinel 字段按 2.3 改口径豁免），任一不等 → 停机上报。
2.3 哨兵条款改口径：改报"命中 finding vs 负样本 severe finding 的 conf_raw
    分离度（AUC 近似）"，≥0.7 记"校准良好"；band-share 差值仍并报但不再单独触发
    "博弈嫌疑"标注。
2.4 报表自动附注：κ 分层样本不足时自动写明缺口原因；成本调用次数一律以
    ledger 行数为准（修 RUN4 的 1158/1246/2493 文案乱象）。
2.5 微修 RUN4 报告两处（S2 标题行 $123→~$79 重复相加；调用次数改按 ledger 行数）。
    **留痕合规（宪章 11.1/6.2）**：run4_report.md 头部加日期化修订注记
    （"[run5 修订] 原字节版本见 tag run4-complete；修订项：…"），run5_report.md 复述。
2.6 judge 接 ledger 的端到端验证：允许 1–2 次真实 judge 冒烟调用（~$0.05）确认
    label=judge 正确落账后即停，除此之外全程 cache-only。

2.7 **【新增·必做·会污染 S2.2】修 `eval_metrics.py` 的字段名地雷**（$0）。
    `topk_severe()`（约 188 行）与 `score_predictions()` 内的 `is_fp()`（约 345 行）
    读的都是 `f.get("confidence", 0)`，但 **v2.1 的 finding 没有 `confidence` 字段**
    （字段并集＝category/claim/conf_final/conf_raw/file/severity/source/
    suggested_benchmark/unverified）。后果：给 v2.1 打分时 `is_fp` **恒为 False
    → FPR 静默报 0.0%**，`topk_severe` 排序键恒为 0 → 退化成保留原顺序。**不抛异常。**
    - RUN4 已发布的 23/100 **不受影响**（走的是 `detector_v2_1_report.py`，第 93 行读
      `conf_raw`，正确）。风险是本轮 S2.2 若走 `eval_metrics` 会拿到假 FPR。
    - 改法：两处一律改成双兼容 `f.get("conf_raw", f.get("confidence", 0))`（与
      `detector_v2_1.py:_finalize` 约 229 行的既有兼容写法同风格）。
    - **追加退化告警**：任一计分函数在"全体负样本零越线"或"全体分数相同"时必须
      打警告并在报表中标注——这是字段读空的唯一指纹（宪章新增 7.9(b)）。
    - 验收：改完对 **v1 的历史预测**重跑一遍，确认数字与 RUN2 发布值逐位不变（回归无损）；
      对 v2.1 预测跑一遍，确认 FPR 得到 23/100 而非 0。两个结果都写进 run5_report。
    - **S2.2 铁律追加**：分带表/对称口径表**禁止调用 `eval_metrics.cmd_score`**，
      一律从 `detector_v2_1_report.py` 的路径派生。

2.8 **【新增】修成本漏计**（$0）。`_scan_blocks` 里 `UL.record(_STAGE,"scan-big",...)`
    的返回值被丢弃（`detector_v2_1.py` 约 341 行），从未累加进 `meta["cost"]`，
    导致 predictions 落盘的 per-item cost 与 `--max-cost-usd` 熔断累加器**系统性漏计
    大 commit 快扫那一档**（熔断会比真实花费晚触发，是预算风险）。
    - 铁证（Mac 已复算）：新窗 predictions cost 求和 $4.5084 ＝ ledger 的
      detect $1.5466 ＋ deep-big $2.5918 ＋ triage-big $0.3699；差额恰为
      scan-big $0.5226，真实合计 $5.0309。
    - 改法：把 `_scan_blocks` 的记账返回值累加进 `meta["cost"]`（需把 meta 或一个
      cost 累加器传进去）。**注意这会改变 predictions 的 cost 字段值**——属修 bug 不属
      改检测逻辑，findings 输出必须逐字节不变；请在 report 里声明"cost 字段口径修正"。
    - 另：`replay` 的 $11.47 vs ledger $14.71 差额里 scan-big 只占 $1.151，其余约 $2.1
      疑为断点续跑重复调用（S4-replay 的 detect 有 111 次调用但只有 61 个 item）。
      请顺手核一下续跑是否会重复计费，结论写进报告（不必修）。

# Stage 3 — 滚动前瞻窗（$35）★ 本轮唯一新数据
3.1 `git fetch` 四仓。Megatron 候选＝fetch 到的全部新 commit **按 sha 排除
    `prospective/new_window_run4.json` 的 22 个**（防与 RUN4 第 1 窗重叠双计），
    去重后 ≥15 → 开窗跑冻结版 v2.1（budget=2）；**每窗硬上限：最新 40 个**（超出
    截断并按宪章 2.3 如实披露）。报双口径：无门槛触发率 ＋ **展示门 0.6 下触发率**
    （产品口径）＋ top-10 清单（供 Rui 核验）。
3.2 vLLM 同窗（可选）：同样排重、同样 ≤40 上限；≥15 → 跑一窗（首次跨仓前瞻）；
    不足则如实跳过。
3.3 无标签 → 定性产出；leak 纪律同前。

# Stage 4 — 报告与回传（$5）
`reports/run5_report.md`（各 Stage＋成本对账平账）；tag `run5-complete`；
回传 zip：新配置、修补后的报表与生成器（**含修订后的 run4_report.md 与修订注记**）、
前瞻预测与报告、完整 ledger、judge 缓存增量、git_tags.txt、**MANIFEST.txt＋逐文件
md5**（宪章 5.2）。

## 完成判据
- [ ] S1 op 配置＋依据注明；detector 代码/prompt 与 RUN4 diff = 空（新增 `detector_v2_1_op.config.json`、报表生成器改动、S2.7/S2.8 的两处 bug 修复除外——**prompt 与 findings 输出必须逐字节不变**）
- [ ] S2 七项修补落地；RUN4 报表 cache-only 重生成 0 miss 且数字无漂移（新表除外）
- [ ] **S2.7 字段名地雷已修**：v1 历史预测重算数字逐位不变；v2.1 预测 FPR 得 23/100 而非 0；退化告警已加
- [ ] **S2.8 成本漏计已修**＋续跑重复计费结论已写；报表声明 cost 字段口径修正
- [ ] 七条勘误的表述修正已落地（"三轴"改写、deep 误报分列、恒真式护栏改表述、报警率≠精确率、Mac-40 三行入重叠表、成本只引 ledger）
- [ ] S3 Megatron 窗跑或如实跳过（<15）；双口径＋top-10；vLLM 窗跑或如实跳过
- [ ] S4 对账平账；回传清单齐
- [ ] 全程 leak=0；花费 ≤$100

## 跑完之后
停下等 Rui。届时产品态＝ v2.1@0.6 定档配置 ＋ 持续累积的前瞻证据（Megatron 第 2 窗
＋可能的 vLLM 首窗）→ 下一步（fork CI 接线 / NVIDIA 材料 / v2.3 深审实验设计）由
Rui 与主 agent 决定。
