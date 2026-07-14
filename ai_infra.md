# AI Infra 性能回归数据集（Phase 0）— 执行计划

> 项目代号：**MegaPerfBench**。目标：为 Megatron-LM 性能回归检测 agent 构建地基数据集。
> 模型纪律：**所有 LLM 调用一律使用 `claude-opus-4-8`**（Tier 1 走 Batch API 五折，其余走标准 API）。
> 预算硬帽：**$6,000 API 成本**（中心估计 $3,000–4,800）。日历时间：2–3 周（含人工仲裁）。
> 本文档自包含：执行者不需要读任何其他对话记录即可开工。

---

## 0. 一页摘要

把四个 AI infra 仓库（Megatron-LM / vLLM / DeepSpeed / TransformerEngine）的 **33,085 个 commit 全量**过一遍三层漏斗（确定性路由 → Batch 筛查 → agentic 深读），加上 issue 流、golden-values 流、revert 反向流、学术数据集引导四路补充信号，产出三件套：

1. **cases.jsonl** — ~1,000–1,500 张结构化性能 bug case card（含证据链）
2. **regression_pairs.jsonl** — ~250–400 个 `(引入 commit, 修复 commit, 表现条件, 量级)` 回归对，分 A/B/C 证据级
3. **冻结的评测协议** — train/dev/test 切分 + 防泄漏规则 + 指标定义 + baseline 运行结果

所有产出进入新建**私有**仓库 `huthvincent/MegaPerfBench`（目录规范见 §8——这是本计划最重要的一节，后续所有 Phase 都消费这些文件）。

**为什么全量而不是关键词过滤**：pilot（2026-07-13，48 commit 分层人审级标注）实测关键词过滤 precision ≈60% 但 recall 未知；大量性能 bug 的 commit message 是中性的。全量处理买到：无偏分类学、可测量的过滤 recall、全量良性池（hard negatives + 真实基率）、更完整的回归对。

---

## 1. 背景与关键前置事实

- **要打败的 baseline**：NVIDIA/Megatron-LM 现有 `.github/workflows/claude_review.yml` 的 `/claude strict-review`（手写 rubric：dtype、TP/PP/SP/CP/EP、`.item()` 同步、多余 collectives、CUDA-graph 兼容性），claude[bot] 已评论 329 个 PR；另有 Greptile 自动 review 和 golden-values/perf-baseline 动态 CI。我们的增量 = 把手写 rubric 变成数据驱动可进化的知识库 + 与 perf CI 联动。
- **GitHub Advisory Database 不含性能 bug**（只收安全漏洞，CIA 定义）——不要用它。
- **不存在公开的"ML infra 性能回归 inducing commits"数据集**——本数据集本身是贡献（可发 dataset paper）。
- **分类学规模先验**：文献一致收敛在 12–34 类（如 TF/PyTorch perf bug 12 大类 19 子类；DeepSpeed/Megatron/Colossal 849 issue 归 34 症状/28 根因）。目标两层、30–50 叶子类，不是几千类。
- **Pilot 实测数字**（校准一切估算的锚点，产物在 `~/Desktop/OPB/perf-agent/phase0/pilot/`）：
  - 关键词候选 1,226 个（Megatron 262 / vLLM 689 / DeepSpeed 132 / TE 143）
  - 48 样本人审级标注：60% 真性能相关；构成 optimization 19 / regression-fix 6 / config-default-change 3
  - regression-fix 的 inducing commit 可追溯率 5/6（direct 4）
  - inducing change 静态可标记率 12/13（high 4 / medium 8）→ agent 前提成立
  - 83% 环境条件相关（并行配置/dtype/硬件/shape）→ manifest 条件必须记录
  - Revert 直接挖低产（vLLM 187 个中仅 2–4 个 perf-motivated；Megatron 0 documented）；但"被 revert 的性能优化"是自带 benchmark 数字的反向金矿
  - vLLM issues 最富矿（503 个 `performance` label）但正式 closed-by 链接仅 ~7%，修复关系要从评论文本挖
  - Megatron golden-values 323 次变更中仅 37/332 test case 真正 gate iteration-time；join 靠时间窗；含"被接受的回归"（如 `6204b925f`）
  - 性能回归通常 fix-forward 不 revert → fix-based agentic SZZ 是主回溯路径

