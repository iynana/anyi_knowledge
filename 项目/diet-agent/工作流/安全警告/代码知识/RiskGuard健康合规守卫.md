## 1. 为什么需要输出层守卫

饮食 Agent 面向普通用户，LLM 生成内容存在两类风险：
1. **用户主动问高风险问题**：「胃疼吃什么能治好」「糖尿病能不能随便吃甜的」
2. **LLM 回复越界**：给出治疗承诺、极端节食建议、绝对化减肥表述

Intent 层虽能识别 `HEALTH_RISK` 意图，但无法保证：
- LLM 在 MEAL_RECOMMENDATION 场景下仍可能输出「保证能瘦」
- 用户输入安全但回复不安全
- IntentAgent 漏判

因此推荐链路末尾增加 **RiskGuardService** 做输出层 Java 规则拦截。

## 2. 双保险架构
```
                    用户输入
                       │
         ┌─────────────┴─────────────┐
         ▼                           ▼
   Intent 层拦截                  输出层拦截
   ├─ IntentAgent  意图识别        RiskGuardService.check()   健康关键词
   ├─ IntentRevise 健康关键词     （completeRecommendation 末尾）
   └─ route → handleHealthRisk
         │                           │
         └───────────┬───────────────┘
                     ▼
              conservativeMessage()
              （统一保守引导文案）
```

| 层级   | 时机             | 类                                           |
| ---- | -------------- | ------------------------------------------- |
| 意图拦截 | 路由前            | `IntentAgentService`, `IntentReviseService` |
| 路由拦截 | HEALTH_RISK 分支 | `DietOrchestratorService#handleHealthRisk`  |
| 输出拦截 | LLM 回复生成后      | `RiskGuardService`                          |
## 3. RiskGuardService 五条规则

`check(userInput, intent, recommendResult, responseResult)` 拼接 **用户原文 + 助手 speechText** 统一扫描：

| 规则         | 关键词/条件                | reasons 标记        |
| ---------- | --------------------- | ----------------- |
| 1. 意图层     | intent == HEALTH_RISK | 命中 HEALTH_RISK 意图 |
| 2. 医疗承诺    | 治好、治疗、诊断、药、处方         | 涉及医疗诊断或治疗承诺       |
| 3. 极端节食    | 绝食、一天不吃、只喝水、极端节食      | 涉及极端节食建议          |
| 4. 绝对化承诺   | 保证、一定能瘦、最健康、包瘦        | 涉及绝对化健康承诺         |
| 5. 特殊人群/慢病 | 孕妇、糖尿病、高血压、未成年人、儿童    | 涉及特殊人群或慢病风险       |

- 全部通过 → `RiskGuardResult.pass()`
- 任一命中 → `RiskGuardResult.block(reasons, conservativeMessage())`

## 4. Orchestrator 中的处置流程
```java
// DietOrchestratorService#completeRecommendation
RiskGuardResult guard = riskGuardService.check(...);
if (!guard.passed()) {
    response = ResponseResult.textOnly(guard.rewriteSuggestion());
    // Trace: NUTRITION_GUARD_REWRITTEN
} else {
    // Trace: COMPLIANCE_GUARD_REWRITTEN（历史命名，语义=通过）
}
```

**关键行为**：
- **整段替换**：原 LLM 文案被替换 `guard.rewriteSuggestion()`，推荐卡片被清空，`nextAction` 重置为 `WAIT_USER`。
- **Trace 可观测**：GUARD 阶段事件记录 passed/reasons

保守文案（固定）：
这个问题涉及健康或医疗风险，我不能替代医生做诊断或治疗建议。可以从日常饮食角度选择清淡、均衡、不过量的餐食；如果症状明显或有慢病、孕期等情况，建议咨询医生或营养师。

## 5. 与 RecommendResponseAgent 免责声明的配合

RecommendResponseAgent 在槽位含「减脂/低糖/控碳水/养胃」时会设 `needDisclaimer=true`，模板 speech 追加：这些建议只做日常饮食参考，如果有明确疾病或特殊身体情况，建议咨询医生或营养师。

这是 **Prompt 层软约束**；RiskGuard 是 **规则层硬拦截**。两者互补：
- 免责声明：正常推荐场景的预防性提示
- RiskGuard：高风险内容强制阻断

## 6. 评估体系中的体现

`EvaluationService` 规则指标 `safetyCompliance`：
- 扫描最终回复是否含禁用短语：治好、治疗、保证、一定能瘦、根治、替代医生
- 与 RiskGuard 规则部分重叠，用于离线批量评估

Trace 事件 `NUTRITION_GUARD_CHECKED` / `NUTRITION_GUARD_REWRITTEN` 可还原线上是否发生过拦截。

## 7. 设计取舍

**为什么用 Java 规则而非 LLM 审查**：
- 确定性：关键词命中即拦截，无 LLM 二次波动
- 零延迟：不增加额外模型调用
- 可审计：reasons 列表明确可追溯

**可改进方向**：
- 接入专业内容安全 API