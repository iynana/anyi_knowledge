MultiAgent 本质是将一个复杂任务拆分给多个具备不同职责的 Agent 协同完成。
单个 Agent 在完成复杂任务时，任务的所有步骤都由同一个Agent完成，可能会有几种情况
- 多个任务交由一个agent处理上下文容易混乱
- 不同的步骤的专业领域不同，单个Agent不够专、精，多个agent可以进行微调适配
- 任务中存在多个并行子任务，但单个Agent只能串行执行

**Orchestrator-Subagent 模式**：一个 **orchestrator** 负责全局规划和任务分发，多个 **subagent** 并行或串行执行具体子任务，最终由 orchestrator 汇总输出

**Peer-to-Peer 模式**：agent 之间平等对话、相互审查，适合需要辩论或验证的场景