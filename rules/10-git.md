# Git Rules

## Commit Message

commit message 必须使用中文。

推荐格式：

```text
<动词><对象><目的>
```

示例：

```text
初始化 vibecoding 仓库框架
补充 AI infra 项目路线图
修正 runtime benchmark 指标说明
```

避免：

```text
update
fix
wip
misc
```

## Commit Scope

- 每个 commit 聚焦一个主题。
- 不把无关文档、代码、格式化混在同一个 commit。
- 提交前运行 `git status --short` 检查范围。
- 有测试或校验命令时，在最终说明中写明。

## Branch

- 默认主分支使用 `main`。
- 个人实验分支可使用：

```text
exp/<topic>
doc/<topic>
feature/<topic>
```

## Remote

- GitHub 仓库默认使用 private，除非明确决定开源。
- 推送前确认没有密钥、token、私有数据、大文件或本地缓存。

