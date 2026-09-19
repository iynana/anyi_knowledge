## 1. 全部 Trace 记录点与调用时机

当前代码共有 26 种 Trace 事件。每轮请求只记录公共事件和实际命中的业务分支事件，不会出现全部 26 种事件。

|序号|Trace 事件|Phase|调用时机|适用范围|
|---|---|---|---|---|
|1|`REQUEST_RECEIVED`|`HTTP`|加载初始 `SessionState`、开启 Trace 后，进入会话锁前|所有业务请求|
|2|`USER_MESSAGE_RECORDED`|`SESSION`|用户消息成功追加到 `diet_messages` 后|所有请求|
|3|`PERSONAL_LIBRARY_EMPTY`|`ROUTE`|`PERSONAL` 模式下检查到用户没有个人餐食时|个人库为空|
|4|`AGENT_CALL`|`AGENT`|每次通过 `AgentTraceService.callAgent()` 调用 LLM Agent 时|根据分支出现 0～多个|
|5|`INTENT_RECOGNIZED`|`INTENT`|`IntentAgentService.recognize()` 返回原始意图、槽位、置信度后|除个人空库外|
|6|`INTENT_REVISED`|`INTENT`|`IntentReviseService.revise()` 完成 Java 规则矫正后|除个人空库外|
|7|`ROUTE_SELECTED`|`ROUTE`|得到最终意图，即将进入对应业务处理方法前|除个人空库外|
|8|`SLOTS_MERGED`|`SLOT`|普通推荐分支合并历史槽位与本轮槽位后|推荐、澄清|
|9|`CLARIFY_DECISION`|`CLARIFY`|`ClarifyAgentService.decide()` 返回 `ASK/READY` 后|推荐、澄清|
|10|`ADJUST_CONTEXT_RESOLVED`|`ADJUST`|换一批分支完成槽位合并并取得历史推荐 ID 后|`MEAL_ADJUST`|
|11|`PLAN_CONTEXT_RESOLVED`|`PLAN`|多餐规划完成槽位合并和目标餐次解析后|`MEAL_PLAN`|
|12|`MEAL_SEARCHED`|`SEARCH`|`MealSearchService.search()` 完成数据库候选召回后|普通推荐、换一批|
|13|`MEAL_RANKED`|`RANK`|`MealRankService.rank()` 完成排除、打分、排序并返回 Top10 后|普通推荐、换一批|
|14|`NO_MEAL_MATCHED`|`RECOMMEND`|普通推荐的 `ranked` 为空，无法继续生成推荐时|普通推荐无结果|
|15|`RECOMMEND_RESULT_BUILT`|`RECOMMEND`|`recommendAndRespond()` 返回并取得结构化 `RecommendResult` 后|普通推荐、换一批|
|16|`RESPONSE_AGENT_RESULT`|`RESPONSE`|普通推荐取得 `ResponseResult` 后|普通推荐、换一批|
|17|`MEAL_PLAN_SEARCHED`|`PLAN`|多餐规划按目标餐次完成检索和排序后|`MEAL_PLAN`|
|18|`NO_MEAL_PLAN_MATCHED`|`PLAN`|多餐规划所有目标餐次都没有匹配结果时|多餐无结果|
|19|`PLAN_RESULT_BUILT`|`PLAN`|取得结构化多餐 `RecommendResult` 后|多餐有结果|
|20|`PLAN_RESPONSE_AGENT_RESULT`|`RESPONSE`|取得多餐规划 `ResponseResult` 后|多餐有结果|
|21|`NUTRITION_GUARD_CHECKED`|`GUARD`|推荐回复生成后，`RiskGuardService.check()` 检查完成时|普通推荐、换一批、多餐规划|
|22|`NUTRITION_GUARD_REWRITTEN`|`GUARD`|Guard 未通过，用保守提示替换原回复后|不合规回复|
|23|`COMPLIANCE_GUARD_REWRITTEN`|`GUARD`|Guard 通过、不需要替换回复时|合规回复|
|24|`RESPONSE_READY`|`CLARIFY` / `RESPONSE`|状态和助手消息保存完成，最终 `ChatResponse` 构建后|所有正常返回分支|
|25|`REQUEST_FINISHED`|`HTTP`|`handleTurn()` 正常返回，HTTP 方法返回响应前|正常请求|
|26|`REQUEST_FAILED`|`HTTP`|处理过程中出现未处理的 `RuntimeException` 时|失败请求|