---

## 2. 范围与操作性定义

### 2.1 什么算"性能 bug"（收录标准）

影响**已发布代码运行时性能**的问题，症状限于：

| symptom | 说明 |
|---|---|
| `throughput` | tokens/sec、iteration time、samples/sec 下降 |
| `latency` | TTFT/TPOT/单步延迟上升 |
| `memory` | 峰值显存/内存上升、泄漏、碎片化、同配置 OOM |
| `gpu-util-or-bubble` | GPU 利用率下降、pipeline bubble、同步空转 |
| `hang` | 死锁/卡死（性能的极端形态） |
| `compile-or-startup-time` | 编译/启动/capture 时间显著上升（次要，单独标记） |

**排除**：纯功能 bug、崩溃（除 hang）、数值精度问题（除非它导致静默的性能路径切换，如 fused kernel 被绕开）、测试/benchmark 脚本自身的修复（标 `perf-infra-or-test`，保留在旁路桶）、文档。

### 2.2 kind 五分类（pilot 验证过的标注口径）

- `regression-fix`：恢复被某个更早 commit 损害的性能 → 是回归对的来源
- `optimization`：把"从来就慢"的东西变快（latent inefficiency）→ 进案例库，不进回归对
- `config-default-change`：默认配置变化导致的性能变化
- `perf-infra-or-test`：只动 benchmark/CI/golden values → 旁路桶（烟雾信号）
- `not-perf`：误报

### 2.3 数据源

| 源 | 范围 |
|---|---|
| S1 commit 流 | 4 仓库全量 33,085 commits（Megatron-LM 9,214 / vLLM 18,699 / DeepSpeed 3,225 / TransformerEngine 1,947，统计于 2026-07-13） |
| S2 issue 流 | vLLM `performance` label 503 个 + Megatron/DeepSpeed 关键词命中（~150 个候选） |
| S3 revert 反向流 | 两仓库 329 个 revert，重点挖"被 revert 的性能优化" |
| S4 golden-values 流 | Megatron 323 次 golden 变更 + 6 次 baseline_values 变更 + 2026 年 CI 报的回归 issue（#5692、#5781） |
| S5 学术引导 | FSE'22 58 例可复现 benchmark（Zenodo 7060209）；向 arXiv 2512.20345（849 例含 Megatron）与 2506.10426（308 例）作者邮件要标注数据；GSO/SWE-Perf 的 perf-improving commits 反向用 |

---

## 3. 主管线：三层漏斗（S1 commit 流）

### Tier 0 — 确定性路由（零成本）

对每个 commit 按改动路径和元数据分桶，**不丢弃任何桶**：

- `skip-docs`：只动 `*.md`/`docs/`/LICENSE
- `skip-format`：纯 lint/format（diff 无语义变化，用启发式：只有空白/import 排序）
- `smoke-ci`：只动 CI recipe/golden values/benchmark 配置 → 转入 S4 子管线（烟雾信号：关闭 throughput 测试的 commit 是回归发生过的线索）
- `screen`：其余全部进 Tier 1（预计 ~28–30k）

**输出**：`screening/tier0_routing.jsonl`（见 §8 schema）。

### Tier 1 — 全量筛查（Batch API + Opus 4.8）

每个 commit 一次**非 agentic** 调用。输入打包（离线用 git 生成，不联网）：

- commit message 全文 + author/date
- `git show --stat` 的文件清单
- diff 裁剪：优先保留非测试代码的 hunk，总量 ≤ 6k tokens（超长 diff 按文件相关性采样，记 `diff_truncated: true`）

Prompt 要点：给出 §2 的定义 + pilot gold cards 中的 8–10 个 few-shot（覆盖五个 kind）；structured output 强制 JSON（schema 见 §8.3）。判定字段核心是 `perf_relevance`（0–1）、`kind_guess`、`needs_deep_read`。

**校准闸门（先行，不通过不放全量）**：
1. 用 pilot 的 48 张 gold cards + 1,226 关键词候选跑校准批（成本 ~$30）
2. 调阈值使 **对已知阳性（29 个 perf-related，尤其 6 个 regression-fix）的 recall ≥ 95%**——漏检在这层不可恢复，误报只是多花 Tier 2 的钱
3. 校准报告写入 `reports/tier1_calibration.md`（混淆矩阵、阈值选择、token 实测均值）

