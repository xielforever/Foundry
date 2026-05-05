# Foundry v1 - 技术栈选型与项目架构文档

| 属性 | 内容 |
|------|------|
| **文档标题** | Foundry v1 - 技术栈选型与项目架构文档 |
| **文档作者** | Foundry Team |
| **文档日期** | 2026-05-04 |
| **文档版本** | v1.2 |
| **文档描述** | Foundry v1 核心技术栈选型对比分析与项目目录架构设计 |

---

## 概述

本文档解决 Foundry v1 的两个基础决策：

1. **技术栈选型**：确定编程语言、框架、工具链，给出对比分析和决策理由
2. **项目架构**：确定目录结构规范和模块划分

本文档是所有后续设计文档的基础，技术栈选型直接影响数据模型定义、Executor 接口设计、Harness 集成方案等。

## 设计动机

Foundry v1 的技术栈选型必须满足以下约束：

- **Deterministic Over Smart**：优先稳定性和可预测性，避免追新
- **Agent Is Replaceable**：不依赖特定模型或语言的特性，支持多种 Agent 类型接入
- **Harness 集成**：需与 Harness 流程引擎松耦合集成，Harness 核心为 Java，Step SDK 为 Java
- **可审计、可回滚**：技术栈需支持结构化日志、状态持久化、版本化配置

---

## 候选技术栈对比分析

### 评估维度

| 维度 | 权重 | 说明 |
|------|------|------|
| 稳定性与成熟度 | 高 | Deterministic Over Smart 原则的核心体现 |
| Harness 集成友好度 | 高 | 直接影响 Task 7 集成方案的复杂度 |
| 类型安全与可维护性 | 高 | 结构化 Artifact 和 Task 模型需要强类型保障 |
| 并发与性能 | 中 | 多 Agent 并行执行场景 |
| 生态与工具链 | 中 | 开发效率和调试能力 |
| 学习曲线与团队可用性 | 中 | 项目可持续维护 |

### 候选方案

#### 方案 A：Go

| 维度 | 评分 | 分析 |
|------|------|------|
| 稳定性与成熟度 | ⭐⭐⭐⭐⭐ | Go 1.x 承诺向后兼容，标准库稳定，生产级项目广泛验证 |
| Harness 集成友好度 | ⭐⭐⭐ | 无法直接使用 Harness Step SDK（Java），需走 Plugin Step（容器化）路径；Drone CI（Harness 收购项目）用 Go 编写，生态有参考 |
| 类型安全与可维护性 | ⭐⭐⭐⭐ | 静态类型、编译检查、接口显式定义；泛型支持（1.18+）但不如 Java/TS 灵活 |
| 并发与性能 | ⭐⭐⭐⭐⭐ | goroutine + channel 原生并发模型，极低资源开销，适合高并发 Agent 调度 |
| 生态与工具链 | ⭐⭐⭐⭐ | 丰富的 CLI/基础设施生态（Kubernetes、Terraform 均用 Go），但 AI/ML 生态薄弱 |
| 学习曲线 | ⭐⭐⭐⭐ | 语法简洁，上手快，但 error handling 和泛型模式需要适应 |

**优势**：
- 编译为单一二进制，部署简单，无运行时依赖
- 原生并发模型天然适合多 Agent 并行调度
- CLI 工具生态丰富，与 DevOps 基础设施天然亲和
- Drone CI（Harness 旗下）用 Go 编写，容器化插件模型与 Go 高度匹配

**劣势**：
- 无法直接使用 Harness Java Step SDK，需走容器化插件路径
- 泛型和类型系统不如 Java/TS 丰富，复杂模型定义较冗长
- AI/ML 生态薄弱，远程大模型 API 集成需自行封装

#### 方案 B：Java (Kotlin)

| 维度 | 评分 | 分析 |
|------|------|------|
| 稳定性与成熟度 | ⭐⭐⭐⭐⭐ | JVM 生态 20+ 年验证，Spring 生态成熟，企业级首选 |
| Harness 集成友好度 | ⭐⭐⭐⭐⭐ | 与 Harness 同技术栈，可直接使用 Step SDK，原生 Delegate 集成 |
| 类型安全与可维护性 | ⭐⭐⭐⭐⭐ | 强类型、泛型、接口体系完善；Kotlin 更现代，空安全、协程 |
| 并发与性能 | ⭐⭐⭐⭐ | Virtual Thread（Java 21+）或 Kotlin 协程支持高并发，但资源开销高于 Go |
| 生态与工具链 | ⭐⭐⭐⭐⭐ | 最丰富的企业级生态，AI SDK（LangChain4j 等）可用 |
| 学习曲线 | ⭐⭐⭐ | JVM 生态复杂度高，Spring Boot 学习曲线较陡 |

