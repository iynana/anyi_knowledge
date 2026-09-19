每轮 Agent 调用前，Orchestrator 或各 `AgentService` 将上述两层记忆**显式序列化**进 user prompt，而非依赖 Agent 内部记忆自动累积。

| Agent                  | 注入的短期记忆字段                                              |
| ---------------------- | ------------------------------------------------------ |
| IntentAgent            | `recentHistory` + `currentSlots` + `slotOptions` + 当前句 |
| ClarifyAgent           | `mergedSlots` + `missingSlots` + 用户原话                  |
| RecommendResponseAgent | `mergedSlots` + `sourceMode` + top3 候选 + 用户原话          |

其中，`recentHistory` 是读取时通过 `recentConversationTurns(sessionId, userId, n)` 转为 `ConversationTurn`的：
```java
ConversationTurn {
    role,       // user | assistant
    intent,     // assistant 轮次携带，user 轮次可为 null
    summary,    // content 截断至 120 字符（省 token）
    timestamp
}
```
**记忆语义**：
- 完整原文永久留在 `diet_messages`（审计、评估、Trace 交叉定位）
- 注入 LLM 时只取最近 **n 条**（Orchestrator 固定传 `n=3`），且每条最多 **120 字**