# orchestrator

专为 Claude Code 设计的任务编排插件 — 保持主 agent 上下文干净，同时将繁重工作委托给成本更低的模型。

[English](README.md)

![许可证: MIT](https://img.shields.io/badge/License-MIT-blue.svg) ![版本: 1.0.0](https://img.shields.io/badge/version-1.0.0-green.svg)

---

## 设计初衷

该插件围绕两个核心约束构建：

**1. 保持主 agent 上下文干净**

在复杂工程任务中，主 agent 的上下文窗口会被原始工具输出、文件内容和中间结果快速填满——这既影响推理质量，也推高了成本。orchestrator 让主 agent 只专注于决策。所有探索、实现和验证工作都卸载给子代理，这些子代理的原始输出不会直接出现在主 agent 的上下文中。

**2. 将执行工作交给成本更低的模型**

复杂任务不需要从头到尾都使用昂贵的模型。orchestrator 采用三层模型分级：

```
主 agent (opus)       — 任务分解、依赖管理、结果综合
  └─ Wrapper (haiku)  — 提示词整形、结果压缩
       └─ Codex (GPT) — 实际的文件读取、代码编写、测试运行
```

昂贵的推理集中在主 agent（少量高层决策），廉价的执行分散在 haiku wrapper 和 Codex——也就是工作量真正密集的地方。

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

## 使用方法

```
/orchestrate Add JWT authentication to this Express API
/orchestrate Refactor the database layer to use connection pooling
/orchestrate Find and fix all TypeScript type errors in src/
/orchestrate Write unit tests for the auth module
```

当 Claude 检测到任务横跨 3 个以上文件、包含多个实现步骤，或同时涉及探索 + 实现 + 测试时，也会自动触发编排模式。

---

## 工作原理

### 三层执行模型

每个子代理都是一个轻量级的 **haiku wrapper**——它的工作只有两件事：（1）将主 agent 的自然语言任务整形为结构化的 Codex 提示词；（2）将 Codex 的原始输出压缩为有界的结构化摘要，再返回给主 agent。

```
主 agent 发出：
  "探索认证模块的架构"

Wrapper 整形为：
  <task>探索 src/auth/ 架构</task>
  <research_mode>区分事实与推断</research_mode>
  <grounding_rules>每个结论必须引用具体文件路径</grounding_rules>
  <structured_output_contract>返回：关键文件、依赖关系、开放问题</structured_output_contract>

Codex 执行，Wrapper 压缩后返回：
  <summary>
  - AuthService 依赖 JwtStrategy 和 UserRepository
  - Token 刷新逻辑在 src/auth/refresh.guard.ts
  </summary>
  <files>
  - src/auth/auth.service.ts — 核心服务
  - src/auth/refresh.guard.ts — token 刷新逻辑
  </files>
  <open_questions>
  - 未在此范围内找到 token 过期配置
  </open_questions>
```

主 agent 只看到压缩后的摘要，而不是 Codex 的原始输出。

### 合约注册表

orchestrator 在项目根目录维护一个 **CONTRACTS.md** 文件，作为所有模块间公开接口的唯一事实来源。这使主 agent 在串联各 subagent 时无需读取代码文件。

信息沿单一方向流动：

```
codex-implementer 完成一个批次
  └─ 在输出中返回 <interface_contract> 块

codex-architect 读取产出文件
  └─ 更新 CONTRACTS.md
  └─ 向 orchestrator 返回 <contract_summary>

orchestrator 将 <contract_summary> 以 <upstream_contracts> 形式注入下一批次
  └─ 下游 implementer 从 prompt 中读取接口，无需访问磁盘
```

主 agent 仅在**关键错误无法从 agent 输出中诊断时**才读取代码文件。

### 五阶段协议

```
第 1 阶段 · 分析    Claude 分解任务，展示计划，等待用户确认
第 2 阶段 · 探索    codex-explorer + codex-researcher 并行运行
第 3 阶段 · 执行    多个实现批次，每批次后跟一个 codex-architect
第 4 阶段 · 验证    codex-tester + codex-reviewer 并行运行
第 5 阶段 · 综合    Claude 汇总摘要，解决关键问题，输出最终结果
```

第 3 阶段对多模块任务采用批次-架构师模式：

```
批次 A（并行 implementer）
  └─ 批次 A.arch：codex-architect → 更新 CONTRACTS.md，返回 contract_summary
批次 B（并行 implementer，通过 upstream_contracts 接收批次 A 的合约）
  └─ 批次 B.arch：codex-architect → 更新 CONTRACTS.md，返回 contract_summary
...
```

并非每个任务都需要完整的五个阶段：

| 任务类型 | 使用阶段 |
|---|---|
| 简单单模块修复 | 仅实现（无需架构师） |
| 多模块功能开发 | 完整流程，每批次间插入架构师步骤 |
| 纯研究任务 | 仅研究代理 |
| 代码重构 | 探索 → 重构 → 架构师 → 测试 → 审查 |

### 代理列表

每个代理分两层：**wrapper**（haiku）负责提示词整形和结果压缩，**executor**（Codex / GPT）执行实际工作。"Wrapper" 列为提示词整形所用的 Claude 模型；真正的执行始终发生在 Codex 内部。

| 代理 | Wrapper | Executor | 模式 | 职责 |
|---|---|---|---|---|
| `codex-explorer` | haiku | Codex (GPT) | 只读 | 代码库探索与架构映射 |
| `codex-implementer` | haiku | Codex (GPT) | 可写 | 功能实现与缺陷修复；返回 `<interface_contract>` |
| `codex-architect` | haiku | Codex (GPT) | 可写\* | 读取产出文件，更新 CONTRACTS.md，返回 `<contract_summary>` |
| `codex-tester` | haiku | Codex (GPT) | 可写 | 编写并运行测试 |
| `codex-reviewer` | haiku | Codex (GPT) | 只读 | 代码审查与质量分析 |
| `codex-researcher` | haiku | Codex (GPT) | 只读 | 网络研究与文档查阅 |
| `codex-refactorer` | haiku | Codex (GPT) | 可写 | 代码重构、重命名与整合 |
| `codex-server` | haiku | Codex (GPT) | — | 远程服务器命令执行、日志监控、状态检查 |

\* `codex-architect` 仅写入 CONTRACTS.md，不修改任何生产代码文件。

---

## 目录结构

```
orchestrator/
├── .claude-plugin/
│   └── plugin.json          # 插件清单
├── agents/
│   ├── codex-architect.md   # 可写 wrapper：合约提取，维护 CONTRACTS.md
│   ├── codex-explorer.md    # 只读 wrapper：探索与架构映射
│   ├── codex-implementer.md # 可写 wrapper：功能实现与缺陷修复
│   ├── codex-refactorer.md  # 可写 wrapper：代码重构与整合
│   ├── codex-researcher.md  # 只读 wrapper：网络研究与文档查阅
│   ├── codex-reviewer.md    # 只读 wrapper：代码审查与质量分析
│   ├── codex-server.md      # 服务器 wrapper：SSH 命令执行与监控
│   └── codex-tester.md      # 可写 wrapper：测试编写与执行
├── commands/
│   └── orchestrate.md       # /orchestrate 斜杠命令
└── skills/
    └── orchestrator/
        └── SKILL.md         # 自动触发技能
```

---

## 参与贡献

欢迎提交 Pull Request。对于重大变更，请先提 Issue 说明。

1. Fork 仓库：https://github.com/Joker7531/orchestrator
2. 创建功能分支：`git checkout -b feature/my-feature`
3. 提交更改，并向 `develop` 分支发起 Pull Request

---

## 许可证

[MIT](https://opensource.org/licenses/MIT) — Copyright (c) Joker7531