**工程**：Message Batches API（`custom_id = {repo}@{sha}`，单 batch ≤10 万请求，24h 内出结果，五折）；结果乱序返回按 custom_id 归位；幂等（每 sha 一条记录，重跑跳过已有）。

**输出**：`screening/tier1_screen.jsonl` + `reports/tier1_summary.md`（每仓库的阳性率、kind 分布、token/成本实测）。

### Tier 2 — agentic 深读（标准 API + Opus 4.8 worker 池）

对象：Tier 1 判 `needs_deep_read` 的 commit（预计 3–5k）。每个 commit 一个 agent，工具三件套：

- `git_show(sha, path?)`：看 diff/文件（本地全量 clone，见 §7）
- `git_log(args)`：追历史（`-S`、`--follow` 等）
- `fetch_pr(repo, number)`：抓 GitHub PR 正文+评论（本地磁盘缓存，避免重复请求）

产出完整 case card（schema 见 §8.4）。**铁律：每个判断字段必须附 `evidence` 句，引用 diff 行号或 PR 原话**——不许凭感觉填。

**质检**：随机 20% 的深读由第二个独立 agent 盲标；字段级一致率入 `reports/tier2_agreement.md`；分歧样本进人工仲裁队列（§6）。

**并发**：30–50 workers；prompt 前缀（系统提示+定义+few-shot）冻结以吃 prompt caching（0.1× 缓存读价）；SDK 自动重试 429/5xx。

**输出**：`cases/cards.jsonl`（含双标记录）+ `reports/tier2_summary.md`。

### Tier 3 — inducing commit 回溯（agentic SZZ，3 票制）

对象：深读判 `regression-fix` 的案例（预计 500–800）。

- **Tier A（直接命中）**：revert 指认 / issue 里用户 bisect / commit message 明说 "regression introduced by #X" / CI 报的 commit range → 直接收录，标 `evidence_tier: A`
- **Tier B（agentic 回溯）**：3 个独立 agent，各自从 fix diff 提取关键符号 → `git log -S` 追引入点 → 回答"这段慢是某 commit 引入的，还是从来就慢？"。**≥2 票指向同一 commit 才收**，标 `evidence_tier: B`；注意文献确认的坑：性能 fix 经常不触碰引入行（在别处加 cache）——agent 必须显式区分 regression 与 latent inefficiency
- **Tier C（无法回溯）**：保留为案例，不进 pairs

**输出**：`pairs/regression_pairs.jsonl` + `reports/tier3_summary.md`（各 tier 数量、投票一致率）。

---

## 4. 补充信号源子管线

### S2 issue 流

1. 拉取 vLLM `performance` label 全量 503 个 + Megatron/DeepSpeed 关键词检索命中（`slower`/`regression`/`throughput drop`/`memory leak`）
2. 每个 issue 一个 agent 深读（正文+全部评论）：提取量化数字（tok/s、TTFT、%）、版本区间、复现配置（TP/PP/EP、硬件、模型）、**评论中的修复/肇事线索**（"fixed by #43245"、"reverting #41181 fixes it"——正式 closed-by 链接只有 ~7%，主要靠评论挖掘）
3. issue 记录与 commit 流的 case card 做 join（同一 bug 的 issue 侧信息合并进 card 的 `issue_refs`，量级/配置字段以 issue 实测为准）

**输出**：`raw/issues/{repo}.jsonl`（原始）+ join 后回填 `cases/cards.jsonl`。

### S3 revert 反向流

1. 列出两仓库全部 revert commit
2. agent 分类动机（perf / functional / build / accuracy / unknown）
3. **重点收"被 revert 的性能优化"**：revert 本身是已知致慢 commit，原 PR 里常有 benchmark 数字 → 直接生成 Tier A pair（`inducing = revert commit`，`fix = 后续 re-land（若有）`）
4. 少量 perf-motivated revert（如 vLLM #33771）直接给出 inducing commit + 量级 → Tier A

**输出**：`raw/reverts.jsonl` + 回填 pairs。

### S4 golden-values 流（Megatron 特有，需激进过滤）

