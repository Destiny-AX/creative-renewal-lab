# 02 AI 节点与 Prompt 说明

## 0. 设计总则

本项目采用“**规则形成事实信号和确定性约束，LLM 输出主变量排序、归因假设与候选，人工核验业务背景并决定是否继续**”的分工。AI 排序只代表实验优先级，不视为因果证明。

所有节点必须遵守：

- 不补写输入中不存在的投放数据、品牌事实、法规条款或实验结论。
- 事实字段必须携带数据源和时间窗口；缺失时输出 `unknown`，不得猜测。
- 归因只输出假设及证据，不把相关性写成因果。
- 变体只能改变 `allowed_variable` 指定的变量，固定项逐项校验。
- 合规预审必须返回命中的原句、规则依据、风险等级和建议；不是最终审核结论。
- 复盘必须把 `facts` 与 `hypotheses` 分开，放量仍需人工确认。
- 当前 Web 页面没有调用这些 Prompt；现有 Dify YML 是串行文本 Demo。本文件的 JSON 结构是面向下一版的产品化规格。

---

## 1. 素材疲劳归因

### 输入

- `material_id`、素材文本/视觉标签。
- 指标窗口：基准期和观察期 CTR、CVR、消耗、频次、ROI、样本量。
- 规则层计算结果：衰减率、异常标记、数据质量状态。
- 品牌/品类背景、历史相似素材和已知外部事件（如促销结束）。

### 输出

- 疲劳状态与置信度。
- 证据列表、备选解释和主变量排序。
- 缺失信息、风险提示和是否需要业务背景核验。

### Prompt 目标

把多指标信号转化为可读的归因假设，帮助优化师选择下一轮单变量实验，而不是预测真实效果。

### 约束条件

1. 只引用输入数据；每条证据标记字段名。
2. 至少给出一个竞争性解释，例如受众变化、预算/出价变化或落地页问题。
3. 数据质量为 `insufficient` 时不得输出高置信结论。
4. 只能从 `hook/visual/selling_order/cta/compliance/no_action` 中选择一个主变量。

### 结构化输出

```json
{
  "material_id": "A03",
  "status": "high_fatigue",
  "evidence": [{"metric": "ctr", "observation": "1.82% -> 1.13%", "source": "mock.recent_7d"}],
  "hypotheses": [{"cause": "hook_repetition", "confidence": 0.72, "why": "...", "counter_evidence": "..."}],
  "recommended_variable": "hook",
  "missing_information": ["placement breakdown"],
  "requires_business_review": true
}
```

### 示例 Prompt

```text
SYSTEM
你是广告素材诊断助手。只使用 INPUT 中的事实，不预测 CTR/CVR，不补写数据。
将事实与归因假设分开；给出至少一个竞争性解释；只能推荐一个主变量。
若样本量或时间窗口不足，status=insufficient_data。
严格输出 JSON Schema。

USER
INPUT={{material_metrics_json}}
RULE_SIGNALS={{rule_output_json}}
BUSINESS_CONTEXT={{campaign_brief_json}}
```

### 示例结果（基于模拟数据）

```json
{
  "material_id": "A03",
  "status": "high_fatigue",
  "evidence": [
    {"metric": "ctr", "observation": "1.82% -> 1.13%, relative decline 37.9%", "source": "SIM-GL-A03-v1"},
    {"metric": "frequency", "observation": "3.8", "source": "SIM-GL-A03-v1"}
  ],
  "hypotheses": [
    {"cause": "hook_repetition", "confidence": 0.72, "why": "评论重复信号与点击衰减同时出现", "counter_evidence": "尚无版位和人群拆分，不能排除流量结构变化"}
  ],
  "recommended_variable": "hook",
  "missing_information": ["版位拆分", "人群结构变化", "同期出价变化"],
  "requires_business_review": true
}
```

### 异常情况与兜底

- 指标窗口不一致：拒绝比较，返回 `invalid_window`。
- 样本过少：退回“继续观察”，不触发续命。
- 指标冲突：展示冲突，不强行归因；建议按人群/版位拆分。
- LLM 输出非法 JSON：一次低温重试；仍失败则展示规则层结果。

### 人工继续决策

**必须。** 规则层先形成事实信号，AI 输出主变量排序和归因假设；优化师核验业务事件、流量结构、预算、受众和版位变化，决定是否采用该实验方向并进入变体生成，但不重新计算 AI 排序，也不把排序视为因果证明。

