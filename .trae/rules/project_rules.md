---
alwaysApply: true
description: Foundry v1 核心约束 - 工程哲学原则与绝对禁止事项，所有对话始终生效
---

## 工程哲学原则

Foundry 项目的一切设计和实现必须遵循以下四项原则，违反任何一项的方案均不可接受：

### Flow First

- 一切行为必须附着在明确的 Pipeline / Stage / Step 中
- Agent 禁止拥有流程控制权
- 流程必须能独立于 Agent 运行和测试

### Deterministic Over Smart

- 给定相同输入，输出必须确定
- 禁止依赖模型"聪明程度"的设计
- 失败模式必须有明确定义

### Artifact Over Conversation

- Agent 输出必须是结构化的
- 必须定义 Artifact 的类型和 Schema
- 禁止存在无结构的文本输出

### Agent Is Replaceable

- 必须能通过配置替换 Agent 的实现
- 禁止依赖特定模型的特性
- 必须支持动态禁用某个 Agent

## 绝对禁止事项

以下行为在 Foundry 项目中严格禁止，无例外：

| 禁止项 | 原因 |
|--------|------|
| 设计或实现自治 Agent | 违反 Flow First |
| 让 Agent 之间直接通信 | 违反 Artifact Over Conversation |
| Executor 返回流程控制指令（如 skip_next_step、retry_pipeline） | 违反 Flow First——流程控制权属于 Harness，Executor 只能返回 Artifact 和 ExecutionStatus |
| Agent 产出无结构文本而非结构化 Artifact | 违反 Artifact Over Conversation——Agent 唯一合法输出是带类型和 Schema 的 Artifact |
| 设计自我规划/自我反思的 Agent | 超出 v1 能力边界 |
| 自动规划研发流程 | 超出 v1 能力边界 |
| 端到端无人值守交付 | 超出 v1 能力边界 |
| Agent 之间智能协商 | 超出 v1 能力边界 |
| 硬编码特定模型依赖 | 违反 Agent Is Replaceable |
| 在代码中硬编码密钥/凭证 | 安全基线 |
| 跳过审计日志 | 违反可审计性要求 |
| 修改已评审通过的设计文档而不更新 spec.md | 破坏文档一致性 |
| 未经用户确认自动 Git 提交 | 用户控制权要求 |