1. 只看 **37 个真正 gate iteration-time 的 test case**（`model_config.yaml` 的 METRICS 含 iteration-time）
2. 对这些 golden 文件的每次变更，解析 iteration-time 中位数的方向和幅度；>5% 变慢且出现在专门 re-baseline commit 中 → 标记"**被接受的回归**"
3. join 肇事 commit 用时间窗：同一 golden 文件路径（编码了 test×硬件）两次变更之间的 commits 为候选集，agent 排查
4. 排除已知混杂：container 升级 commit（~24 个）、GB200 集群搬迁批次单独标 `env-confounded`
5. 2026 年 CI 自动报的回归 issue（#5692 GB200 -18.7%、#5781 MoE step-time）直接生成带测量的记录

**输出**：`raw/golden_values/megatron_churn.jsonl` + 回填 pairs（tier A：有测量的）。

### S5 学术引导

- 下载 FSE'22 Zenodo 数据 + GSO/SWE-Perf 的 perf-improving commits 清单，转换成 card 格式并打 `source: academic` 标（分类学归纳时用，不进主评测集，避免许可/口径混杂）
- 邮件模板发给 2512.20345 / 2506.10426 作者要标注数据（Rui 发）

---

## 5. 负样本构建（三层，决定评测有没有意义）

利用全量筛查的副产品——每个 commit 都有 `perf_relevance` 分数：

1. **随机良性**：各仓库按时间分层随机抽 `perf_relevance < 0.2` 且深读未推翻的 commit，量为正样本 3–5 倍
2. **hard negatives（最值钱）**：`perf_relevance ∈ [0.4, 0.7]` 且深读判 not-perf 的（"长得像但不是"）+ 触碰热点文件（与回归对同文件）但 golden values 未变动的 commit
3. **假信号**：`perf-infra-or-test` 桶（改 benchmark 脚本、关 CI 测试——现实中最容易骗过检测器）

**输出**：`negatives/negatives.jsonl`（schema 同 card 的精简版，含 negative_type 字段）。

---

## 6. 人工仲裁（Rui，预计 3–5 天）

仲裁队列来源：Tier 2 双标分歧 + Tier 3 投票分歧 + 全部 Tier A/B pairs 逐条过目 + 全量 cards 的 10% 分层随机审计。

- 工具：仲裁就在 `cases/audits/` 下追加 JSONL 记录（schema §8.6），不改原 card——**机器标注永远保留，人工裁决叠加**，两者都可追溯
- 报告：`reports/human_audit.md`——字段级 agreement、Cohen's kappa（人 vs 机、机 vs 机）、按 kind 的修正率
- kappa < 0.6 的字段 → 修 prompt 定义后重跑该字段（只重跑受影响子集）

---

## 7. 工程纪律（会翻车的地方）

1. **全量 clone 先行**：现有四个 clone 是 blobless 的，33k 次 `git show` 会逐 blob 打 GitHub。开工前 `git fetch --refetch`（或重新完整 clone）。磁盘几个 GB
2. **幂等 + 断点续跑**：所有 stage 输出 JSONL 按 key（sha / issue# / case_id）幂等；重启跳过已有；每条机器标注带 provenance（§8.2）
3. **Prompt 版本化**：所有 prompt 存 `prompts/` 目录，文件名带版本号；每条标注记录引用 `prompt_version`——换 prompt 必须换版本号，禁止原地改
4. **GitHub API 缓存**：PR/issue 响应落盘 `raw/gh_cache/`（token 5k req/h；带 ETag 复用）
5. **确定性裁剪**：diff 裁剪算法固定且版本化（影响可复现性）
6. **成本监控**：每个 stage 记录实测 token 用量到 report；累计超 $4,500 时暂停并向 Rui 汇报（硬帽 $6,000）
7. **不用 tiktoken 估 token**——用 `count_tokens` API 按 Opus 4.8 tokenizer 校准

---

## 8. Output 规范 ★（后续所有 Phase 的消费接口）

### 8.1 仓库与目录

新建**私有**仓库 `huthvincent/MegaPerfBench`（数据从第一天按可公开标准构建：全部 provenance 可追溯、上游 license 记录在 DATASHEET；发布与否后议）。目录：

