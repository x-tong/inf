# inf

面向 AI infra 转型的个人工程项目集。

当前主项目：

- [SLO-aware Context Execution Runtime](./docs/rag-aware-request-execution-runtime-project.md)

## 项目定位

这个仓库用于沉淀 AI infra 方向的项目规划、设计文档、实验记录、实现代码和 benchmark 报告。默认采用文档优先、可验证优先、AI 协作友好的开发方式。

## 仓库结构

```text
.
├── AGENTS.md                  # AI coding agent 协作入口规则
├── rules/                     # 人和 AI 都应遵守的工程规则
├── docs/                      # 项目文档、设计、路线图、ADR
├── .github/                   # GitHub issue / PR 模板
├── .gitmessage                # 中文 commit message 模板
└── README.md
```

## 开发原则

- 文档先行：重要功能先写设计，再进入实现。
- 小步提交：每次 commit 聚焦一个清晰变更。
- 中文提交：commit message 使用中文，说明“做了什么”和“为什么”。
- 可验证：功能、性能、benchmark 结论都要有复现路径。
- 尊重上下文：修改前先读相关文档和规则，不做无关重构。

## 快速开始

1. 阅读 [AGENTS.md](./AGENTS.md)。
2. 阅读 [rules/README.md](./rules/README.md)。
3. 从 [docs/README.md](./docs/README.md) 进入项目文档。