**优势**：
- 与 Harness 同技术栈，Step SDK 原生集成，集成复杂度最低
- 类型系统最完善，适合定义复杂的 Task/Artifact 数据模型
- 企业级生态成熟，AI SDK 可用
- Kotlin 兼具 Java 生态和现代语言特性

**劣势**：
- JVM 启动慢、内存占用高，不适合轻量 CLI 工具场景
- 容器镜像较大，Agent 执行器打包部署不够轻量
- 过度依赖 Spring 生态可能违反 Agent Is Replaceable 原则
- 开发体验偏重，对"快速迭代"不够友好

#### 方案 C：TypeScript (Node.js)

| 维度 | 评分 | 分析 |
|------|------|------|
| 稳定性与成熟度 | ⭐⭐⭐⭐ | Node.js 生态成熟，但依赖管理（node_modules）和版本碎片化是已知问题 |
| Harness 集成友好度 | ⭐⭐⭐ | 与 Java SDK 不兼容，需走容器化插件或 HTTP Step 路径 |
| 类型安全与可维护性 | ⭐⭐⭐⭐ | TypeScript 类型系统灵活，interface/type 定义数据模型自然；但运行时无类型检查 |
| 并发与性能 | ⭐⭐⭐ | 单线程事件循环，CPU 密集型任务需 Worker Thread；异步模型成熟但回调/Promise 链复杂 |
| 生态与工具链 | ⭐⭐⭐⭐⭐ | AI SDK 生态最丰富（OpenAI SDK、LangChain.js 等），前后端统一 |
| 学习曲线 | ⭐⭐⭐⭐ | 前端开发者友好，但 Node.js 后端最佳实践需要额外学习 |

**优势**：
- AI SDK 生态最丰富，远程大模型 API 集成最便捷
- TypeScript 类型系统定义数据模型自然（interface/type/enum）
- 前后端统一，未来如需 Web 管理界面可复用类型定义
- JSON 原生支持，与 Artifact 序列化天然匹配

**劣势**：
- 运行时无类型检查，数据校验需额外库（zod/joi）
- 单线程模型不适合 CPU 密集型 Agent 执行
- node_modules 依赖管理复杂，生产部署需要额外处理
- 与 Harness Java SDK 不兼容

### 综合评分

| 维度 | 权重 | Go | Java/Kotlin | TypeScript |
|------|------|-----|-------------|------------|
| 稳定性与成熟度 | 高 | 5 | 5 | 4 |
| Harness 集成友好度 | 高 | 3 | 5 | 3 |
| 类型安全与可维护性 | 高 | 4 | 5 | 4 |
| 并发与性能 | 中 | 5 | 4 | 3 |
| 生态与工具链 | 中 | 4 | 5 | 5 |
| 学习曲线 | 中 | 4 | 3 | 4 |
| **加权总分** | — | **4.17** | **4.50** | **3.83** |

---

## 决策：Go

### 决策理由

选择 **Go** 作为 Foundry v1 的核心技术栈，理由如下：

#### 1. Deterministic Over Smart 原则

Go 的设计哲学与 Foundry 的工程哲学高度一致：

- **显式优于隐式**：Go 的错误处理、接口定义、并发模型都是显式的，没有"魔法"
- **编译时确定**：Go 编译为静态二进制，运行时行为可预测，无 JVM 的动态代理或 Spring 的自动装配
- **最小惊喜原则**：Go 的语言特性少而精，代码行为可预测

#### 2. Agent Is Replaceable 原则

选择容器化插件模型（而非 Java Step SDK 原生集成）更符合 Agent Is Replaceable：

- **语言无关**：容器化插件模型允许 Agent 用任何语言实现，不绑定 Java
- **隔离性好**：每个 Agent 在独立容器中运行，故障隔离彻底
- **可替换**：替换 Agent 只需替换容器镜像，无需重新编译 Foundry 核心
- **可禁用**：通过配置跳过特定 Step 即可禁用 Agent

如果选择 Java + Step SDK 原生集成，Agent 与 Harness 的耦合度会增加，违反 Agent Is Replaceable 原则。

#### 3. 部署与运维

- **单一二进制**：Foundry 核心编译为单一二进制，部署简单
- **轻量容器**：Agent 执行器容器镜像小（Go 静态编译 + scratch 镜像 < 20MB）
- **资源效率**：Go 的内存占用远低于 JVM，适合在资源受限环境中运行多个 Agent

#### 4. 与 Harness 集成的实际路径