```
MegaPerfBench/
├── README.md                  # 数据集总览 + 快速上手（如何加载、字段含义索引）
├── DATASHEET.md               # datasheet for datasets：来源、license、构建方法、已知偏差
├── schemas/                   # 所有 JSON Schema，文件名带版本（card.v1.schema.json 等）
├── prompts/                   # 所有 prompt 模板，带版本号
├── raw/
│   ├── commits/{repo}.jsonl   # 全量 commit 元数据（sha, date, author, subject, files, stats）
│   ├── issues/{repo}.jsonl    # issue 深读记录
│   ├── reverts.jsonl
│   ├── golden_values/megatron_churn.jsonl
│   └── gh_cache/              # GitHub API 响应缓存（gitignore，不提交）
├── screening/
│   ├── tier0_routing.jsonl
│   └── tier1_screen.jsonl
├── cases/
│   ├── cards.jsonl            # ★ 核心产出 1
│   └── audits/audit_log.jsonl # 人工仲裁记录（append-only）
├── pairs/
│   └── regression_pairs.jsonl # ★ 核心产出 2
├── negatives/
│   └── negatives.jsonl
├── taxonomy/
│   └── taxonomy.yaml          # Phase 1 产出，Phase 0 先放 kind 五分类
├── splits/
│   ├── protocol.md            # ★ 核心产出 3：评测协议全文
│   ├── train.txt / dev.txt / test.txt   # 每行一个 case_id / pair_id
│   └── FROZEN                 # 空文件，出现即表示切分已冻结，禁止再改
├── reports/                   # 每 stage 一份 md 报告（校准、汇总、一致率、审计、成本）
└── scripts/                   # 管线代码（Phase 0 执行时写入）
```

**规则**：数据文件全部 JSONL（一行一记录，UTF-8）；总量预计 <50MB，plain git 即可；`raw/gh_cache/` 进 .gitignore；每次大批量写入单独 commit，message 注明 stage 和数量。

### 8.2 通用字段：ID 与 provenance

- `case_id`: `{repo}@{sha前12位}`，如 `Megatron-LM@72171c0d531d`
- `pair_id`: `{repo}@{inducing_sha12}->{fix_sha12}`
- 每条**机器标注**必须携带：

```json
"provenance": {
  "model": "claude-opus-4-8",
  "prompt_version": "tier2_card.v1",
  "run_id": "batch_xxx 或 worker run id",
  "ts": "ISO8601",
  "tokens": {"in": 0, "out": 0}
}
```

### 8.3 tier1_screen.jsonl（每行一条）

```json
{
  "id": "vllm@8cc26acd8b77",
  "repo": "vllm", "sha": "8cc26acd8b77...", "date": "2026-01-18",
  "perf_relevance": 0.85,
  "kind_guess": "optimization",
  "symptom_guess": "latency",
  "needs_deep_read": true,
  "diff_truncated": false,
  "rationale": "一句话",
  "provenance": {...}
}
```

### 8.4 cards.jsonl（核心产出 1；pilot schema 的 v1 定版）

```json
{
  "case_id": "Megatron-LM@72171c0d531d",
  "repo": "Megatron-LM", "sha": "...", "date": "2026-06-16",
  "subject": "Fix memory leak with log_max_attention_logit (#4699) (#5067)",
  "source": "commit-stream | issue-stream | revert-stream | golden-stream | academic",
  "is_perf_related": true,
  "kind": "regression-fix | optimization | config-default-change | perf-infra-or-test | not-perf | unclear",
  "symptom": "throughput | latency | memory | gpu-util-or-bubble | hang | compile-or-startup-time | n/a",
  "mechanism": "自由文本：根因机制，如 'torch.maximum 累积统计量未 detach 导致 autograd graph 保留'",
  "magnitude_reported": "13.7x | 24% | null（引用来源）",
  "env_conditional": true,
  "manifest_conditions": {"parallelism": "TP=4", "dtype": "fp8", "hardware": "GB200", "shape": null},
  "inducing_commit_traceable": "direct | likely | hard | n/a",
  "inducing_ref": "sha / PR# / issue# / null",
  "static_detectability": "high | medium | low | n/a",
  "taxonomy_label": null,
  "issue_refs": ["vllm#26320"],
  "pr_ref": "#5067",
  "evidence": "1-3 句，必须引用 diff 行或 PR/issue 原话",
  "confidence": 0.9,
  "double_annotated": true,
  "provenance": {...}
}
```

