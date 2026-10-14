

## 代码仓目录结构

Pyrhon项目目录结构如下。

```shell
DigitalMate/
├── digitalmate/                         # 核心Python源码文件
│   ├── component_xxx                    # 组件xxx（基础公共组件、功能组件）
│
├── recipes/                             # 可直接运行的实验配方
│
├── examples/                            # 此目录存放的东西xxx
│
├── tests/                               # 尽量镜像 digitalmate/
│   ├── component_xxx                    # tests/{component}/，与vllm_omni/{component}/保持镜像关系
│   ├── e2e/                             # 端到端功能用例
│   ├── dfx/                             # dfx用例：性能、可靠性、稳定性用例
|       ├── perf
|       ├── reliability
|       ├── stability
│
├── docs/                                # 项目文档源文件，由MkDocs构建
│   ├── getting_started/
│   ├── architecture/
│   ├── design/
│   ├── user_guide/
│   ├── developer_guide/
│   ├── api/
│
├── scripts/                             # 存放标准化项目脚本：不同系统包构建脚本
│   ├── build_pkg_harmonyos.sh
│   ├── build_pkg_ios.sh
│   ├── build_pkg_androids.sh
│   ├── build_pkg_windows.sh
│   ├── build_pkg_macos.sh
│
├── tools/                               # 开发、维护、测试工具集合。该目录通常服务于开发者、维护者和 CI。       
│   ├── xxx
│
├── requirements/                        # 不同系统的依赖
│   ├── harmonyos/
│   ├── ios/
│   ├── android/
│   ├── windows/
│   ├── macos/
|
├── .buildkite/                          # Buildkite持续集成配置CI
|
├── .github/                             # GitHub 仓库自动化和协作配置
|
├── .agents/                             # 面向代码Agent的技能说明，主要内容位于.agents/skills/
|
├── .claude/                             # 为Claude类代码Agent提供项目级技能和工作流程说明
|
├── .codex/                              # 为Codex类代码Agent提供项目级技能和工作流程说明
|
├── .editorconfig
├── .gitignore                           # 指定不应提交到Git的文件
├── .pre-commit-config.yaml              # 提交前自动检查配置
├── .markdownlint.yaml                   # Markdown格式规范。用于文档检查及Pre-commit使用
├── AGENTS.md                            # AI Agent编码项目工作说明书：告诉Agent如何理解项目、开发环境搭建、用例执行、编码风格、文件修改边界等
├── CONTRIBUTING.md                      # 贡献指南：如何参与贡献、开发文档入口、Issue和PR基本要求
├── LICENSE                              # 项目许可证
├── README.md                            # 项目首页和总览文档：项目定位、核心能力、架构、安装和快速开始链接、文档、社区和论文入口、许可证信息
└── pyproject.toml                       # Python 项目的核心元数据和开发工具配置文件
```

## CI

### 静态检查

可在本地IDE运行，在门禁时自动运行。

| 检查范围        | 推荐工具           |
| --------------- | ------------------ |
| Python 代码质量 | Ruff Check         |
| Python 代码格式 | Ruff Format        |
| Python 类型检查 | mypy               |
| 通用文件格式    | `pre-commit-hooks` |
| 拼写            | Typos              |
| Markdown        | Markdownlint CLI2  |
| Shell 脚本      | ShellCheck         |
| GitHub Actions  | Actionlint         |
| 统一执行入口    | Pre-commit         |

.pre-commit-config.yaml

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: check-yaml
      - id: check-json
      - id: check-toml
      - id: debug-statements
      - id: end-of-file-fixer
      - id: mixed-line-ending
        args: ["--fix=lf"]
      - id: trailing-whitespace
        args: ["--markdown-linebreak-ext=md"]

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.10
    hooks:
      - id: ruff-check
        args: ["--fix"]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.1
    hooks:
      - id: mypy
        additional_dependencies:
          - types-PyYAML
          - types-requests

  - repo: https://github.com/crate-ci/typos
    rev: v1.49.0
    hooks:
      - id: typos

  - repo: https://github.com/rhysd/actionlint
    rev: v1.7.7
    hooks:
      - id: actionlint
        files: ^\.github/workflows/.*\.ya?ml$

  - repo: https://github.com/DavidAnson/markdownlint-cli2
    rev: v0.23.2
    hooks:
      - id: markdownlint-cli2
        args: ["--fix"]
        files: \.md$

  - repo: local
    hooks:
      - id: shellcheck
        name: ShellCheck
        entry: shellcheck
        language: system
        types: [shell]
