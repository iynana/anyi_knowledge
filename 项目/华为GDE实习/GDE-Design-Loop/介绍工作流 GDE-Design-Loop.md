GDE-Design-Loop 本质上是基于 SDD，把“需求直接交给 AI 写代码”改造成“知识准备 → 目标澄清 → Spec 设计 → 多轮质询 → 高质量 Spec → AI Coding”的受控设计闭环。

**知识增强（Init）解决“Agent 不理解存量系统”的问题。** 单纯依赖代码目录或者有限上下文，Agent 很难知道模块之间的依赖以及修改影响。因此通过 GDE 知识工程和 omaka，引入 codewiki + codegraph，补充系统关系、模块依赖和代码修改影响，让后面的设计建立在真实系统上下文上。

**目标澄清（Def·Goal）解决“目标不明确就开始设计”的问题。** 原来可能只有一句自然语言需求，Agent 很容易自行补全需求，导致后面不断 Review 和返工。因此在 Design 前先通过需求分析和 BDD，把**目标、核心方案、业务场景和边界条件**明确下来，再把它作为后续 Spec 的约束。

**多轮设计质询（Challenge）解决“一次生成的 Spec 质量不可控”的问题。** Design 之后不是直接进入 Coding，而是引入设计与测试的红蓝对抗，从不同维度 Challenge 当前 Spec，发现遗漏后通过 spec-optimizer 修订，再重新审查。