虽然 Go 无法直接使用 Harness Java Step SDK，但容器化插件模型是更优的集成路径：

```
Harness Pipeline
  └── Step (Plugin Step / Shell Script Step)
        └── 调用 Foundry CLI / gRPC API
              └── Foundry 调度 Agent（容器化执行）
```

Foundry 作为 Harness Step 的"编排层"，通过 CLI 或 gRPC API 接收 Harness 的调度指令，内部管理 Agent 的执行。这种松耦合方式比直接在 Delegate 进程内运行 Java 插件更符合 Foundry 的架构定位。

#### 5. gRPC 通信架构

gRPC 在 Foundry 中承担三个层次的通信职责：

**层次 1：Foundry 核心组件间通信**

Foundry 内部各模块（Agent 调度器、注册中心、审计服务、失败处理器）通过 gRPC 通信，实现模块解耦和独立演进：

```
Foundry CLI / API Gateway
  └── gRPC → Agent Scheduler
  └── gRPC → Registry Service
  └── gRPC → Audit Service
  └── gRPC → Failure Handler
```

**层次 2：Executor 接口定义**

gRPC + Protobuf 作为 Executor 接口的 IDL（接口定义语言），优势：

- **语言无关**：Protobuf 定义可生成 Go/Java/Python/TypeScript 等多种语言的客户端和服务端代码，Agent 可用任何语言实现
- **强类型契约**：Protobuf 的类型系统比 JSON Schema 更严格，编译时即可发现接口不匹配
- **向后兼容**：Protobuf 的字段编号机制天然支持接口演进，新增字段不破坏旧客户端
- **流式通信**：Agent 执行过程中的日志流、进度更新可通过 gRPC Server Streaming 实时推送

**层次 3：远程 Agent 调度**

远程大模型 API Agent 和跨网络 Agent 通过 gRPC 调度：

```
Foundry Scheduler
  └── gRPC → Remote API Agent（远程大模型服务）
  └── gRPC → Local CLI Agent（本地容器化执行）
  └── gRPC → Traditional CLI Agent（本地容器化执行）
  └── HTTP → Human Gate（人类审批通过 Web Hook）
```

**gRPC 与 HTTP API 的分工**：

| 通信场景 | 协议 | 理由 |
|---------|------|------|
| Foundry 内部组件间 | gRPC | 高性能、强类型、流式支持 |
| Executor 接口定义 | gRPC + Protobuf | 语言无关、编译时检查、向后兼容 |
| 远程 Agent 调度 | gRPC | 低延迟、双向流、连接复用 |
| 外部 API（CLI / Harness 调用） | HTTP + JSON | 兼容性好、调试方便、Harness 原生支持 |
| 人类 Gate 介入 | HTTP Web Hook | 浏览器/审批系统兼容 |

#### 6. Drone CI 生态参考

Drone CI（Harness 旗下开源项目）用 Go 编写，其容器化插件模型已被验证：

- 插件以 Docker 容器运行
- 通过环境变量接收输入
- 通过标准输出/文件系统输出结果
- 生态中有数百个社区插件

Foundry 的 Agent 执行模型可以直接复用这一成熟模式。

### 风险与缓解

| 风险 | 缓解措施 |
|------|---------|
| Go 泛型不够灵活，复杂模型定义冗长 | 使用 code generation 或泛型+接口组合模式；Task/Artifact 模型使用 JSON Schema 定义，Go 结构体由工具生成 |
| AI SDK 生态薄弱 | 远程大模型 API 通过 HTTP 客户端直接调用，不依赖高级 SDK；本地 AI CLI 工具通过子进程调用 |
| 无法使用 Harness Step SDK | 采用容器化插件模型，通过 Foundry CLI/API 作为集成桥梁 |
| Go 社区对复杂领域建模经验较少 | 参考 Kubernetes（Go 项目）的 API 模型设计模式 |

---

## 技术栈清单

### 核心技术栈

| 类别 | 选型 | 版本 | 理由 |
|------|------|------|------|
| 编程语言 | Go | 1.22+ | 稳定、并发原生、单一二进制部署 |
| 模块管理 | Go Modules | — | Go 官方标准 |
| 配置管理 | Viper / Koanf | latest | 结构化配置，支持多格式（YAML/JSON/ENV） |
| CLI 框架 | Cobra | latest | Go 生态标准 CLI 框架，Kubernetes/Hugo 等项目使用 |
| 日志 | Zap / Slog | latest | 结构化日志，满足审计需求；Slog 为 Go 1.21+ 标准库 |
| 数据校验 | go-playground/validator | latest | 结构体标签校验，保障 Task/Artifact 数据完整性 |
| HTTP 框架 | net/http (标准库) / Chi | latest | 优先标准库；Chi 作为轻量路由补充 |
| 容器交互 | Docker SDK for Go | latest | Agent 容器化执行的核心依赖 |
| YAML 处理 | gopkg.in/yaml.v3 | v3 | Pipeline/Stage/Step 配置定义 |
| JSON Schema | github.com/xeipuuv/gojsonschema | latest | Task/Artifact Schema 验证 |
| RPC 框架 | gRPC + Protobuf | latest | Foundry 核心组件间高性能通信；Executor 接口定义；Agent 远程调度 |