## 2. Agent 调用时机

所有模型调用统一记录为 `AGENT_CALL`。

|Agent|调用时机|不调用的情况|
|---|---|---|
|`IntentAgent`|个人餐食库前置检查通过后，识别本轮意图与七维槽位|个人餐食库为空时提前返回|
|`ClarifyAgent`|普通推荐链路中，规则判定存在必要槽位缺失，需要生成自然追问时|`missingSlots` 为空、直接 `READY` 时|
|`RecommendResponseAgent`|普通推荐或换一批得到非空候选 Top3 后，生成理由和 `speechText` 时|候选为空时|
|`PlanResponseAgent`|多餐规划至少匹配到一道餐食后，生成多餐理由和回复时|所有餐次均无匹配时|
|`EvaluationJudgeAgent`|离线评估开启 LLM Judge 时|`includeLlmJudge=false` 时；通常不属于原请求 Trace|

每条 `AGENT_CALL` 记录：

agentName  
modelName  
inputPayload  
outputPayload  
latencyMs  
inputTokens  
outputTokens  
totalTokens  
errorMessage

## 3. 普通推荐路径

### 3.1 槽位足够

ROUTE_SELECTED（MEAL_RECOMMENDATION / CLARIFY_NEEDED）  
→ SLOTS_MERGED  
→ CLARIFY_DECISION（READY）  
→ MEAL_SEARCHED  
→ MEAL_RANKED  
→ AGENT_CALL（RecommendResponseAgent）  
→ RECOMMEND_RESULT_BUILT  
→ RESPONSE_AGENT_RESULT  
→ NUTRITION_GUARD_CHECKED  
→ COMPLIANCE_GUARD_REWRITTEN  
   或 NUTRITION_GUARD_REWRITTEN  
→ RESPONSE_READY  
→ REQUEST_FINISHED

### 3.2 没有匹配餐食

MEAL_SEARCHED  
→ MEAL_RANKED  
→ NO_MEAL_MATCHED  
→ RESPONSE_READY  
→ REQUEST_FINISHED

此时不会调用 `RecommendResponseAgent`。

## 4. 澄清路径

ROUTE_SELECTED（MEAL_RECOMMENDATION / CLARIFY_NEEDED）  
→ SLOTS_MERGED  
→ AGENT_CALL（ClarifyAgent，需要追问时）  
→ CLARIFY_DECISION（ASK）  
→ 保存 CLARIFY 状态和助手追问  
→ RESPONSE_READY（phase=CLARIFY）  
→ REQUEST_FINISHED

是否需要澄清由 Java 规则决定，ClarifyAgent 只负责生成自然追问。ClarifyAgent 失败或返回空文本时，使用 Java 模板追问。

## 5. 换一批路径

ROUTE_SELECTED（MEAL_ADJUST）  
→ ADJUST_CONTEXT_RESOLVED  
→ MEAL_SEARCHED  
→ MEAL_RANKED  
→ AGENT_CALL（RecommendResponseAgent）  
→ RECOMMEND_RESULT_BUILT  
→ RESPONSE_AGENT_RESULT  
→ NUTRITION_GUARD_CHECKED  
→ Guard 结果事件  
→ RESPONSE_READY  
→ REQUEST_FINISHED

`ADJUST_CONTEXT_RESOLVED` 记录：

{  
  "mergedSlots": "历史槽位与本轮槽位的合并结果",  
  "excludeMealIds": \[12, 25, 31\]  
}

`excludeMealIds` 来自 `SessionState.lastRecommendations`，真正的过滤发生在 `MealRankService` 排序层。

