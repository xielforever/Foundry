# Foundry v1 - Progress Tracking

| 属性 | 内容 |
|------|------|
| **文档标题** | Foundry v1 - Progress Tracking |
| **文档作者** | Foundry Team |
| **文档日期** | 2026-05-06 |
| **文档版本** | v1.9 |
| **文档描述** | Foundry v1 设计文档阶段任务进度追踪 |

---

## 任务进度

| Task | 名称 | 优先级 | 状态 | 依赖 | 阻塞原因 |
|------|------|--------|------|------|----------|
| Task 1 | [技术栈选型与项目架构文档](.trae/specs/foundry-v1/tasks.md) | P0 | ✅ 已完成（v1.3 修订） | — | — |
| Task 2 | [核心数据模型规范文档（Task & Artifact）](.trae/specs/foundry-v1/tasks.md) | P0 | ✅ 已完成（v1.6 修订） | Task 1 | — |
| Task 3 | [Agent Executor 架构设计文档](.trae/specs/foundry-v1/tasks.md) | P0 | ✅ 已完成（v1.3 修订） | Task 2 | — |
| Task 4 | [Agent 注册与发现机制设计文档](.trae/specs/foundry-v1/tasks.md) | P0 | ✅ 已完成 | Task 2, Task 3 | — |
| Task 5 | [失败处理与人工介入机制设计文档](.trae/specs/foundry-v1/tasks.md) | P0 | ✅ 已完成 | Task 2, Task 3 | — |
| Task 6 | [流程审计方案设计文档](.trae/specs/foundry-v1/tasks.md) | P0 | ✅ 已完成 | Task 2, Task 3 | — |
| Task 7 | [Harness 集成方案设计文档](.trae/specs/foundry-v1/tasks.md) | P0 | ✅ 已完成（v1.2 修订） | Task 3, Task 5, Task 6 | — |
| Task 8 | [可复用流程模板文档](.trae/specs/foundry-v1/tasks.md) | P1 | ✅ 已完成 | Task 7 | — |
| Task 9 | [文档间一致性审查与修订](.trae/specs/foundry-v1/tasks.md) | P1 | ⬜ 待开始 | Task 8 | — |

> Task 4、Task 5、Task 6 可并行执行

## 状态说明

- ⬜ 待开始
- 🔄 进行中
- ✅ 已完成
- ❌ 阻塞（需在「阻塞原因」列填写具体原因和解除条件）

## Open Questions 追踪

> 对应 [spec.md](.trae/specs/foundry-v1/spec.md) 中的 Open Questions 章节，追踪每个待决问题的解决状态。
> 各设计文档中新增的待决问题（OQ-x.x）也在此统一追踪。

### 来自 spec.md 的待决问题

| 待决问题 | 解决任务 | 当前状态 | 备注 |
|----------|----------|----------|------|
| 具体技术栈选择（编程语言、框架等） | Task 1 | ✅ 已解决 | Go 1.22+，容器化插件模型集成 Harness，详见 docs/design/tech_stack_and_architecture.md |
| Harness 具体版本和集成方式 | Task 7 | ✅ 已解决 | Harness SaaS + Plugin Step，详见 docs/design/harness_integration.md |
| 第一版流程模板的具体场景 | Task 8 | ✅ 已解决 | 功能开发全流程 + 代码提交到生产发布，详见 docs/design/flow_templates.md |

### 来自设计文档的待决问题

