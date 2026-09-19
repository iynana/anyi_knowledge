## 总体架构

本项目采用 **Orchestrator + Worker Agent** 模式：由 `DietOrchestratorService` 统一驱动状态机，各 Worker Agent 只负责产出结构化结果，**不直接读写 SessionState**。

五个 Agent 的协作关系如下：
```
用户请求
    ↓
DietOrchestratorService（编排层 / 状态机）
    ↓
IntentAgent
    ↓
Java 意图矫正、槽位合并、澄清规则
    ├─ 信息不足 → ClarifyAgent -> 返回追问
    └─ 信息充足 → 数据库检索 + Java 排序
                       ├─ 普通推荐 → RecommendResponseAgent
                       └─ 多餐规划 → PlanResponseAgent

离线评估：
历史 Trace + 最终回复 → EvaluationJudgeAgent
```

| Agent                  | 模型           | 核心作用                  |
| ---------------------- | ------------ | --------------------- |
| IntentAgent            | `qwen-turbo` | 意图识别和槽位抽取             |
| ClarifyAgent           | `qwen-turbo` | 将缺失信息包装成自然追问          |
| RecommendResponseAgent | `qwen-max`   | 基于 Top3，生成单餐推荐理由和完整回复 |
| PlanResponseAgent      | `qwen-max`   | 生成多餐规划理由和完整回复         |
| EvaluationJudgeAgent   | `qwen-turbo` | 离线评估：解释质量、自然度（1-5 分）  |
分类/追问/评估用**轻量模型**降延迟和成本；最终推荐文案用**主模型**保质量。

核心设计原则：
- ==**LLM 负责理解与生成，Java 负责状态、路由、检索与合规**==，Java 规则层是护栏，确保 LLM 不会跑偏或说出危险的话。
- **Agent 无状态调用**：每次调用前 `agent.getMemory().clear()`，上下文由 Orchestrator 通过 prompt 注入
- **会话级 Agent 隔离**：`AgentFactory` 按 `sessionId + promptVersion` 缓存 AgentSet，避免多用户记忆串话


## 0. Agent 调用统一入口

所有 Agent 调用经 `AgentTraceService.callAgent()` 包装：
+ 同步 `agent.call(...).block()`  等待 LLM 返回；
+ 自动记录 `AGENT_CALL` 事件：agentName、modelName、input、output、latencyMs、token 用量
+ 异常时 markFailed 并向上抛出

各 Service 在调用前统一：
+ `agentFactory.get(sessionId).xxx()` 取实例
+ `agent.getMemory().clear()` 清记忆构造结构化
+ `user prompt`（含 `session` 上下文、槽位字典等）
+ `agentTraceService.callAgent(...)` 发起调用

### 1. IntentAgent：识别意图、提取条件

负责理解用户当前想做什么，输出：
- `intent`：用户意图
- `slots`：饮食条件
- `confidence`：识别置信度

可以识别的意图包括：
- `MEAL_RECOMMENDATION`：推荐吃什么
- `CLARIFY_NEEDED`：想吃东西但信息不足
- `MEAL_ADJUST`：换一批或调整推荐
- `MEAL_PLAN`：规划三餐或多餐
- `HEALTH_RISK`：涉及疾病治疗、极端节食等风险
- `OTHER`：与饮食推荐无关

```json
{
  "intent": "MEAL_RECOMMENDATION",
  "slots": {
    "mealTime": "午餐",
    "taste": "清淡",
    "convenience": "快速"
  },
  "confidence": 0.9
}
```

==只理解不回复==

### 2. ClarifyAgent：生成自然追问

当用户信息不足时，根据 Java 提供的 `missingSlots` 生成一句自然追问。

槽位字段：
- **mealTime**：餐次（早餐/早午餐/午餐/下午茶/晚餐/夜宵/加餐/三餐）  
- **mood**：心情（疲惫/烦躁/开心/焦虑/低落/平静/压力大/没胃口/想放松/想奖励自己）  
- **scene**：场景（工作/校园/家里/周末/加班/运动后/通勤/聚餐/独处/旅行/夜宵）  
- **healthGoal**：健康诉求（减脂/清淡/养胃/高蛋白/均衡/降火/低油/低盐/低糖/补能/增肌/控碳水/易消化/暖胃）  
- **cuisine**：菜系偏好（川菜/粤菜/湘菜/江浙菜/东北菜/鲁菜/闽南菜/云南菜/新疆菜/轻食/西餐/日料/韩餐/东南亚菜/火锅/烧烤/海鲜/素食/家常/小吃/粉面/粥汤/快餐/甜品）  
- **taste**：口味（清淡/辣/微辣/中辣/麻辣/甜/酸甜/咸鲜/鲜香/酱香/蒜香/番茄味/咖喱味/奶香/油香/烟火气）  
- **convenience**：便捷需求（快速/慢享/外带方便/堂食舒服/少排队/少餐具/一人食/多人共享/适合备餐/适合边走边吃）

例如缺少餐次时：

> 这顿主要是早餐、午餐还是晚餐？

**LLM 返回空时会使用 Java 固定追问模板兜底。**

### 3. RecommendResponseAgent：生成普通推荐回复

用于普通的单餐推荐。它接收到的是已经完成**数据库检索和 Java 排序的候选餐食**。

主要负责：
- 给每款候选餐食生成推荐理由。
- 将多个候选包装成自然的口语回复。
- 结合用户的心情、场景和健康诉求解释推荐原因。
- 输出 `recommendations` 和 `speechText`。

### 4. PlanResponseAgent：生成多餐规划回复

用于“三餐”“早中晚”“多餐安排”等规划请求。

在调用之前，`MealPlanService` 已经**按餐次完成检索和排序**。该 Agent 负责：
- 为每个餐次的餐食生成理由。
- 将早餐、午餐、晚餐组合成连贯方案。
- 某个餐次没有候选时如实说明。
- 输出 `mealPlans` 和 `speechText`。

它不能跨餐次移动候选，也不能编造 `mealId`。

### 5. EvaluationJudgeAgent：离线评价回答质量

这个 Agent 不参与用户在线对话，而是用于离线评估历史请求的回答质量。

主要评估：
- `explanationQuality`：推荐理由是否清楚、是否贴合用户需求
- `naturalness`：回复是否自然、连贯、符合中文交流习惯

一般输出 1～5 分及简短评价。它不负责重新推荐，也不参与规则准确率、安全性和幻觉检测；这些指标主要由 `EvaluationService` 根据 trace 和业务规则计算。