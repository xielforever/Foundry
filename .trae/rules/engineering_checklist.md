---
alwaysApply: false
description: 设计或实现新功能、编写设计方案时使用此规则，用于验证是否符合工程哲学
---

## Flow First 验证

> 对应 checklist.md: Flow First 验证项 [TR-3.4, TR-7.4]

设计或实现任何功能时，必须逐项确认：

- 这个行为附着在哪个 Pipeline/Stage/Step？必须明确回答
- Agent 是否拥有流程控制权？必须为"否"
- 流程是否能独立于 Agent 运行和测试？必须为"是"

## Deterministic Over Smart 验证

> 对应 checklist.md: Deterministic Over Smart 验证项 [TR-1.1]

- 给定相同输入，输出是否确定？必须为"是"
- 是否依赖了模型"聪明程度"？必须为"否"
- 失败模式是否有明确定义？必须为"是"

## Artifact Over Conversation 验证

> 对应 checklist.md: Artifact Over Conversation 验证项 [TR-2.4]

- Agent 输出是否是结构化的？必须为"是"
- 是否定义了 Artifact 的类型和 Schema？必须为"是"
- 是否存在无结构的文本输出？必须为"否"

## Agent Is Replaceable 验证

> 对应 checklist.md: Agent Is Replaceable 验证项 [TR-3.4, TR-4.3]

- 是否可以通过配置替换 Agent 的实现？必须为"是"
- 是否依赖了特定模型的特性？必须为"否"
- 是否支持动态禁用某个 Agent？必须为"是"
