## 1. 业务背景

餐食在 `meal_item` 表中不是扁平字段，而是 **7 维 JSON 数组标签**：
```
meal_time, mood, scene, health_goal, cuisine, taste, convenience
```

每道菜可以有多个标签（如「番茄鸡蛋面」同时标记午餐/晚餐/三餐、疲惫/低落、工作/校园…）。

用户查询也是 7 维 `SlotBundle`，需要找到「标签与用户偏好有交集」的餐食。传统 LIKE 或单字段匹配无法表达这种 **多标签、多维度、集合交集** 的检索需求。

## 2. 技术选型：JSON_OVERLAPS

MySQL 8.0 提供的 `JSON_OVERLAPS(doc, candidate)` 判断两个 JSON 文档是否存在交集。
对 JSON 数组而言，即 **两个数组是否有共同元素**。
本项目在 `MealMapper.xml#search` 中对 7 维逐一应用：
```json
AND (#{mealTimeJson} = '[]' OR JSON_OVERLAPS(meal_time, #{mealTimeJson}))
AND (#{moodJson} = '[]' OR JSON_OVERLAPS(mood, #{moodJson}))
AND (#{sceneJson} = '[]' OR JSON_OVERLAPS(scene, #{sceneJson}))
AND (#{healthGoalJson} = '[]' OR JSON_OVERLAPS(health_goal, #{healthGoalJson}))
AND (#{cuisineJson} = '[]' OR JSON_OVERLAPS(cuisine, #{cuisineJson}))
AND (#{tasteJson} = '[]' OR JSON_OVERLAPS(taste, #{tasteJson}))
AND (#{convenienceJson} = '[]' OR JSON_OVERLAPS(convenience, #{convenienceJson}))
```

## 3. 空维度短路设计

每个维度都有 `#{xxxJson} = '[]' OR JSON_OVERLAPS(...)` 的前置条件：
- 用户该维槽位 **为空** → 传 `'[]'` → 条件恒真，**不参与过滤**
- 用户该维槽位 **非空** → 必须与餐食对应 JSON 数组有交集

**意义**：用户只说「午餐 + 清淡」，不会因为没有 mood/scene 等标签而漏召回；未提及的维度是「不限」而非「必须为空」。
Java 侧由 `JsonService.toJsonArray(slots.xxx())` 将 `List<String>` 转为 JSON 数组字符串传入 MyBatis。

## 4. 数据源隔离

同一 SQL 通过 `<choose>` 分支隔离 PERSONAL / PUBLIC：

| sourceMode | WHERE 条件                                                 |
| ---------- | -------------------------------------------------------- |
| PERSONAL   | `source_type = 'PERSONAL' AND owner_user_id = #{userId}` |
| PUBLIC     | `source_type = 'PUBLIC' AND owner_user_id IS NULL`       

**禁止混查**：个人饭堂菜单与公共库不会在 SQL 层合并，由请求 `ChatRequest.sourceMode` 决定。

## 5. 召回策略：宽召回 + 限流

```
ORDER BY updated_at DESC
LIMIT #{limit}   -- SEARCH_LIMIT = 50
```

- **宽召回**：7 维 AND 交集，任一用户指定维有交集即可（其他维为空则跳过）
- **上限 50**：防止大库全表扫描式返回
- **初排仅按 updated_at**：精细排序交给 Java `MealRankService`

这是典型的 **DB 负责过滤 + Java 负责排序** 分工。