（`taxonomy_label` Phase 0 留空，Phase 1 回填。字段完整定义与枚举值见 `schemas/card.v1.schema.json`。）

### 8.5 regression_pairs.jsonl（核心产出 2）

```json
{
  "pair_id": "vllm@e0327c9db123->179ae7da8456",
  "repo": "vllm",
  "inducing_sha": "...", "fix_sha": "...",
  "case_id": "vllm@179ae7da8456",
  "evidence_tier": "A | B | C",
  "evidence_source": "revert-body | user-bisect | commit-message | ci-report | agentic-szz-3vote",
  "szz_votes": {"agent_1": "sha_x", "agent_2": "sha_x", "agent_3": "sha_y"},
  "symptom": "...", "magnitude": "95->86 tps decode (-9.5%)",
  "manifest_conditions": {...},
  "static_detectability": "medium",
  "inducing_at_time_context": {"parent_sha": "...", "date": "..."},
  "human_audited": true,
  "provenance": {...}
}
```

### 8.6 audits/audit_log.jsonl（append-only）

```json
{
  "ts": "ISO8601", "auditor": "rui",
  "target_id": "case_id 或 pair_id",
  "field": "kind",
  "machine_value": "optimization", "human_value": "regression-fix",
  "verdict": "override | confirm",
  "note": "自由文本"
}
```

消费规则：读 card 时若 audit_log 中存在同 target+field 的 override，以人工值为准（提供一个 `scripts/apply_audits.py` 输出合并视图 `cases/cards_final.jsonl`）。

### 8.7 splits/protocol.md（核心产出 3）必须包含

1. **切分规则**：时间切分（2026-01-01 前 → train，之后 → dev/test 对半）+ 仓库留出视角（附加报告：train 排除 Megatron 时在 Megatron test 上的表现）
2. **防泄漏呈现规则**：评测时给检测器的输入 = inducing commit 时点的 `(diff, message, parent 树的仓库快照)`，不含任何该时点之后的信息（fix、issue、golden 变更）
3. **Memorization 检查**：对 test 集每个 pair，问裸 Opus"这个 commit 后来被修复了吗/有什么问题"，能背出答案的标 `memorized: true` 单独报告
4. **指标定义**：recall@每-PR-评论预算（预算=2）、per-kind/per-taxonomy recall、良性流误报率、静态天花板（detectability=low 的占比，作为诚实上限）
5. **Baseline 定义**：(a) Megatron 现行 strict-review prompt 原样跑 test 集；(b) 裸 Opus 4.8 泛型 review prompt。两者的结果表进 `reports/baseline_results.md`（此项在 Phase 2 执行，协议先写死）

### 8.8 每个 stage 的验收标准（Definition of Done）

| Stage | DoD |
|---|---|
| Tier 0 | 33,085 sha 全部有路由记录；桶分布进 report |
| Tier 1 校准 | 对 pilot 已知阳性 recall ≥95%；报告落盘 |
| Tier 1 全量 | 全部 `screen` 桶 sha 有判定；成本实测 vs 预算差异 <30% |
| Tier 2 | 全部 needs_deep_read 有 card；20% 双标完成；字段级一致率报告 |
| Tier 3 | 全部 regression-fix 走完回溯；pairs 按 tier 分布报告 |
| 子管线 S2–S5 | 各自 raw 文件 + 回填完成 |
| 负样本 | 三层负样本齐 + 分布报告 |
| 人工仲裁 | 全部 A/B pairs 过目 + 10% 审计 + kappa 报告 |
| 冻结 | `splits/FROZEN` 落盘；README/DATASHEET 完稿；v0.1 tag |

---

## 9. 预算与时间线（全 Opus 4.8：$5/$25 per MTok，Batch 五折）

