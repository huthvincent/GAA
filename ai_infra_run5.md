# 执行机作业书 · 第五轮（RUN5：轻量轮——工作点定档＋工具修补＋滚动前瞻）

> **给执行机（EC2 + Bedrock，Claude Code）**。自包含。版本 run5.v1（2026-07-23）。
> 工作根：run4 目录。模型：Opus 4.8 标准通道。版本 run5.v2（红队 8 条修订后定稿）。
> 预算：**总硬帽 $100，估 ~$50**。usage 台账逐调用记账（含 judge，见 S2.1）。

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
- [ ] S1 op 配置＋依据注明；detector 代码/prompt 与 RUN4 diff = 空（新增 `detector_v2_1_op.config.json` 与报表生成器改动除外）
- [ ] S2 五项修补落地；RUN4 报表 cache-only 重生成 0 miss 且数字无漂移（新表除外）
- [ ] S3 Megatron 窗跑或如实跳过（<15）；双口径＋top-10；vLLM 窗跑或如实跳过
- [ ] S4 对账平账；回传清单齐
- [ ] 全程 leak=0；花费 ≤$100

## 跑完之后
停下等 Rui。届时产品态＝ v2.1@0.6 定档配置 ＋ 持续累积的前瞻证据（Megatron 第 2 窗
＋可能的 vLLM 首窗）→ 下一步（fork CI 接线 / NVIDIA 材料 / v2.3 深审实验设计）由
Rui 与主 agent 决定。