| 编号 | 待决问题 | 来源文档 | 解决任务 | 当前状态 |
|------|---------|---------|---------|---------|
| OQ-3.1 | Executor 是否支持自定义启动前脚本（initContainer） | agent_executor_architecture.md | Task 7 | ✅ 已解决（v1 不支持） |
| OQ-3.2 | gRPC Execute 的负载均衡策略 | agent_executor_architecture.md | Task 4 | ✅ 已解决 | 
| OQ-3.3 | Artifact 产出后是否自动触发下游 Step | agent_executor_architecture.md | Task 7 | ✅ 已解决（v1 不支持） |
| OQ-3.4 | Executor Metrics 的更细粒度指标 | agent_executor_architecture.md | Task 5 | ✅ 转移至 OQ-5.4 |
| OQ-3.5 | Human Gate 的 Web UI 形态 | agent_executor_architecture.md | Task 7 | ✅ 已解决（v1 通过 CLI 操作） |
| OQ-3.6 | Agent 类型分类维度是否需要重新设计 | agent_executor_architecture.md | — | ✅ 已解决（v1.2 Capabilities 层） |
| OQ-4.1 | Registry 是否需要提供 gRPC 反射服务 | agent_registry_and_discovery.md | Task 7 | ✅ 已解决（v1 不提供） |
| OQ-4.2 | Agent 注册是否需要认证 | agent_registry_and_discovery.md | Task 7 | ✅ 已解决（v1 不要求，网络层隔离） |
| OQ-4.3 | 自定义 Artifact 类型的 Schema 注册机制 | agent_registry_and_discovery.md | Task 9 | ⬜ 待解决 |
| OQ-5.1 | Artifact Store 是否支持 RolledBack 状态的物理清理 | failure_handling_and_human_intervention.md | 编码阶段 | ⬜ 待解决 |
| OQ-5.2 | 回滚操作的原子性保障 | failure_handling_and_human_intervention.md | Task 7 | ✅ 已解决（状态标记而非物理删除） |
| OQ-5.3 | 介入操作的 Web UI 形态 | failure_handling_and_human_intervention.md | Task 7 | ✅ 已解决（v1 通过 CLI 操作） |
| OQ-5.4 | Executor Metrics 的资源级指标 | failure_handling_and_human_intervention.md | 编码阶段 | ⬜ 待解决 |
| OQ-6.1 | Artifact Store 是否需要独立于审计存储 | audit_scheme.md | 编码阶段 | ⬜ 待解决 |
| OQ-6.2 | 审计日志的合规标准 | audit_scheme.md | 编码阶段 | ⬜ 待解决 |
| OQ-6.3 | 实时审计流的协议选择 | audit_scheme.md | v2 | ⬜ 待解决 |
| OQ-6.4 | Agent stdout/stderr 的审计存储 | audit_scheme.md | Task 7 | ✅ 已解决（写入 Harness 日志，不写入 Foundry AuditStore） |
| OQ-7.1 | Harness Delegate 与 Foundry Core 的 mTLS 配置 | harness_integration.md | 编码阶段 | ⬜ 待解决 |
| OQ-7.2 | SubmitTask 长连接的超时处理 | harness_integration.md | 编码阶段 | ⬜ 待解决 |
| OQ-7.3 | 多个 Harness Pipeline 并发执行时的资源隔离 | harness_integration.md | 编码阶段 | ⬜ 待解决 |

### 跨文档同步任务

| 同步项 | 来源 | 目标 | 计划执行 | 状态 |
|--------|------|------|---------|------|
| RolledBack 状态同步到 Artifact 生命周期状态机 | Task 5 | Task 2 | Task 9 | ✅ 已完成（v1.6 同步） |
| AgentRegistrationSpec/Runtime 拆分同步到相关文档 | Task 4 | README/CLAUDE | — | ✅ 已完成 |

## 变更记录

