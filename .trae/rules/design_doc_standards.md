---
alwaysApply: false
description: 编写设计文档或评审设计文档时使用此规则
---

## 设计文档结构

每个设计文档必须包含以下基础章节：

1. **概述** (Overview) - 文档目的、范围、读者
2. **设计动机** (Motivation) - 为什么需要这个设计
3. **详细设计** (Design) - 核心设计内容
4. **接口定义** (Interfaces) - 明确的 API / 数据接口
5. **操作规范** (Operations) - 操作流程、状态转换、触发条件（如适用）
6. **约束与限制** (Constraints) - 当前设计的边界
7. **待决问题** (Open Questions) - 未解决的问题

不同设计文档可根据对应 AC 的验收要求灵活调整章节。例如：
- 回滚机制文档（AC-6.1）必须包含"回滚粒度/触发条件/回滚后状态/操作规范"
- 失败处理文档（AC-6）必须包含"失败状态机设计"

## 设计文档存放位置

- 设计文档位于 `docs/design/` 目录
- spec/tasks/checklist 位于 `.trae/specs/foundry-v1/` 目录
- 架构图位于 `docs/architecture.md`
- 场景模板位于 `docs/pipeline_scenarios.md`

## 跨文档同步规则

当设计文档引入影响其他文档的变更时，必须执行跨文档同步。

### 需要同步的变更类型

| 变更类型 | 示例 | 受影响文档 |
|---------|------|-----------|
| 新增枚举值 | Task 3 新增 `ARTIFACT_TYPE_EXECUTION_RECORD` | Task 2 的 ArtifactType 枚举表、Protobuf 定义、JSON Schema |
| 新增校验错误码 | Task 3 新增 `ARTIFACT_COUNT_EXCEEDS_MAX` | Task 2 的校验错误码表 |
| 新增 Protobuf 字段 | Task 3 新增 `expected_artifact_counts` | Task 2 的 task.proto 定义、Go 结构体、JSON Schema |
| 新增核心概念 | Task 3 新增 Capabilities | README/CLAUDE/project_rules 的术语表和概念命名 |

### 同步执行要求

1. **声明**：在设计文档中用 `> **跨文档同步说明**` 显式声明需要同步的变更
2. **执行**：同步更新受影响文档的对应章节，并在修订历史中记录
3. **验证**：在 PROGRESS.md 变更记录中注明同步完成
4. **版本联动**：被同步文档的版本号必须递增（至少 minor 版本）

### 同步检查清单

完成设计文档后，检查以下维度是否需要同步：

- [ ] 枚举值变更是否同步到枚举定义源文档？
- [ ] Protobuf 字段变更是否同步到 proto 定义和 JSON Schema？
- [ ] 校验错误码变更是否同步到错误码定义源文档？
- [ ] 新增核心概念是否同步到 README/CLAUDE/project_rules？
- [ ] 目录结构变更是否同步到 README/CLAUDE 的项目结构章节？
