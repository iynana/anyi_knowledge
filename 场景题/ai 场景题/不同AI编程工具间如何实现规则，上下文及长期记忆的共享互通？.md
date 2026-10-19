**统一事实源**：
```
.ai/
├── rules/
│   ├── coding.md          编码风格
│   ├── architecture.md    系统架构
│   ├── testing.md         测试规则
│   ├── git.md             提交规范
│   └── security.md        安全规则
├── context/
│   ├── project.md         项目背景，架构
│   └── modules.md         模块职责，接口
├── memory/
│   ├── decisions.md       架构决策
│   ├── issues.md          问题风险
│   └── learned.md         踩坑经验
└── skills/
```

**适配层**：
```
CLAUDE.md -> claude code
AGENTS.md -> codex
.cursor/rules/* -> cursor
.github/copilot-instructions.md ->copilot
```

适配层不需要独立维护，只是充当入口索引：
比如 `CLAUDE.md` ：
```
本项目统一 AI 规则位于：

.ai/rules/
.ai/context/
.ai/memory/

执行任务前优先读取与当前任务相关的规则文件。
```