### 开发工具链

| 类别 | 选型 | 说明 |
|------|------|------|
| 代码规范 | golangci-lint | Go 生态标准 linter 集合 |
| 测试 | testing (标准库) + testify | 标准库 + 断言辅助 |
| Mock | gomock / testify/mock | 接口 Mock，支持 Executor 接口测试 |
| 文档生成 | godoc / pkgsite | API 文档自动生成 |
| Protobuf 构建 | buf | Protobuf 构建和代码生成工具，替代 protoc |
| gRPC 代码生成 | protoc-gen-go + protoc-gen-go-grpc | Go gRPC 代码生成插件 |
| CI | Harness CI (自身) | 吃自己的狗粮 |

### 不选用的技术

| 技术 | 不选用原因 |
|------|-----------|
| Spring Boot / Java | 违反 Agent Is Replaceable 原则，与 Harness 耦合过深 |
| 微服务框架 (go-kit/go-micro) | v1 使用 gRPC 原生服务化，不需要额外微服务框架 |
| ORM (GORM) | v1 不涉及数据库操作，审计持久化方案在 Task 6 中确定 |

---

## 项目目录架构

### 设计原则

1. **按领域划分**：目录结构与 Foundry 的核心领域模型对齐（Agent/Task/Artifact/Audit/Registry）
2. **接口与实现分离**：每个领域定义接口（interface），实现可替换
3. **内聚优先**：每个包（package）有明确的职责边界
4. **扁平化**：避免过深的嵌套，Go 社区推荐扁平包结构

### 目录结构