```

pyproject.toml

```toml
[tool.ruff]
line-length = 120
target-version = "py310"

[tool.ruff.lint]
select = [
    "E",   # pycodestyle错误
    "W",   # pycodestyle警告
    "F",   # Pyflakes
    "I",   # Import排序
    "N",   # 命名规范
    "UP",  # Python语法升级
]
ignore = ["E203"]

[tool.mypy]
python_version = "3.10"
ignore_missing_imports = true
check_untyped_defs = true
warn_unused_ignores = true
warn_redundant_casts = true

[tool.typos.default]
extend-ignore-identifiers-re = [
    "项目专用缩写",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
```



### 用例分层

| 阶段   | 阶段名称              | 执行时间点/触发条件                                          | 用例覆盖重点                                                 | 执行环境                                                     | 测试目录                                                     | 建议时长                                  | 评率                                | 失败处理                                                     |
| ------ | --------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ----------------------------------------- | ----------------------------------- | ------------------------------------------------------------ |
| **G1** | 提交与PR基础门禁      | 本地 Commit 前执行快速检查；PR 创建、更新或重新打开时执行完整基础检查 | UT。代码规范、类型安全、敏感信息、可编译性、核心逻辑和确定性 Agent 行为 | 本地环境及 CI CPU Runner；Mock 服务；Scripted Model；不依赖真实模型和真机 | tests/{component}/…，与 DigitalMate/{component}/ 保持镜像关系 | 5～12分钟                                 | 每次PR提交                          | CI失败则PR标红，不允许进入Ready                              |
| **G2** | PR Ready防线          | PR 从 Draft 转为 Ready，或添加 `ready-for-review` 标签时触发 | UT&基础E2E。关键业务链路、基础E2E、模拟器交互、安全和隐私Smoke | CI Runner、HarmonyOS模拟器、Mock/Sandbox服务、固定或轻量模型 | tests/e2e/feature/...<br />tests/e2e/basic/...               | 15～30分钟                                | 带有Ready标签的PR（同时运行L1）     | 阻止批准和合入；必须修复或明确豁免                           |
| **G3** | 集成与合入防线        | 合入`main`后继续执行真实资源深度验证                         | 关键E2E功能、性能。关键集成、真实服务、真实模型、真机Smoke、质量和性能Smoke | CI Runner、模拟器、Sandbox、受限真实模型、至少一台真机或设备云 | tests/e2e/feature_expand/...                                 | 15～30分钟                                | PR 合并后（同时运行 L1 与 L2 测试） | 合入后失败则主分支标红、暂停Release；严重问题Revert或关闭功能 |
| **G4** | Nightly全量防线       | 每日低峰定时执行；重大架构修改后可手动触发                   | E2E。全功能、Agent质量、性能、稳定性、可靠性、异常恢复和兼容性 | 真机矩阵、模拟器、Sandbox、真实模型、故障注入环境            | tests/dfx/perf/                                              | 2～4小时                                  | 每日夜间运行                        | 生成报警和趋势报告；阻止产生Release Candidate                |
| **G5** | Release Candidate防线 | 创建Release Tag、RC分支或准备应用市场发布时触发              | E2E。最终发布产物、安装升级、签名、安全、隐私、设备兼容和发布验收 | 最终签名环境、Release配置、真机矩阵、预发布后端              | tests/dfx/stabity/...<br />tests/dfx/reliabity/...           | 4～8小时                                  | 版本发布前数日运行                  | 阻止发布；P0/P1、安全、升级和签名问题不得豁免                |
| **G6** | 发布后灰度防线        | 发布后持续执行，按内部用户、1%、5%、20%、全量逐级扩量        | 真实用户环境中的稳定性、任务质量、成本、隐私和回滚能力       | 生产灰度环境、脱敏遥测、Feature Flag、多模型和多Tool Provider |                                                              | 持续监控；每个灰度阶段建议观察数小时至1天 | 版本发布后灰度                      | 自动停止扩量；关闭问题功能；切换备用模型或服务；必要时回滚   |





## 看板

CI看板

项目生态健康度看板



## 参考链接

看板参考

https://hsliuustc0106.github.io/vllm-omni-kanban/

https://nanzhaogang.github.io/community-ecosystem/#/contributors



用例分级参考

https://github.com/vllm-project/vllm-omni/blob/main/docs/contributing/ci/test_system_overview.md

https://github.com/openJiuwen-ai/sciencediscovery/pull/259