---

## 2. 单变量变体生成

### 输入

- 对照组完整素材结构。
- `allowed_variable` 与 `locked_fields`。
- 品牌调性、禁用词、字符长度和平台格式要求。
- 归因节点结论与人工继续决策记录。

### 输出

- 3 个候选变体。
- 每个候选的差异说明、固定项校验和生成风险。

### Prompt 目标

围绕一个主变量生成可比较候选，避免“一次性大改”导致实验不可归因。

### 约束条件

1. `allowed_variable=hook` 时，只能修改标题/入口钩子；图、卖点顺序、CTA、受众和实验窗口不变。
2. 不增加输入中没有的产品功效、折扣、背书或数据。
3. 对照组原文必须原样保留。
4. 输出前逐字段比较；发现固定项变化则 `constraint_pass=false`。

### 结构化输出

```json
{
  "control_id": "A03",
  "allowed_variable": "hook",
  "locked_fields": ["visual", "selling_points", "cta"],
  "variants": [{
    "id": "B01",
    "changed_value": "...",
    "unchanged_snapshot": {"cta": "查看修护方案"},
    "constraint_pass": true,
    "change_summary": "仅修改入口钩子"
  }],
  "requires_human_confirmation": true
}
```

### 示例 Prompt

```text
SYSTEM
你是单变量广告变体生成器。CONTROL 是唯一事实来源。
只允许修改 ALLOWED_VARIABLE；LOCKED_FIELDS 必须逐字/逐项保持。
不得添加新功效、价格、优惠、认证、用户证言或效果数据。
生成 3 个候选，并输出字段级 diff 与 constraint_pass。

USER
CONTROL={{control_json}}
ALLOWED_VARIABLE=hook
LOCKED_FIELDS=[visual,selling_points,cta,audience,window]
BRAND_RULES={{brand_rules_json}}
```

### 示例结果（节选）

```json
{
  "control_id": "A03",
  "allowed_variable": "hook",
  "variants": [
    {"id": "B01", "changed_value": "明早有会，今晚的脸别先投降", "constraint_pass": true, "change_summary": "仅改钩子"},
    {"id": "B02", "changed_value": "暗沉不一定只是没睡够", "constraint_pass": true, "change_summary": "仅改钩子"}
  ],
  "requires_human_confirmation": true
}
```

### 异常情况与兜底

- 固定项被改：自动丢弃该候选并重试；最多一次。
- 三个候选同质化：调高差异要求，但不放宽固定项。
- 品牌事实不足：输出可编辑占位，不补写卖点。
- 所有候选合规高风险：不进入实验，返回人工改写。

### 人工确认

**必须。** 创意/品牌人员选择候选并可修改；修改后再次运行差异校验和合规预审。

---

## 3. 广告合规预审

### 输入

- 变体文本、视觉描述、落地页关键声明。
- 平台、行业、地区、规则库版本与生效时间。
- 品牌已批准声明和证据材料索引（如有）。

### 输出

- 风险总览与逐句风险。
- 规则依据、风险等级、建议改写、证据缺口。
- 是否硬拦截、是否需法务/审核人员确认。

### Prompt 目标

在实验前筛出明显风险并给出可追溯依据，减少后置返工；不代替正式审核。

### 约束条件

1. 每个风险必须绑定原句和规则 ID/规则描述。
2. 无规则依据时标记 `uncertain`，不能给“违规”定论。
3. 高风险硬拦截；中低风险给建议并进入人工复核。
4. 改写不得改变产品事实，不得为了“合规”添加新承诺。

### 结构化输出

```json
{
  "rule_version": "demo-v1",
  "items": [{
    "variant_id": "B02",
    "source_text": "屏障在求救",
    "risk_level": "low",
    "rule_id": "EXPRESSION-ANTHROPOMORPHISM",
    "rationale": "拟人化表达可能引起功效误解",
    "suggested_rewrite": "肌肤状态在提醒你需要修护",
    "hard_block": false
  }],
  "overall_decision": "human_review",
  "disclaimer": "预审结果不替代平台或法务终审"
}
```

### 示例 Prompt

```text
SYSTEM
你是广告合规预审助手，不是最终审核机构。
只使用 RULES 判断；每条风险必须返回 source_text、rule_id 和 rationale。
没有依据时 risk_level=uncertain。高风险不得自动改写后放行。

USER
PLATFORM={{platform}}
RULES={{versioned_rules_json}}
VARIANTS={{variant_json}}
APPROVED_CLAIMS={{approved_claims_json}}
```