| 项 | 量 | 通道 | 估算 |
|---|---|---|---|
| Tier 1 校准 ×2 轮 | ~1.3k commits | Batch | ~$30 |
| Tier 1 全量 | ~30k commits × ~8k in | Batch（$2.5/$12.5） | ~$700 |
| Tier 2 深读 | ~4k × ~50k tokens | 标准 + caching | $1,200–2,500 |
| Tier 2 双标 20% | ~800 | 标准 | ~$300–500 |
| Tier 3 回溯 | ~600 × 3 票 × 25k | 标准 | $300–600 |
| S2 issues | ~650 深读 | 标准 | $150–300 |
| S3/S4 子管线 | — | 标准 | ~$100 |
| 重跑/修 prompt 余量 | — | — | ~$500 |
| **合计** | | | **$3,300–5,200（帽 $6,000）** |

时间线：W1 = 全量 clone + Tier 0 + Tier 1 校准与全量 + S3/S4 子管线；W2 = Tier 2 + S2 + Tier 3；W3 = 负样本 + 人工仲裁 + 协议冻结 + v0.1 tag。Rui 的人工投入集中在 W2 末–W3（3–5 天）。

---

## 10. 风险与缓解

| 风险 | 缓解 |
|---|---|
| Tier 1 漏检不可恢复 | 校准闸门 recall ≥95% 先行；阈值宁松勿紧（误报由 Tier 2 消化） |
| SZZ 标签噪声（perf fix 远离引入点） | 3 票制 + regression/latent 显式二分 + 全部 pairs 人工过目 |
| 模型记忆污染评测 | memorization 检查（§8.7.3）+ 呈现时点隔离 |
| 环境混杂（container 升级/集群搬迁） | golden 流单独标 `env-confounded`，不进 pairs |
| 上游 license/issue 文本合规 | DATASHEET 记录来源与 license（四仓库均 Apache/BSD 系）；仓库先 private |
| 成本超支 | $4,500 预警线 + $6,000 硬帽；每 stage 实测入 report |

---

## 11. Phase 0 之后的路线图

> 每个 Phase 的输入都是上一 Phase 落盘在 MegaPerfBench 里的文件——这是 §8 输出规范存在的意义。

**Phase 1 — 分类学归纳与检测配方分派（~1 周）**
输入 `cases/cards_final.jsonl` → 自底向上聚类归纳两层分类学（目标 30–50 叶子类），每叶挂：真实案例列表、检测要点、`static_detectability` 分布 → 据此把每叶分派到三种检测配方之一：(a) 机械可判 → 静态规则生成；(b) 语义类 → 检索增强 review；(c) 环境相关 → 风险路由+定向 benchmark。回填 `taxonomy_label` 到 cards。产出：`taxonomy/taxonomy.yaml` v1。

**Phase 2 — Baseline 评测（~3–5 天，先于任何检测器开发）**
按 `splits/protocol.md` 在冻结 test 集上跑两个 baseline（Megatron 现行 strict-review prompt / 裸 Opus review），得到要打败的数字。产出：`reports/baseline_results.md`。**没有这一步，后面所有"更好"都无从主张。**

**Phase 3 — 检测器 v1（3–4 周，三条腿并行）**
- 3a 规则生成：对机械可判类，KNighter 式从历史 patch 合成静态检查器（perf 领域无人做过，本身可发表），规则库进 `rules/`
- 3b 检索增强 review：分类学+案例做成检索库，按 diff 触及文件/API 检索案例喂给 Opus review
- 3c 风险路由：对环境相关类输出"建议触发哪个 Megatron perf recipe"（gpt-perf / moe_perf 等），与现有 perf CI 联动

**Phase 4 — 对抗验证层与端到端评测（~2 周）**
发布前对抗验证（第二 agent 试图反驳每条 finding）+ 每 PR 评论预算；在 test 集上端到端跑 v1 vs baseline，报 recall@budget 与误报率。北极星指标是 action rate 的代理（评测集上）。

**Phase 5 — 影子运行与飞轮（持续）**
在 Megatron-LM 真实 PR 流上影子运行（只记录不评论），与 claude[bot] 现网评论对比；误报反馈进 hard negatives，新 bug 进案例库——数据集从此活起来。

**Phase 6 —（可并行）论文与 v1.1**
Dataset paper（MegaPerfBench）+ system paper（检测器）；可执行复现子集（30–50 例 Docker 化，$2–5k GPU 预算）作为 v1.1，不阻塞 v1。

---

*计划定稿于 2026-07-13。执行中的任何口径变更须更新本文档并在 commit message 中注明。*
