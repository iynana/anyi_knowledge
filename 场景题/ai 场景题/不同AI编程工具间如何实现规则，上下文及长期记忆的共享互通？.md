**统一事实源**：
```
.ai/
├── rules/
│   ├── coding.md
│   ├── architecture.md
│   ├── testing.md
│   ├── git.md
│   └── security.md
├── context/
│   ├── project.md
│   └── modules.md
├── memory/
│   ├── decisions.md
│   ├── issues.md
│   └── learned.md
└── skills/
```

**+**

**适配层**：
```
CLAUDE.md -> claude code
AGENTS.md ->codex
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

