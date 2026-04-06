# orchestrator

专为 Claude Code 设计的动态任务编排插件 — 将复杂请求分解为子任务，分派给 Codex 子代理，综合汇总结果。

[English](README.md)

![许可证: MIT](https://img.shields.io/badge/License-MIT-blue.svg) ![版本: 1.0.0](https://img.shields.io/badge/version-1.0.0-green.svg)

---

## 功能简介

orchestrator 插件让 Claude Code 能够以结构化的方式处理大型、多步骤的工程任务。当你调用 `/orchestrate` 时，Claude 会将你的请求分解为若干专注的子任务，并将它们分派给六个专业化的 Codex 子代理 — 每个代理针对不同角色进行了优化（探索、实现、测试、审查、研究、重构）。所有结果汇总后，Claude 会解决关键问题，并将简洁的摘要反馈给你。

---

## 环境要求

- 安装了 Codex 插件的 [Claude Code](https://claude.ai/code)
- 需要安装以下技能：
  - `codex:codex-cli-runtime`
  - `codex:gpt-5-4-prompting`

---

## 安装方式

```bash
claude plugin install https://github.com/Joker7531/orchestrator
```

---

## 包含内容

### 代理列表

| 代理 | 模式 | 描述 |
|---|---|---|
| `codex-explorer` | 只读 | 代码库探索与架构映射 |
| `codex-implementer` | 可写 | 代码实现、功能新增、缺陷修复 |
| `codex-tester` | 可写 | 编写并运行测试 |
| `codex-reviewer` | 只读 | 代码审查与质量分析 |
| `codex-researcher` | 只读 | 网络研究与文档查阅 |
| `codex-refactorer` | 可写 | 代码重构、重命名与整合 |

### 命令

| 命令 | 描述 |
|---|---|
| `/orchestrate <task>` | 入口命令 — 描述任意任务，Claude 将负责分解与分派 |

### 技能

| 文件 | 描述 |
|---|---|
| `skills/orchestrator/SKILL.md` | 自动触发技能 — 当任务涉及 3 个以上文件、多个实现步骤，或同时包含探索 + 实现 + 测试时自动激活 |

---

## 使用示例

**新增功能：**

```
/orchestrate Add JWT authentication to this Express API
```

**重构现有代码：**

```
/orchestrate Refactor the database layer to use connection pooling
```

**修复全局类型错误：**

```
/orchestrate Find and fix all TypeScript type errors in src/
```

**为模块编写测试：**

```
/orchestrate Write unit tests for the auth module
```

---

## 工作原理

orchestrator 遵循五阶段协议：

```
┌─────────────────────────────────────────────────────────────┐
│  第 1 阶段 · 分析                                           │
│  Claude 解析请求，分解为子任务，                            │
│  等待用户确认后继续执行。                                   │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  第 2 阶段 · 探索                                   ░░ ░░   │
│  codex-explorer + codex-researcher 并行运行，               │
│  完成代码库映射并收集背景上下文。                           │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  第 3 阶段 · 执行                                           │
│  codex-implementer + codex-refactorer 运行 —                │
│  相互独立时并行，存在依赖时顺序执行。                       │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  第 4 阶段 · 验证                                   ░░ ░░   │
│  codex-tester + codex-reviewer 并行运行，                   │
│  验证正确性与代码质量。                                     │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  第 5 阶段 · 综合                                           │
│  Claude 汇总所有结果，解决关键问题，                        │
│  输出简洁摘要并给出后续步骤建议。                           │
└─────────────────────────────────────────────────────────────┘
```

### 复杂度快捷路径

并非每个任务都需要完整的五个阶段。orchestrator 会自动选择合适的执行路径：

| 任务类型 | 使用阶段 |
|---|---|
| 简单缺陷修复 | 探索 → 实现 → 测试 |
| 纯研究任务 | 仅研究代理 |
| 大型功能开发 | 完整五阶段流程 |
| 代码重构 | 探索 → 重构 → 测试 → 审查 |

---

## 目录结构

```
orchestrator/
├── .claude-plugin/
│   └── plugin.json          # 插件清单（名称、版本、描述）
├── agents/
│   ├── codex-explorer.md    # 只读代理：代码库与架构映射
│   ├── codex-implementer.md # 可写代理：功能实现与缺陷修复
│   ├── codex-refactorer.md  # 可写代理：代码重构与整合
│   ├── codex-researcher.md  # 只读代理：网络研究与文档查阅
│   ├── codex-reviewer.md    # 只读代理：代码审查与质量分析
│   └── codex-tester.md      # 可写代理：测试编写与执行
├── commands/
│   └── orchestrate.md       # /orchestrate 斜杠命令定义
└── skills/
    └── orchestrator/
        └── SKILL.md         # 多步骤任务自动触发技能
```

---

## 参与贡献

欢迎提交 Pull Request。对于重大变更，请先提 Issue 说明你希望修改的内容。

1. Fork 仓库：https://github.com/Joker7531/orchestrator
2. 创建功能分支：`git checkout -b feature/my-feature`
3. 提交更改：`git commit -m 'Add my feature'`
4. 推送分支：`git push origin feature/my-feature`
5. 发起 Pull Request

---

## 许可证

[MIT](https://opensource.org/licenses/MIT) — Copyright (c) Joker7531