| 日期 | 关联 Task | 变更内容 |
|------|-----------|----------|
| 2026-05-04 | — | 初始化进度追踪文档 |
| 2026-05-04 | Task 1 | 完成 Task 1：技术栈选型（Go）与项目架构文档。关键决策：Go 1.22+、gRPC+Protobuf 三层通信架构、容器化插件模型集成 Harness、proto/foundry/v1/ 目录约定 |
| 2026-05-04 | Task 2 | 完成 Task 2：核心数据模型规范文档（Task & Artifact），含 Protobuf 定义和 JSON Schema。关键决策：Task 不含 status 字段（状态与数据分离）、parameters 使用 map<string,string>（非 Struct）、ArtifactContent 支持 data/file_path 二选一、ArtifactType 使用枚举（非自由字符串）、common.proto 提取共享枚举避免循环依赖 |
| 2026-05-05 | — | 增强跨会话上下文传递：tasks.md 增加 Required Reading 字段、CLAUDE.md 增加前置产出物读取步骤、PROGRESS.md 变更记录补充关键决策摘要 |
| 2026-05-05 | Task 3 | 完成 Task 3：Agent Executor 架构设计文档。关键决策：Executor 以 gRPC Service 暴露（Execute/GetCapabilities/HealthCheck）、Execute 方法四条不变式、四种 Agent 类型各有独立接入规范（配置示例+执行模型+失败模式+Artifact大小限制）、FR-7 四项约束（C-1~C-4）+四层保障机制、Agent 执行生命周期状态机（12个状态）、Scheduler调度流程（9步）；解决 Task 2 遗留的三个待决问题（Artifact大小限制/调度结果记录/产出数量约束） |
| 2026-05-05 | Task 3 | 评审修订 v1.1：修复5个阻塞项（LocalAiCli网络模式矛盾→bridge、gRPC消息大小→file_path引用、retry_count语义→移至Scheduler、HumanGate超时独立、ARTIFACT_COUNT_EXCEEDS_MAX同步到Task2）；修复6个建议项（7处拼写错误、artifact_type确定性规则、配置结构说明表、并发控制机制、Validating拆分、import common.proto）；同步更新Task 2至v1.4 |
| 2026-05-05 | Task 2 | v1.4 修订（由 Task 3 评审触发）：新增校验错误码 ARTIFACT_COUNT_EXCEEDS_MAX（Warning 级别） |
| 2026-05-05 | Task 3 | v1.2 Capabilities 能力声明层：解决Agent类型分类维度混合问题（执行方式vs能力），保留4种AgentType作为执行模板，增加capabilities能力声明作为第二维度；v1预定义8种能力（ai_reasoning/tool_use/code_generation/code_review/security_scan/deterministic/approval/notification）；新增ARTIFACT_TYPE_EXECUTION_RECORD（枚举值11）解决通知类Agent无自然产出问题；Scheduler匹配逻辑从5级扩展为6级；同步更新Task 2至v1.5 |
| 2026-05-05 | Task 2 | v1.5 修订（由 Task 3 v1.2 触发）：新增 ARTIFACT_TYPE_EXECUTION_RECORD（枚举值 11），用于通知类 Agent 无自然产出时由 Foundry Core 自动构造的执行记录 Artifact；同步更新枚举表、Protobuf 定义、2 处 JSON Schema |
| 2026-05-05 | — | 一致性审查：更新 README/CLAUDE/PROGRESS/project_rules 以反映 Task 1-3 产出物（新增设计文档目录项、Capabilities 概念、术语表补充、命名规范补充）；project_rules 新增 2 条禁止项（Executor 返回流程控制指令、Agent 产出无结构文本）；新增 protobuf_schema_standards.md 规则；design_doc_standards.md 增加跨文档同步规则 |
| 2026-05-05 | Task 5 | 完成 Task 5：失败处理与人工介入机制设计文档。关键决策：4种失败检测维度（D-1输出无效/D-2格式错误/D-3超时/D-4人工中断）、Step执行失败状态机（15个状态）、6种人工介入操作（approve/reject/correct/rollback/cancel/skip）、3种回滚粒度（Step/Stage/Pipeline）+4种触发条件+级联回滚、Artifact生命周期新增RolledBack状态、ERROR_REPORT Artifact自动构造、Pipeline级别on_failure策略配置、Intervention数据结构+超时升级机制。⚠️ 待同步：RolledBack 状态需同步到 Task 2 的 Artifact 生命周期状态机（Task 9 一致性审查时执行） |
| 2026-05-05 | Task 6 | 完成 Task 6：流程审计方案设计文档。关键决策：AuditEvent核心结构（16个字段）+15种事件类型、存储选型SQLite默认+PostgreSQL可选（AuditStore抽象层）、9个数据库索引支持多维度查询、敏感数据脱敏规则（键名模式匹配+URL凭证+显式标记）、Artifact Store与审计日志共享SQLite数据库、gRPC+REST双协议查询接口+Pipeline时间线查询+JSONL/CSV导出、保留策略（90天+空间限制+归档） |
| 2026-05-05 | Task 4 | 完成 Task 4：Agent 注册与发现机制设计文档。关键决策：AgentRegistrationSpec（11个静态字段）+AgentRuntimeState（6个动态字段）拆分、7条注册校验规则、注册/注销流程（含异常注销心跳超时两级策略）、6级发现算法（AgentType→ArtifactType→Capabilities→Labels→Status→负载均衡）+3种负载均衡策略（最少并发优先/轮询/随机）、CapabilityRegistry能力注册表（v1预注册8种能力+自定义能力配置注册）、YAML配置文件格式+动态启用/禁用（CLI命令）+有限热重载+Executor侧热重载处理规范、心跳机制（30s间隔+90s超时+60s自动注销）、内存存储+60s快照持久化、进程内事件通道通知Scheduler；gRPC接口（ExecutorRegistry，3个方法）与Go内部接口（Registry，8个方法）分层；解决Task 3待决问题OQ-3.2（负载均衡策略：v1默认最少并发优先） |
| 2026-05-05 | Task 4 | 评审修订 v1.1：1) 拆分 AgentRegistration 为 AgentRegistrationSpec（静态）+ AgentRuntimeState（动态），RegisterRequest 只传递 Spec；2) 拆分 gRPC 接口（ExecutorRegistry，3 个方法）和 Go 内部接口（Registry，8 个方法）；3) 统一发现算法过滤顺序与 Task 3 一致（AgentType→ArtifactType→Capabilities）；4) 补充 Agent 类型专属配置映射表（RemoteApi/HumanGate→parameters 键）；5) 补充 Executor 侧热重载处理规范 |
| 2026-05-05 | Task 5 | 评审修订 v1.1：B-5.1 修正状态机 Retrying→RetryExhausted 转换逻辑；B-5.2 修正失败检测流程 D-1/D-2 触发时机；B-5.3 补充 ErrorReportContent 与标准 Artifact 结构映射；S-5.1 新增 T-4 判定规则和 SECURITY_VULNERABILITY_DETECTED 错误码；S-5.2 新增 Intervention 存储方案；S-5.3 新增 on_failure 与 retry_limit 交互规则；S-5.4 FailureContext 改用 ArtifactRef 引用；S-5.5 完善跨文档同步声明 |
| 2026-05-05 | Task 6 | 评审修订 v1.1：B-6.1 修正 audit.proto ExecutionStatus import 问题；B-6.2 新增 AuditService 部署架构说明；S-6.1 AuditEventInput 新增 required_capabilities 字段；S-6.2 新增 L-6.8 content_data 大小限制；S-6.3 收窄脱敏规则 auth 模式；S-6.4 新增 AuditQuerySort.field 合法值约束；S-6.5 修正审计写入保障表述；S-6.6 修正 OQ-5.1 引用；新增跨文档同步说明 |
| 2026-05-05 | — | PROGRESS v1.6 评审修订：1) 修正 Task 4 变更记录（AgentRegistration 拆分为 Spec+Runtime、发现算法顺序修正）；2) 补充 Task 4/5/6 评审修订变更记录；3) Open Questions 追踪从 3 项扩展为 17 项（新增 OQ-3.1~3.6、OQ-4.1~4.3、OQ-5.1~5.4、OQ-6.1~6.4）；4) 新增跨文档同步任务追踪表 |
| 2026-05-05 | Task 7 | 完成 Task 7：Harness 集成方案设计文档。关键决策：Harness SaaS + Plugin Step 集成方式（4维度选型对比）、9组概念映射+5种边界情况处理、Foundry Step Plugin Docker镜像（10输入参数+5输出+退出码映射+同步阻塞执行流程）、流程控制权矩阵（11项，Harness掌管流程控制，Foundry掌管Agent调度）、三层通信架构+K8s网络拓扑、Harness failure strategy统一MarkAsFailure（重试由Foundry内部管理）、双层审计（Harness日志+Foundry AuditStore，stdout/stderr不写入AuditStore）、回滚原子性（状态标记而非物理删除）、人工介入通过Foundry CLI操作（6种操作映射退出码）；新增SchedulerService gRPC接口（SubmitTask+GetTaskStatus）；解决8个待决问题（OQ-3.1/3.3/3.5/4.1/4.2/5.2/5.3/6.4） |
| 2026-05-05 | Task 7 | 评审修订 v1.1：B-7.1 SubmitTaskRequest 新增 agent_id 字段（支持精确调度模式，agent_id 非空时跳过发现算法）；B-7.2 修正 Registry.Discover 调用签名与 Task 4 一致（使用完整 DiscoverRequest）；B-7.3 同步更新 spec.md Open Questions（Harness 版本选型标记为已解决）；B-7.4 SubmitTaskResponse 新增 intervention_id 字段（人工介入时 Plugin 可获取 intervention_id）；S-7.1 修正退出码语义（退出码 3 仅用于 CANCELLED，人工介入 cancel 退出码为 1）；S-7.2 OQ-7.1~7.3 同步到 PROGRESS.md；S-7.3 新增跨文档同步说明；S-7.4 Artifact Store 配置补充 |
| 2026-05-06 | Task 1-7 | 跨文档同步修正（Task 1-7 重新评审后执行）：Task 2 v1.6（RolledBack 状态/SECURITY_VULNERABILITY_DETECTED 错误码/ExecutionStatus 枚举/intervention.proto/TaskSpec 新增 expected_artifact_counts+required_capabilities）；Task 1 v1.3（待决问题 3 项已解决/存储约束更新/proto 目录更新）；Task 3 v1.3（精确调度模式说明/OQ-3.1~3.5 已解决）；Task 7 v1.2（on_failure 改用 OnFailureStrategy 枚举/import intervention.proto） |
| 2026-05-06 | Task 8 | 完成 Task 8：可复用流程模板设计文档。关键决策：模板 YAML 格式规范（元数据+参数定义+Pipeline定义）、参数引用机制（${param_name}语法）、2 个完整流程模板（功能开发全流程：5 Stage/9 Step/4 Gate/4 种 Agent 类型；代码提交到生产发布：4 Stage/8 Step/1 Gate/3 种 Agent 类型）、Artifact 流转路径 Mermaid 图、失败处理策略表、模板扩展规范、模板与 Harness Pipeline 映射规则；解决 spec.md 最后一个 Open Question（第一版流程模板的具体场景） |
