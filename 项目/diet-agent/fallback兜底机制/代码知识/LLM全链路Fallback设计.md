## 1. 设计原则

LLM 在生产环境不可避免出现：
- 超时 / 限流
- JSON 格式错误或被 markdown 包裹
- 返回空文本
- 枚举值 / mealId 幻觉

本项目策略：**主路径走 LLM，异常路径走 Java 规则/模板，保证 Orchestrator 状态机始终能推进并返回可用响应**。

Trace 中通过 `fallbackRate` / `fallbackScore` 监控 fallback 发生频率。

## 2. 全链路 Fallback 地图

```
IntentAgent ──→ ClarifyAgent ──→ RecommendResponseAgent ──→ EvaluationJudge
     │                 │                      │                      │
     ▼                 ▼                      ▼                      ▼
关键词规则      模板追问文案          模板理由+speechText         返回 null
confidence=0.2  ClarifyRuleService    templateOptions()         不影响规则分
```

## 3. IntentAgent Fallback
### 3.1 触发条件

`IntentAgentService.recognize()` 整体 try-catch：
- Agent 调用异常
- JSON 解析失败
- intent 枚举非法

### 3.2 降级行为
```java
private IntentResult fallback(String userInput) {
    return new IntentResult(
        fallbackIntent(userInput),
        SlotBundle.empty(),
        0.2   // 低置信度
    );
}
```

### 3.3 关键词规则 fallbackIntent()
优先级链（节选）：
1. 空输入 → CLARIFY_NEEDED
2. 健康风险词 → HEALTH_RISK
3. 换一批/清淡点 → MEAL_ADJUST
4. 三餐/一周 → MEAL_PLAN
5. 你是谁 → OTHER
6. 吃什么/推荐 → MEAL_RECOMMENDATION
7. 默认 → CLARIFY_NEEDED

### 3.4 与 IntentRevise 联动

confidence=0.2 的 MEAL_RECOMMENDATION 会被 IntentRevise 降级为 CLARIFY_NEEDED（阈值 0.4），避免低质量识别直接进入推荐。


## 4. ClarifyAgent Fallback
### 4.1 主路径

ClarifyRuleService 判定 missingSlots 非空 → 调 ClarifyAgent 生成自然语言追问。
### 4.2 Fallback 触发

- LLM 异常（catch Exception）
- LLM 返回空文本（question.isBlank()）

### 4.3 降级行为
```
clarifyRuleService.fallbackQuestion(missingSlots)
```
模板按 missingSlots 内容选择：

| missingSlots | 模板                    |
| ------------ | --------------------- |
| 含 mealTime   | 「这顿主要是早餐、午餐还是晚餐？」     |
| 含 healthGoal | 「这顿更想清淡点、顶饱点，还是按口味来？」 |
| 空            | 默认确认口味/健康目标/方便快捷      |

**关键**：ASK/READY **决策始终由 Java 规则做出**，LLM 只影响追问文案风格；fallback 不影响澄清逻辑正确性。


## 5. RecommendResponseAgent Fallback
### 5.1 触发条件
`recommendAndRespond()` catch Exception。

### 5.2 降级行为
```java
RecommendResult recommend = new RecommendResult(templateOptions(topMeals, slots), needDisclaimer);
ResponseResult response = new ResponseResult(templateSpeech(recommend), toDisplayBlocks(recommend), "WAIT_USER");
```
- **templateReason**：按 healthGoal / taste 拼「比较符合你提到的 XX」
- **templateSpeech**：「我优先给你推荐这几款：\n- 名称：理由」+ 可选免责声明 
**仍返回 Top3 餐食卡片**，用户感知是「回复文案变模板化」，而非系统报错。

### 5.3 部分 Fallback（非异常）
`parseOptions` 中 LLM 漏写 reason → 单条 `templateReason` 补全；speechText 为空 → `templateSpeech`。

## 6. EvaluationJudge Fallback
```java
catch (Exception error) {
    log.warn("Evaluation Judge failed: traceId={}", traceId, error);
    return null;
}
```
Judge 失败时：
- `llmJudgeScore = null`
- 总分按 **实际存在权重归一**（规则 60% + 反馈 30% 仍可用）
- 不阻塞整批评估任务

## 7. Trace 层 Fallback 标记

`EvaluationService.parseTrace()`：
```java
boolean fallbackUsed = "FAILED".equalsIgnoreCase(row.getStatus());
// 任意 event 含 errorMessage 或 REQUEST_FAILED → fallbackUsed = true
```

指标：
- `fallbackRate`：单条 0/1，区间平均 = fallback 比例
- `fallbackScore`：未 fallback = 1.0
Also：`AgentTraceService` 中 Agent 调用失败会 `markFailed`，整轮 trace status=FAILED。

## 8. Fallback 设计矩阵

| 环节        | 能否继续主链路 | 用户可见结果      | 数据完整性           |
| --------- | ------- | ----------- | --------------- |
| Intent    | 能       | 可能进澄清/固定分支  | slots 可能为空      |
| Clarify   | 能       | 模板追问        | missingSlots 正确 |
| Recommend | 能       | 模板文案 + 真实卡片 | mealId 来自 Rank  |
| Judge     | 能（评估侧）  | 无（后台）       | 规则分仍可用          |
