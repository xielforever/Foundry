---
alwaysApply: false
globs: "*.go,*.proto,*.yaml,*.yml,*.json,configs/agents/*.yaml"
description: 编写或修改 Go、Protobuf、配置文件时的命名规范（技术栈选型结果：Go 1.22+、gRPC + Protobuf），包含 Agent ID 和错误码命名规范
---

## 文件命名

| 类型 | 规范 | 示例 |
|------|------|------|
| Go 源文件 | `snake_case.go` | `artifact_validator.go` |
| 测试文件 | `snake_case_test.go` | `artifact_validator_test.go` |
| Protobuf 文件 | `snake_case.proto` | `agent_executor.proto` |
| 配置文件 | `kebab-case.yaml` | `harness-pipeline.yaml` |
| 目录 | 全小写无分隔符（Go 社区约定） | `agentexecutor/`、`registry/` |

## Go 代码命名

| 类型 | 规范 | 示例 |
|------|------|------|
| 包（package） | 全小写无下划线，简短有意义 | `package agent`, `package registry` |
| 导出类型/接口 | `PascalCase` | `type Executor interface{}` |
| 导出函数/方法 | `PascalCase` | `func ExecuteTask() error` |
| 未导出类型/函数 | `camelCase` | `type executorImpl struct{}` |
| 常量 | `PascalCase`（导出）或 `camelCase`（未导出） | `MaxRetryCount` / `defaultTimeout` |
| 变量 | `camelCase` | `agentConfig`, `taskID` |
| 接口命名 | `-er` 后缀（单方法接口） | `Executor`, `Validator`, `Logger` |
| Getter | 不加 `Get` 前缀 | `func (a *Agent) Name() string` |
| 缩写 | 全大写或全小写，保持一致 | `HTTPClient`, `grpcServer`, `ID`, `URL` |

## Protobuf 命名

| 类型 | 规范 | 示例 |
|------|------|------|
| 包路径 | 小写 + 版本号：`project/version` | `package foundry.v1` |
| Message | `PascalCase` | `message TaskRequest {}` |
| Service | `PascalCase` + `Service` 后缀 | `service AgentExecutorService {}` |
| RPC 方法 | `PascalCase` | `rpc ExecuteTask(TaskRequest) returns (TaskResponse)` |
| 字段 | `snake_case` | `string task_id = 1` |
| 枚举类型 | `PascalCase` | `enum AgentType {}` |
| 枚举值 | `UPPER_SNAKE_CASE` | `AGENT_TYPE_LOCAL_CLI = 0` |
| go_package | 项目模块路径 + 包路径 | `option go_package = "github.com/foundry/foundry/gen/foundry/v1"` |

## Go 代码组织

| 类型 | 规范 | 说明 |
|------|------|------|
| `cmd/` 入口 | 每个子目录一个 `main.go` | `cmd/foundry/main.go`、`cmd/foundry-agent/main.go` |
| `internal/` 包 | 按领域拆分目录，每目录一个主包 | `internal/agent/`、`internal/registry/` |
| `proto/` 定义 | 按服务/领域拆分 `.proto` 文件 | `proto/foundry/v1/executor.proto` |
| `gen/` 生成代码 | 与 `proto/` 结构镜像，禁止手动编辑 | `gen/foundry/v1/` |

## Agent ID 命名规范

Agent ID 是 Agent 实例的唯一标识，遵循以下格式：

```
{type_prefix}-{name}-{version}
```

| Agent 类型 | type_prefix | 示例 |
|-----------|-------------|------|
| 本地 AI CLI | `local-ai-cli` | `local-ai-cli-codex-v1` |
| 远程 API | `remote-api` | `remote-api-openai-gpt4-v1` |
| 传统 CLI | `traditional-cli` | `traditional-cli-gitleaks-v2` |
| 人类 Gate | `human-gate` | `human-gate-default-v1` |

**规则**：

| 规则 | 说明 |
|------|------|
| type_prefix 必须与 agent_type 对应 | `local-ai-cli` 对应 `AGENT_TYPE_LOCAL_AI_CLI` |
| name 使用 kebab-case | 如 `codex`、`openai-gpt4` |
| version 使用 `v` + 数字 | 如 `v1`、`v2` |
| 全局唯一 | 同一 agent_id 不能重复注册 |
| 配置文件名与 agent_id 一致 | `local-ai-cli-codex-v1.yaml` |

> Agent ID 命名规范详见 [agent_registry_and_discovery.md](../../docs/design/agent_registry_and_discovery.md)。

## 校验错误码命名规范

| 规则 | 说明 | 示例 |
|------|------|------|
| 格式 | `UPPER_SNAKE_CASE` | `ARTIFACT_TYPE_MISMATCH` |
| 前缀按领域 | TASK_ / ARTIFACT_ / AGENT_ / REGISTRY_ | `ARTIFACT_TYPE_MISMATCH`、`AGENT_ID_CONFLICT` |
| 严重级别后缀 | 不加后缀（Error）/ `_WARNING`（Warning） | `ARTIFACT_COUNT_EXCEEDS_MAX`（Warning 级别不加后缀，在文档中标注级别） |
| 不可复用 | 已定义的错误码不可重新定义或更改含义 | — |

> 技术栈选型依据见 [tech_stack_and_architecture.md](../../docs/design/tech_stack_and_architecture.md)。