```
foundry/
├── cmd/                          # 入口
│   ├── foundry/                  # 主 CLI 入口
│   │   └── main.go
│   └── foundry-agent/            # Agent 执行器入口（容器内运行）
│       └── main.go
├── internal/                     # 内部包（不可被外部导入）
│   ├── agent/                    # Agent 领域
│   │   ├── executor.go           # Executor 胶水代码（调用 gen/ 中的 gRPC 生成代码）
│   │   ├── local_cli.go          # 本地 AI CLI Executor 实现
│   │   ├── remote_api.go         # 远程大模型 API Executor 实现
│   │   ├── traditional_cli.go    # 传统 CLI Executor 实现
│   │   ├── human_gate.go         # 人类工程师 Gate Executor 实现
│   │   └── constraint.go         # Agent 约束规范（FR-7）
│   ├── scheduler/                # Agent 调度层
│   │   ├── scheduler.go          # Agent Scheduler gRPC 服务
│   │   └── queue.go              # 调度队列管理
│   ├── gateway/                  # API Gateway
│   │   ├── gateway.go            # Gateway 路由（CLI / HTTP / gRPC）
│   │   ├── http_handler.go       # HTTP API Handler
│   │   └── grpc_handler.go       # gRPC API Handler
│   ├── model/                    # 核心数据模型
│   │   ├── task.go               # Task 模型定义
│   │   ├── artifact.go           # Artifact 模型定义
│   │   ├── context.go            # Context 模型定义
│   │   ├── workspace.go          # Workspace 模型定义
│   │   └── schema/               # JSON Schema 定义
│   │       ├── task.schema.json
│   │       └── artifact.schema.json
│   ├── registry/                 # Agent 注册与发现
│   │   ├── registry.go           # Registry 接口定义
│   │   ├── store.go              # 注册信息存储
│   │   └── discovery.go          # Agent 发现机制
│   ├── failure/                  # 失败处理
│   │   ├── detector.go           # 失败检测规则
│   │   ├── state_machine.go      # 失败状态机
│   │   ├── rollback.go           # 回滚机制
│   │   └── intervention.go       # 人工介入接口
│   ├── audit/                    # 流程审计
│   │   ├── logger.go             # 审计日志记录
│   │   ├── store.go              # 审计记录持久化
│   │   └── query.go              # 审计查询接口
│   ├── harness/                  # Harness 集成
│   │   ├── adapter.go            # Harness 适配器
│   │   ├── step_plugin.go        # Step 插件定义
│   │   └── mapping.go            # Pipeline/Stage/Step 映射
│   └── config/                   # 配置管理
│       ├── config.go             # 配置结构定义
│       └── loader.go             # 配置加载
├── pkg/                          # 可被外部导入的公共包
│   └── api/                      # 公共 API 定义
│       └── types.go              # 公共类型定义
├── proto/                        # Protobuf 定义
│   ├── foundry/v1/               # Protobuf 包路径
│   │   ├── executor.proto        # Executor 接口定义
│   │   ├── scheduler.proto       # Scheduler 服务定义
│   │   ├── task.proto            # Task 消息定义
│   │   ├── artifact.proto        # Artifact 消息定义
│   │   ├── registry.proto        # Registry 服务定义
│   │   ├── audit.proto           # Audit 服务定义
│   │   └── agent.proto           # Agent 生命周期消息
│   └── buf.yaml                  # Buf 配置（Protobuf 构建工具）
├── gen/                          # 生成代码（由 proto 生成，不手动编辑）
│   └── foundry/v1/               # 生成的 Go gRPC 代码
├── configs/                      # 配置文件
│   ├── foundry.yaml              # 主配置
│   └── agents/                   # Agent 配置目录
│       ├── local-cli.yaml
│       ├── remote-api.yaml
│       ├── traditional-cli.yaml
│       └── human-gate.yaml
├── schemas/                      # JSON Schema 文件
│   ├── task.schema.json
│   └── artifact.schema.json
├── test/                         # 集成测试
│   ├── integration/
│   └── fixtures/
├── docs/                         # 文档
│   ├── architecture.md
│   ├── pipeline_scenarios.md
│   └── design/
├── scripts/                      # 构建/部署脚本
│   ├── build.sh
│   └── docker-build.sh
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

### 模块职责说明

| 包 | 职责 | 对应 Task |
|-----|------|----------|
| `internal/agent/` | Executor 胶水代码与四种 Agent 类型实现 | Task 3 |
| `internal/scheduler/` | Agent Scheduler gRPC 服务，调度队列管理 | Task 3 |
| `internal/gateway/` | API Gateway（CLI / HTTP / gRPC 路由） | Task 7 |
| `internal/model/` | Task/Artifact/Context/Workspace 数据模型与 Schema | Task 2 |
| `internal/registry/` | Agent 注册、发现、配置管理 | Task 4 |
| `internal/failure/` | 失败检测、状态机、回滚、人工介入 | Task 5 |
| `internal/audit/` | 审计日志记录、持久化、查询 | Task 6 |
| `internal/harness/` | Harness 集成适配、Step 插件、映射 | Task 7 |
| `internal/config/` | 全局配置管理 | Task 1 |
| `proto/` | Protobuf 接口定义（Executor/Task/Artifact/Registry/Audit/Agent） | Task 2, Task 3 |
| `gen/` | 由 proto 生成的 Go gRPC 代码（不手动编辑） | Task 2, Task 3 |
| `cmd/foundry/` | 主 CLI 入口 | Task 1 |
| `cmd/foundry-agent/` | Agent 容器化执行入口 | Task 3 |

---

## 约束与限制

1. **v1 不引入数据库**：审计持久化方案在 Task 6 中确定，v1 可能使用文件系统或嵌入式存储
2. **v1 不引入 Web UI**：通过 CLI 交互，Web UI 留作 v2
3. **Go 版本锁定 1.22+**：利用泛型和标准库 Slog

## 待决问题

- [ ] 审计持久化存储选型（文件 / SQLite / 其他）——在 Task 6 中决定
- [ ] Foundry CLI 与 Harness 的具体通信方式（CLI 调用 vs HTTP API vs 两者兼有）——在 Task 7 中决定
- [ ] Agent 容器化执行的网络模式（host / bridge / none）——在 Task 3 中决定

---

## 修订历史

| 版本 | 日期 | 修改内容 | 作者 |
|------|------|---------|------|
| v1.0 | 2026-05-04 | 初始版本 | Foundry Team |
| v1.1 | 2026-05-04 | 新增 gRPC + Protobuf 到核心技术栈；补充 gRPC 通信架构三层设计；新增 proto/ 和 gen/ 目录；新增 buf 和 protoc-gen-go 工具链 | Foundry Team |
| v1.2 | 2026-05-04 | 架构定位从单体 CLI 更新为 gRPC 服务化；新增 scheduler/ 和 gateway/ 目录；executor.go 定位为胶水代码而非接口定义；新增 scheduler.proto；修正与 architecture.md 的冲突 | Foundry Team |