### 示例结果

```json
{
  "rule_version": "demo-v1",
  "items": [{"variant_id": "B02", "source_text": "屏障在求救", "risk_level": "low", "rule_id": "EXPRESSION-ANTHROPOMORPHISM", "rationale": "表达偏强，可能被理解为确定功效", "suggested_rewrite": "肌肤状态在提醒你需要修护", "hard_block": false}],
  "overall_decision": "human_review",
  "disclaimer": "模拟预审，不替代正式审核"
}
```

### 异常情况与兜底

- 规则库缺失/过期：停止自动结论，只做关键词提示。
- 模型漏出规则外风险：标记 `uncertain` 并进入人工池。
- 规则冲突：展示冲突规则与版本，不自动放行。
- 输出无法定位原句：判定不可用，要求模型重新引用。

### 人工确认

**必须。** 高风险由法务/审核人员处理；中低风险至少由运营/审核人员终审。

---

## 4. 实验结果复盘

### 输入

- 对照组与变体组真实回填数据、样本量、时间窗口、分流方式。
- CTR、CVR、CPA、消耗、频次、ROI 及护栏指标。
- 数据质量与显著性检验结果。
- 实验前假设、变化项和固定项审计记录。

### 输出

- `facts`：可由数据直接支持的陈述。
- `hypotheses`：可能原因和适用条件。
- `decision_options`：继续观察/扩大样本/放量候选/停止。
- `knowledge_candidate`：待人工确认的知识条目。

### Prompt 目标

将实验数据变成可审阅的决策材料，不跨越数据证据做因果或 ROI 承诺。

### 约束条件

1. 事实只能来自输入；显著性不足时不得写“胜出”。
2. ROI/CPA 结论必须同时说明归因窗口与成本口径。
3. 推测必须标记证据与反证。
4. 知识条目包含适用人群、平台、素材类型和失效条件。

### 结构化输出

```json
{
  "experiment_id": "EXP-A03-001",
  "facts": [{"statement": "B01 CTR 高于对照组", "metric": "ctr", "value": "+39.8%", "significance": "unknown"}],
  "hypotheses": [{"statement": "场景化钩子可能提升点击意愿", "confidence": "low", "caveat": "模拟数据，未做显著性检验"}],
  "decision_options": ["collect_more_data"],
  "knowledge_candidate": null,
  "requires_human_confirmation": true
}
```

### 示例 Prompt

```text
SYSTEM
你是广告实验复盘助手。严格区分 FACTS 与 HYPOTHESES。
没有显著性或数据质量结论时，不得使用“显著、胜出、导致、提升 ROI”等确定性词语。
不得生成输入中不存在的指标。知识条目必须经过人工确认。

USER
EXPERIMENT_PLAN={{plan_json}}
RESULTS={{results_json}}
DATA_QUALITY={{quality_json}}
STAT_TEST={{stat_test_json}}
```

### 示例结果（本项目模拟页面对应）

```json
{
  "experiment_id": "DEMO-A03",
  "facts": [{"statement": "模拟页面中 B01 CTR 为 1.58%，对照组为 1.13%", "metric": "ctr", "source": "mock_ui"}],
  "hypotheses": [{"statement": "场景化钩子可能更匹配通勤人群", "confidence": "unverified", "caveat": "不是实投数据"}],
  "decision_options": ["do_not_scale", "run_real_ab_test"],
  "knowledge_candidate": null,
  "requires_human_confirmation": true
}
```

### 异常情况与兜底

- 数据不完整/分流污染：返回 `invalid_experiment`，不生成结论。
- CTR 上升但 CVR/ROI 恶化：触发护栏，不建议放量。
- 无显著性结果：建议扩大样本或延长窗口。
- LLM 混写事实和推测：结构化校验失败，回退到规则生成的指标摘要。

### 人工确认

**必须。** 预算、放量、停投和知识写入均为高影响动作，不能由当前原型自动执行。

---

## 5. 节点版本与评测记录建议

每次运行保留：`prompt_version`、`model_version`、`rule_version`、输入哈希、结构化输出、人工修改、最终决策和失败原因。这样才能计算 Prompt 版本间的准确率、约束遵循率、人工修改率与采纳率。
