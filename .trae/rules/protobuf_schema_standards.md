---
alwaysApply: false
globs: "*.proto,schemas/**/*.json,proto/**/*"
description: 编写或修改 Protobuf 定义和 JSON Schema 时的结构与规范要求
---

## Proto 文件组织

| 规则 | 说明 |
|------|------|
| proto 文件必须放在 `proto/foundry/v1/` 下 | 包路径为 `foundry.v1`，与 `gen/foundry/v1/` 镜像 |
| 共享枚举必须提取到 `common.proto` | 避免跨文件循环依赖；当前共享枚举：AgentType、ArtifactType、ContentFormat、ExecutionStatus（Task 9 同步）；待注册枚举：AgentStatus（registry.proto）、InterventionStatus/RollbackGranularity/OnFailureStrategy（intervention.proto）、AuditEventType（audit.proto） |
| 每个 proto 文件必须有 `option go_package` | 格式：`option go_package = "github.com/foundry/foundry/gen/foundry/v1"` |
| 跨文件引用使用 `import` | 如 `import "foundry/v1/common.proto"`，禁止内联重复定义 |

## 枚举规范

| 规则 | 说明 |
|------|------|
| 所有枚举必须以 `UNSPECIFIED = 0` 占位 | Protobuf 规范要求零值为默认值，UNSPECIFIED 不作为合法业务值 |
| JSON Schema 中排除 UNSPECIFIED | JSON Schema 校验时 UNSPECIFIED 不是合法值，必须显式指定有效枚举 |
| 新增枚举值追加到末尾 | Protobuf 字段编号不可复用，新增值使用下一个可用编号 |
| 自定义类型使用 99 号段 | 如 `ARTIFACT_TYPE_CUSTOM = 99`，预留扩展空间 |

## 枚举注册规则

新增枚举必须遵循以下规则：

| 规则 | 说明 |
|------|------|
| 跨 2 个以上 proto 文件使用的枚举必须放入 `common.proto` | 如 ExecutionStatus 被 executor.proto/audit.proto/intervention.proto 共用 |
| 仅在单个 proto 文件内使用的枚举定义在该文件中 | 如 AgentStatus 仅在 registry.proto 中使用 |
| 新增枚举值必须追加到末尾 | Protobuf 字段编号不可复用 |
| 枚举注册需在 PROGRESS.md Open Questions 中追踪 | 确保跨文档同步不遗漏 |

**当前枚举分布**：

| 枚举 | 所在文件 | 使用方 |
|------|---------|--------|
| AgentType | common.proto | task.proto, artifact.proto, executor.proto, registry.proto, audit.proto |
| ArtifactType | common.proto | task.proto, artifact.proto, executor.proto, registry.proto |
| ContentFormat | common.proto | artifact.proto |
| ExecutionStatus | common.proto（Task 9 同步） | executor.proto, audit.proto, intervention.proto |
| AgentStatus | registry.proto | registry.proto |
| InterventionStatus | intervention.proto（待创建） | intervention.proto |
| RollbackGranularity | intervention.proto（待创建） | intervention.proto |
| OnFailureStrategy | intervention.proto（待创建） | intervention.proto |
| AuditEventType | audit.proto | audit.proto |

## 生成代码规范

| 规则 | 说明 |
|------|------|
| `gen/` 目录由 buf 自动生成，禁止手动编辑 | 任何修改必须在 `proto/` 中进行，然后重新生成 |
| `gen/` 结构与 `proto/` 镜像 | `proto/foundry/v1/` → `gen/foundry/v1/` |
| 使用 buf 而非 protoc 构建 | buf 是项目标准 Protobuf 构建工具，配置见 `proto/buf.yaml` |

## JSON Schema 规范

| 规则 | 说明 |
|------|------|
| Schema 文件放在 `schemas/` 目录 | 通用 Schema：`schemas/task.schema.json`、`schemas/artifact.schema.json` |
| Artifact 专属 Schema 放在 `schemas/artifacts/` | 如 `schemas/artifacts/code_review_report.schema.json` |
| JSON Schema 与 Protobuf 必须保持同步 | 新增/修改 Protobuf 字段时，对应 JSON Schema 必须同步更新 |
| 枚举在 JSON Schema 中排除 UNSPECIFIED | JSON Schema 的 enum 列表不包含 `*_UNSPECIFIED` 值 |
| 互斥约束在 JSON Schema 中用 `oneOf` 表达 | 如 ArtifactContent 的 data/file_path 二选一 |

## 构建与校验

| 操作 | 命令 | 说明 |
|------|------|------|
| Proto 构建 | `buf generate` | 从 proto/ 生成 Go gRPC 代码到 gen/ |
| Proto 校验 | `buf lint` | 检查 proto 文件是否符合规范 |
| Schema 校验 | `gojsonschema` | 使用 JSON Schema 校验 Task/Artifact 数据 |

> 技术栈选型依据见 [tech_stack_and_architecture.md](../../docs/design/tech_stack_and_architecture.md)。
> 数据模型定义见 [task_artifact_data_model.md](../../docs/design/task_artifact_data_model.md)。