如果没有历史推荐，`IntentReviseService` 会把 `MEAL_ADJUST` 修正为 `MEAL_RECOMMENDATION`，因此不会出现 `ADJUST_CONTEXT_RESOLVED`。

## 6. 多餐规划路径

### 6.1 至少一道餐食匹配

ROUTE_SELECTED（MEAL_PLAN）  
→ PLAN_CONTEXT_RESOLVED  
→ MEAL_PLAN_SEARCHED  
→ AGENT_CALL（PlanResponseAgent）  
→ PLAN_RESULT_BUILT  
→ PLAN_RESPONSE_AGENT_RESULT  
→ NUTRITION_GUARD_CHECKED  
→ Guard 结果事件  
→ RESPONSE_READY  
→ REQUEST_FINISHED

### 6.2 所有餐次都未匹配

PLAN_CONTEXT_RESOLVED  
→ MEAL_PLAN_SEARCHED  
→ NO_MEAL_PLAN_MATCHED  
→ RESPONSE_READY  
→ REQUEST_FINISHED

此时不会调用 `PlanResponseAgent`。

## 7. 健康风险路径

REQUEST_RECEIVED  
→ USER_MESSAGE_RECORDED  
→ AGENT_CALL（IntentAgent）  
→ INTENT_RECOGNIZED  
→ INTENT_REVISED（HEALTH_RISK）  
→ ROUTE_SELECTED  
→ RESPONSE_READY  
→ REQUEST_FINISHED

健康风险分支直接返回 `RiskGuardService.conservativeMessage()`，不检索餐食、不调用推荐 Agent，也不记录推荐后置 Guard 事件。

## 8. 闲聊路径

REQUEST_RECEIVED  
→ USER_MESSAGE_RECORDED  
→ AGENT_CALL（IntentAgent）  
→ INTENT_RECOGNIZED  
→ INTENT_REVISED（OTHER）  
→ ROUTE_SELECTED  
→ RESPONSE_READY  
→ REQUEST_FINISHED

该分支返回固定饮食引导文案，不调用回复生成 Agent。

## 9. 个人餐食库为空路径

REQUEST_RECEIVED  
→ USER_MESSAGE_RECORDED  
→ PERSONAL_LIBRARY_EMPTY  
→ RESPONSE_READY  
→ REQUEST_FINISHED

该检查位于意图识别之前，因此不会出现 `AGENT_CALL（IntentAgent）`、`INTENT_RECOGNIZED`、`INTENT_REVISED` 和 `ROUTE_SELECTED`。

## 10. Guard 调用时机

推荐类回复生成后调用 `RiskGuardService.check()`：

RESPONSE_AGENT_RESULT / PLAN_RESPONSE_AGENT_RESULT  
→ NUTRITION_GUARD_CHECKED

检查通过：

→ COMPLIANCE_GUARD_REWRITTEN

检查未通过：

→ NUTRITION_GUARD_REWRITTEN

`COMPLIANCE_GUARD_REWRITTEN` 是历史命名，实际语义是“合规检查通过，未改写原回复”。

## 11. 失败和降级的 Trace 表现

### 11.1 Agent 调用失败

AGENT_CALL.errorMessage != null  
→ TraceScope.markFailed()  
→ 业务服务使用关键词或模板继续执行  
→ 仍可能出现 RESPONSE_READY 和 REQUEST_FINISHED

用户可能获得正常响应，但 Trace 状态为 `FAILED`，离线评估计入 fallback。

### 11.2 推荐 JSON 解析失败

AGENT_CALL 成功  
→ parseOutput() 异常  
→ catch (Exception ignored)  
→ templateOptions() + templateSpeech() 兜底

当前代码没有为该场景记录单独的 Fallback 事件，Trace 可能仍显示为 `SUCCESS`。建议后续增加：

RECOMMEND_RESPONSE_FALLBACK

并记录 `JSON_PARSE_FAILED`、`EMPTY_OUTPUT` 等原因。