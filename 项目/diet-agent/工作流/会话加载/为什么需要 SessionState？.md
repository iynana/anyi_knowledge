`SessionState` 是系统的结构化跨轮记忆，保存历史槽位、历史推荐 ID和会话阶段。历史槽位用于理解用户的省略表达，历史推荐 ID用于“换一批”时排除重复结果，会话阶段用于记录当前处于澄清、推荐还是规划状态。相比直接把完整聊天记录交给模型，结构化状态更稳定、可控，也更节省 Token。

```java
SessionState {
    sessionId, userId,
    phase,              // START | CLARIFY | RECOMMEND | PLAN  用于交互追踪
    sourceMode,         // PERSONAL | PUBLIC
    currentIntent,
    slots,              // SlotBundle 7 维累积槽位
    lastRecommendations // 已推荐 mealId 列表（累积去重）
}
```

**记忆语义**：
- `slots`：跨轮**累积**的用户偏好（餐次、场景、口味等），本轮 IntentAgent 只输出增量，由 `SlotMergeService` 与历史合并
- `lastRecommendations`：本会话已展示过的餐食 ID，供「换一批」排除重复
- `phase` / `currentIntent`：告诉 Orchestrator 当前会话卡在哪个阶段，辅助路由与规则矫正
- `sourceMode`：持久化在 `slots._meta`，保证多轮始终在同一数据源（PERSONAL / PUBLIC）内检索
