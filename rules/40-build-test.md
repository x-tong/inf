# Build and Test Rules

本规则固定本仓库的基础开发套路。除非具体任务另有说明，C++ 项目默认使用 xmake 构建、测试和管理依赖。

## 默认工具链

- macOS 本地开发优先。
- C++ 默认使用 C++23。
- 构建系统默认使用 xmake。
- 依赖优先通过 xmake package 管理。
- 不在仓库中提交依赖目录、构建目录、缓存或二进制产物。

## xmake 约定

根目录应提供 `xmake.lua`。

推荐 target 命名：

- `<module>`：核心库。
- `<module>_tests`：测试二进制。
- `<module>_cli` 或 `<tool>_cli`：命令行验证工具。
- `<module>_bench`：benchmark 二进制。

依赖声明应集中在 `xmake.lua`：

```lua
add_requires("gtest")
add_requires("nlohmann_json")
add_requires("yaml-cpp")
add_requires("cli11")
```

target 内只绑定实际需要的依赖：

```lua
add_packages("nlohmann_json", "yaml-cpp")
```

## 常用命令

首次配置 debug 构建：

```bash
xmake f -m debug
```

构建：

```bash
xmake
```

运行测试：

```bash
xmake run <target>_tests
```

运行 CLI smoke test：

```bash
xmake run <target>_cli --help
```

清理构建输出：

```bash
xmake clean
```

## 开发流程

每次实现代码变更后，至少执行：

```bash
xmake
xmake run <changed_module>_tests
```

如果修改 runtime 路径、配置解析、trace 输出或 CLI 行为，还应执行对应 CLI smoke test。

如果修改 benchmark 或性能相关代码，应保留：

- benchmark 命令。
- 运行环境。
- raw result 路径。
- 对比 baseline。

## 测试规则

- 新增核心模块时，应新增最小单元测试。
- 修改 router、budget、fallback、trace 等 runtime 行为时，应补充行为测试。
- 测试用例应尽量 deterministic，避免依赖真实网络、真实 LLM API 或不稳定时间。
- 真实 backend 测试默认不作为本地必跑测试，除非任务明确要求。

## 输出规则

最终说明中应写明实际执行过的验证命令。

如果验证没有运行，应说明原因，例如：

- 当前只修改文档。
- 依赖尚未安装。
- 目标尚未实现。
- 需要外部服务或密钥。

不得把未运行的测试描述为已通过。

## Git 忽略规则

以下内容不得提交：

- `build/`
- `.xmake/`
- xmake 缓存目录。
- 编译产物。
- benchmark 临时输出。
- secrets、token、环境变量文件。
