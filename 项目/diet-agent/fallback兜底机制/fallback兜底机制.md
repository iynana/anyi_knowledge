## 全链路 Fallback 地图

```
IntentAgent ──→ ClarifyAgent ──→ RecommendResponseAgent ──→ EvaluationJudge
     │                 │                      │                      │
     ▼                 ▼                      ▼                      ▼
关键词规则      模板追问文案          模板理由+speechText         返回 null
confidence=0.2  ClarifyRuleService    templateOptions()         不影响规则分
```

## IntentAgent Fallback

`Agent`调用异常 或者 `JSON` 解析失败，会根据关键词触发 `fallback`
1. 空输入 → CLARIFY_NEEDED
2. 健康风险词 → HEALTH_RISK
3. 换一批/清淡点 → MEAL_ADJUST
4. 三餐/一周 → MEAL_PLAN
5. 你是谁 → OTHER
6. 吃什么/推荐 → MEAL_RECOMMENDATION
7. 默认 → CLARIFY_NEEDED

## ClarifyAgent

`ClarifyRuleService` 判定 `missingSlots `非空 → 调 `ClarifyAgent` 生成自然语言追问。

当 `LLM` 异常或者 `LLM` 返回空文本的时候，触发降级
+ `missingSlots` 含 `mealTime`：「这顿主要是早餐、午餐还是晚餐？」
+ `missingSlots` 含 `healthGoal`：「这顿更想清淡点、顶饱点，还是按口味来？」
+ 空

## RecommendResponseAgent Fallback

`recommendAndRespond()` catch Exception或者部分 fallback，触发降级。

**抛异常**：
```json
RecommendResult recommend = new RecommendResult(templateOptions(topMeals, slots), needDisclaimer);
ResponseResult response = new ResponseResult(templateSpeech(recommend), toDisplayBlocks(recommend), "WAIT_USER");
```
`recommend` 是推荐理由，`response` 是返回响应。

**部分 fallback**：
`parseOptions` 中 `LLM` 漏写 reason → 单条 `recommend` 补全；`speechText` 为空 → `response` 代替。

## EvaluationJudge Fallback

Judge 失败时：
- `llmJudgeScore = null`
- 总分按 **实际存在权重归一**（规则 60% + 反馈 30% 仍可用）
- 不阻塞整批评估任务