# Foundry 常见场景 Pipeline 模板

| 属性 | 内容 |
|------|------|
| **文档标题** | Foundry 常见场景 Pipeline 模板 |
| **文档作者** | Foundry Team |
| **文档日期** | 2026-05-04 |
| **文档版本** | v1.0 |
| **文档描述** | 列举 6 个典型 DevOps 场景的 Pipeline 定义，帮助理解 Foundry 的编排能力和设计规律 |

---

## 概述

本文档通过 6 个典型场景的 Pipeline 模板，展示 Foundry 如何编排不同类型的 Agent 完成从代码提交到应急响应的全生命周期工作。每个 Pipeline 标注了 Agent 类型、Gate 位置、Artifact 传递关系和失败处理策略。

### 读者对象

- 想要理解 Foundry 项目目标的架构师和管理者
- 需要定义 Pipeline 模板的 DevOps 工程师
- 需要理解 Agent 协作模型的开发者

### 前置知识

阅读本文档前，建议先阅读 [architecture.md](architecture.md) 了解 Foundry 的整体架构。

### 符号约定

| 符号 | 含义 |
|------|------|
| ⚡ | 并行执行的 Step |
| 🚦 Gate | 人工介入点，需要人类确认 |
| 📦 | Artifact 产出汇总 |
| 🔄 | 失败处理分支 |
| 📝 | 审计相关说明 |

---

## 场景 1：代码提交到生产发布

> 最经典的 CI/CD 场景，对应架构图中 Agent 协作模型的基础用例。自动化程度最高，AI Agent 参与最少。

```yaml
Pipeline: 代码提交到生产发布
触发: 开发者 git push 到 main 分支
```

### Stage 1：代码质量门禁

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 1.1 静态代码分析 | SonarQube | 传统 CLI | Task(扫描代码) + Workspace(仓库路径) | 代码质量报告 |
| 1.2 安全漏洞扫描 | Trivy | 远程 API | Task(扫描依赖漏洞) + Workspace(仓库路径) | 安全扫描报告 |
| 1.3 代码规范检查 | ESLint / golangci-lint | 传统 CLI | Workspace(仓库路径) | 规范检查报告 |

⚡ Step 1.1 / 1.2 / 1.3 并行执行

📦 汇总 Artifact：`quality-gate-report`

### Stage 2：自动化测试

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 2.1 单元测试 | Maven Surefire / go test | 传统 CLI | Context(引用 quality-gate-report 中的覆盖率基线) | 单元测试报告 + 覆盖率数据 |
| 2.2 集成测试 | Testcontainers | 传统 CLI | Workspace(代码 + 测试配置) | 集成测试报告 |
| 2.3 API 契约测试 | Pact | 传统 CLI | Workspace(契约定义 + 代码) | 契约测试报告 |

⚡ Step 2.1 / 2.2 / 2.3 并行执行

📦 汇总 Artifact：`test-report`

### Stage 3：构建与打包

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 3.1 编译打包 | Maven Package / go build | 传统 CLI | Workspace(代码) | 二进制产物 |
| 3.2 容器镜像构建 | Kaniko | 传统 CLI | Workspace(Dockerfile + 二进制) | 容器镜像 |
| 3.3 部署清单渲染 | Helm | 传统 CLI | Workspace(Chart + values) | Helm Chart |

⚡ Step 3.1 / 3.2 / 3.3 并行执行

📦 汇总 Artifact：`build-artifacts`

### 🚦 Gate：生产发布审批

| 属性 | 内容 |
|------|------|
| Agent | 技术负责人 [人类 Gate] |
| 输入 | 所有 Stage 的 Artifact 摘要 |
| 动作 | 通过 / 退回 / 拒绝 |

### Stage 4：生产部署

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 4.1 部署到生产环境 | ArgoCD | 传统 CLI | Context(引用 build-artifacts) | 部署记录 |
| 4.2 生产冒烟测试 | Newman | 传统 CLI | Context(生产环境地址 + 测试集合) | 冒烟测试报告 |

### 失败处理

| 失败场景 | 处理策略 |
|---------|---------|
| SonarQube 扫描超时 | Retry（最多 2 次），仍失败则 Rollback + 通知运维 |
| 单元测试失败 | Rollback 到 Stage 2 之前，通知开发者修复 |
| Docker 镜像构建失败 | Retry（可能是临时网络问题），仍失败则通知运维 |
| 冒烟测试失败 | Rollback 部署，通知运维回退版本 |

---

## 场景 2：功能开发全流程

> 从需求到交付的完整研发流程。AI Agent 参与最多，人类把关最重。体现 Foundry 编排 AI Agent 的核心能力。

```yaml
Pipeline: 功能开发全流程
触发: 产品经理提交需求单
```

### Stage 1：需求分析

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 1.1 需求文档解析 | Claude Code | 本地 AI CLI | Task(解析需求描述，产出结构化需求规格) | requirement-spec.json |
| 1.2 需求可行性评估 | Claude Code | 本地 AI CLI | Context(引用 requirement-spec.json) | feasibility-report.json |

### 🚦 Gate：需求确认

| 属性 | 内容 |
|------|------|
| Agent | 产品经理 + 技术负责人 [人类 Gate] |
| 动作 | 确认需求范围 / 退回补充 / 拒绝 |

### Stage 2：技术设计

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 2.1 技术方案设计 | Claude Code | 本地 AI CLI | Context(引用 requirement-spec.json) | technical-design.json |
| 2.2 接口契约定义 | Claude Code | 本地 AI CLI | Context(引用 technical-design.json) | api-contract.json |
| 2.3 影响范围分析 | Claude Code | 本地 AI CLI | Context(引用 technical-design.json + 现有代码库) | impact-analysis.json |

⚡ Step 2.1 先执行，2.2 / 2.3 可并行（依赖 2.1 的 Artifact）

### 🚦 Gate：设计评审

| 属性 | 内容 |
|------|------|
| Agent | 架构师 [人类 Gate] |
| 动作 | 确认技术方案 / 退回修改 / 拒绝 |

### Stage 3：编码实现

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 3.1 核心代码编写 | Claude Code | 本地 AI CLI | Task(实现需求) + Context(引用 technical-design.json, api-contract.json) + Workspace(代码仓库) | source-code-diff.patch |
| 3.2 单元测试编写 | Claude Code | 本地 AI CLI | Context(引用 source-code-diff.patch + api-contract.json) | test-code-diff.patch |
| 3.3 代码自检 | golangci-lint / ESLint | 传统 CLI | Workspace(包含新代码的工作区) | lint-report.json |

### 🚦 Gate：代码评审

| 属性 | 内容 |
|------|------|
| Agent | 高级开发工程师 [人类 Gate] |
| 动作 | 确认代码质量 / 退回修改 / 要求补充测试 |

### Stage 4：验证测试

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 4.1 单元测试执行 | go test / Maven Surefire | 传统 CLI | Workspace(代码 + 测试) | unit-test-report.xml |
| 4.2 集成测试执行 | Testcontainers | 传统 CLI | Workspace(代码 + 测试配置) | integration-test-report.json |
| 4.3 代码覆盖率检查 | gcov / JaCoCo | 传统 CLI | Workspace(代码 + 覆盖率数据) | coverage-report.json |

⚡ Step 4.1 / 4.2 / 4.3 并行执行

### 🚦 Gate：测试结果确认

| 属性 | 内容 |
|------|------|
| Agent | 开发者 [人类 Gate] |
| 动作 | 确认测试通过 / 退回修复 |

### Stage 5：交付

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 5.1 变更日志生成 | Claude Code | 本地 AI CLI | Context(引用 requirement-spec.json + source-code-diff.patch) | CHANGELOG.md |
| 5.2 产物归档 | 归档脚本 | 传统 CLI | Workspace(代码 + 产物) | release-package.tar.gz |

### 失败处理

| 失败场景 | 处理策略 |
|---------|---------|
| AI 产出的需求规格不完整 | Gate 退回，人类批注补充点，AI 修正后重新提交 |
| 技术方案被架构师否决 | Gate 退回，附修改意见，AI 重新设计 |
| 代码评审不通过 | Gate 退回，附评审意见，AI 修正代码 |
| 单元测试失败 | Rollback 到 Stage 3，通知 AI Agent 修复代码 |

---

## 场景 3：Bug 修复流程

> 紧急但需要质量保障的修复流程。强调快速定位 + 精准修复 + 回归验证。

```yaml
Pipeline: Bug 修复流程
触发: Bug 报告提交（含错误日志、复现步骤）
```

### Stage 1：问题定位

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 1.1 错误日志分析 | Claude Code | 本地 AI CLI | Task(分析错误日志，定位可能原因) | error-analysis.json |
| 1.2 代码溯源 | Claude Code | 本地 AI CLI | Context(引用 error-analysis.json + 代码库) | code-trace-report.json |
| 1.3 根因确认 | Claude Code | 本地 AI CLI | Context(引用 error-analysis.json + code-trace-report.json) | root-cause-report.json |

Step 1.1 → 1.2 → 1.3 串行执行（每步依赖上一步的 Artifact）

### 🚦 Gate：修复方案确认

| 属性 | 内容 |
|------|------|
| Agent | 开发负责人 [人类 Gate] |
| 动作 | 确认修复方案 / 调整方案 / 升级处理 |

### Stage 2：修复实现

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 2.1 修复代码编写 | Claude Code | 本地 AI CLI | Task(按修复方案编写修复代码) + Context(引用 root-cause-report.json) + Workspace(代码仓库) | fix-patch.diff |
| 2.2 回归测试用例编写 | Claude Code | 本地 AI CLI | Context(引用 fix-patch.diff + root-cause-report.json) | regression-test-code.diff |

### 🚦 Gate：修复代码评审

| 属性 | 内容 |
|------|------|
| Agent | 高级开发工程师 [人类 Gate] |
| 动作 | 确认修复正确 / 退回修改 |

### Stage 3：验证

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 3.1 原始 Bug 复现验证 | 测试脚本 | 传统 CLI | Context(引用 Bug 报告中的复现步骤) | bug-reproduction-result.json |
| 3.2 回归测试 | go test / Maven Surefire | 传统 CLI | Workspace(修复后代码) | regression-test-report.xml |
| 3.3 影响范围回归 | 集成测试套件 | 传统 CLI | Workspace(修复后代码) | impact-regression-report.json |

⚡ Step 3.1 / 3.2 / 3.3 并行执行

### Stage 4：发布

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 4.1 热修复发布 | ArgoCD | 传统 CLI | Context(引用 fix-patch.diff) | hotfix-deployment-record.json |

### 失败处理

| 失败场景 | 处理策略 |
|---------|---------|
| AI 定位根因错误 | Gate 退回，人类指出正确方向，AI 重新分析 |
| 修复代码引入新问题 | 回归测试拦截 → Rollback → 通知 AI 修正 |
| Bug 无法复现 | Step 3.1 标记为"无法复现" → 通知人类进一步排查 |
| 热修复部署失败 | 自动回滚到修复前版本 → 通知运维 |

---

## 场景 4：安全漏洞应急响应

> 高优先级场景。强调速度 + 审计 + 人类全程把控。审计记录不可跳过，即使 P0 场景。

```yaml
Pipeline: 安全漏洞应急响应
触发: 安全扫描发现 CRITICAL 漏洞 / 收到 CVE 通报
```

### Stage 1：漏洞评估

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 1.1 漏洞影响分析 | Claude Code | 本地 AI CLI | Task(分析 CVE 详情，评估对当前系统的影响) | cve-impact-analysis.json |
| 1.2 漏洞范围扫描 | Trivy / Aqua | 远程 API | Context(引用 CVE 编号) | vulnerability-scan-report.json |

⚡ Step 1.1 / 1.2 并行执行

### 🚦 Gate：应急等级判定

| 属性 | 内容 |
|------|------|
| Agent | 安全负责人 [人类 Gate] |
| 动作 | 判定 P0(立即修复) / P1(24h内修复) / P2(下个迭代修复) |
| 特殊规则 | P0 场景可跳过部分 Gate，但审计记录不可跳过 |

### Stage 2：修复实施

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 2.1 依赖升级/补丁应用 | Claude Code | 本地 AI CLI | Task(升级受影响依赖到安全版本) + Context(引用 vulnerability-scan-report.json) | dependency-upgrade-patch.diff |
| 2.2 安全配置加固 | Claude Code | 本地 AI CLI | Task(根据 CVE 建议加固安全配置) | security-config-patch.diff |

⚡ Step 2.1 / 2.2 并行执行

### 🚦 Gate：修复方案评审

| 属性 | 内容 |
|------|------|
| Agent | 安全工程师 [人类 Gate] |
| 动作 | 确认修复有效 / 退回补充 |

### Stage 3：验证

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 3.1 漏洞复测 | Trivy | 远程 API | Context(修复后的代码/镜像) | rescan-report.json |
| 3.2 回归测试 | 完整测试套件 | 传统 CLI | Workspace(修复后代码) | regression-report.xml |
| 3.3 安全回归测试 | OWASP ZAP | 传统 CLI | Context(目标服务地址) | security-regression-report.json |

⚡ Step 3.1 / 3.2 / 3.3 并行执行

### Stage 4：应急发布

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 4.1 安全补丁发布 | ArgoCD | 传统 CLI | Context(引用修复产物) | security-patch-deployment.json |

📝 强制审计：全流程审计记录不可跳过，即使 P0 场景

### 失败处理

| 失败场景 | 处理策略 |
|---------|---------|
| 漏洞复测仍存在 | 修复不彻底 → Rollback → 通知 AI 补充修复 |
| 回归测试失败 | 修复引入新问题 → Rollback → 通知 AI 修正 |
| 安全回归发现新漏洞 | 升级处理 → 开启新的应急响应流程 |
| 补丁部署失败 | 自动回滚 → 通知运维 |

---

## 场景 5：基础设施变更

> IaC 场景。强调变更影响评估 + 审批 + 回滚能力。破坏性变更需要更高级别审批。

```yaml
Pipeline: 基础设施变更
触发: Terraform / Helm 配置变更提交
```

### Stage 1：变更分析

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 1.1 配置 Diff 分析 | Claude Code | 本地 AI CLI | Task(分析配置变更内容，识别影响) + Workspace(包含 terraform plan 输出) | infra-change-analysis.json |
| 1.2 成本影响评估 | Claude Code | 本地 AI CLI | Context(引用 infra-change-analysis.json) | cost-impact-report.json |
| 1.3 合规性检查 | Open Policy Agent | 传统 CLI | Workspace(变更配置) | compliance-report.json |

⚡ Step 1.1 / 1.2 / 1.3 并行执行

### 🚦 Gate：变更审批

| 属性 | 内容 |
|------|------|
| Agent | 运维负责人 + 架构师 [人类 Gate] |
| 输入 | 变更分析 + 成本影响 + 合规报告 |
| 动作 | 批准变更 / 退回修改 / 拒绝 |
| 特殊规则 | 破坏性变更必须 CTO 级别审批 |

### Stage 2：预发验证

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 2.1 预发环境变更 | Terraform Apply | 传统 CLI | Workspace(配置 + 目标=预发环境) | staging-apply-result.json |
| 2.2 预发环境冒烟测试 | 基础设施测试套件 | 传统 CLI | Context(预发环境地址) | staging-smoke-test-report.json |

### 🚦 Gate：预发验证确认

| 属性 | 内容 |
|------|------|
| Agent | 运维工程师 [人类 Gate] |
| 动作 | 确认预发正常 / 退回 |

### Stage 3：生产变更

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 3.1 生产环境变更 | Terraform Apply | 传统 CLI | Workspace(配置 + 目标=生产环境) | prod-apply-result.json |
| 3.2 生产环境健康检查 | 健康检查脚本 | 传统 CLI | Context(生产环境地址) | health-check-report.json |

🔄 失败处理：如果 Step 3.1 失败 → 自动触发 Terraform Destroy 回滚 → 通知运维负责人

### 失败处理

| 失败场景 | 处理策略 |
|---------|---------|
| Terraform Plan 显示破坏性变更 | Gate 标记为"破坏性" → 需要 CTO 审批 |
| 预发环境变更失败 | 自动回滚预发 → 通知运维排查 |
| 生产环境变更失败 | 自动 Terraform Destroy 回滚 → 通知运维 |
| 健康检查失败 | 通知运维 → 人工决定是否回滚 |

---

## 场景 6：技术债务治理

> 非紧急但长期重要的场景。强调渐进式改进 + 风险可控 + 性能基准对比。

```yaml
Pipeline: 技术债务治理
触发: 技术负责人发起治理任务（如"升级 Spring Boot 2.x → 3.x"）
```

### Stage 1：债务评估

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 1.1 现状分析 | Claude Code | 本地 AI CLI | Task(分析当前技术债务状况) + Workspace(代码仓库) | tech-debt-inventory.json |
| 1.2 升级影响分析 | Claude Code | 本地 AI CLI | Context(引用 tech-debt-inventory.json + 目标版本信息) | upgrade-impact-analysis.json |
| 1.3 自动化兼容性检查 | OpenRewrite | 传统 CLI | Workspace(代码仓库) | compatibility-report.json |

⚡ Step 1.1 / 1.2 / 1.3 并行执行

### 🚦 Gate：治理方案确认

| 属性 | 内容 |
|------|------|
| Agent | 架构师 [人类 Gate] |
| 动作 | 确认治理范围和优先级 / 调整 / 拒绝 |

### Stage 2：自动化迁移

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 2.1 自动代码迁移 | OpenRewrite | 传统 CLI | Task(执行可自动迁移的变更) | auto-migration-patch.diff |
| 2.2 手动迁移项处理 | Claude Code | 本地 AI CLI | Task(处理需手动迁移的变更项) + Context(引用 compatibility-report.json 中的手动项) | manual-migration-patch.diff |

⚡ Step 2.1 / 2.2 可并行执行

### 🚦 Gate：迁移代码评审

| 属性 | 内容 |
|------|------|
| Agent | 高级开发工程师 [人类 Gate] |
| 动作 | 确认迁移正确 / 退回修改 |

### Stage 3：全面验证

| Step | Agent | 类型 | 输入 | 输出 Artifact |
|------|-------|------|------|--------------|
| 3.1 编译验证 | Maven / go build | 传统 CLI | Workspace(迁移后代码) | compilation-result.json |
| 3.2 完整测试套件 | 测试运行器 | 传统 CLI | Workspace(迁移后代码) | full-test-report.xml |
| 3.3 性能基准对比 | JMeter / wrk | 传统 CLI | Context(迁移前后的性能基线) | performance-comparison.json |

⚡ Step 3.1 / 3.2 / 3.3 并行执行

### 🚦 Gate：治理结果确认

| 属性 | 内容 |
|------|------|
| Agent | 架构师 + 性能工程师 [人类 Gate] |
| 动作 | 确认治理达标 / 退回继续优化 / 降级处理 |

### 失败处理

| 失败场景 | 处理策略 |
|---------|---------|
| 编译失败 | 迁移代码有语法错误 → 通知 AI 修正 |
| 测试失败 | 迁移引入回归 → Rollback → 通知 AI 修正迁移逻辑 |
| 性能下降超过阈值 | 性能基准对比拦截 → 通知架构师决策（接受/优化/回滚） |
| 自动迁移覆盖不全 | compatibility-report 中标记为"需手动处理" → AI Agent 接手 |

---

## 场景对比总结

### 按维度对比

| 维度 | 场景1: 代码发布 | 场景2: 功能开发 | 场景3: Bug修复 | 场景4: 安全应急 | 场景5: 基础设施 | 场景6: 技术债务 |
|------|---------------|---------------|--------------|---------------|---------------|---------------|
| AI Agent 占比 | 低 | 高 | 中 | 中 | 低 | 高 |
| 人类 Gate 数量 | 1 | 4 | 2 | 2 | 2 | 3 |
| 紧急程度 | 中 | 低 | 高 | 极高 | 中 | 低 |
| 失败容忍度 | 低 | 中 | 低 | 极低 | 极低 | 中 |
| 审计严格度 | 标准 | 标准 | 标准 | 强制 | 标准 | 标准 |
| 回滚粒度 | Stage | Step | Stage | Stage | Stage | Step |

### 按 Agent 类型统计

| Agent 类型 | 使用场景 | 典型职责 |
|-----------|---------|---------|
| 本地 AI CLI | 场景2/3/4/6 | 需求解析、技术设计、代码编写、日志分析、迁移处理 |
| 远程 API | 场景1/4 | 安全扫描、漏洞扫描 |
| 传统 CLI | 全部场景 | 静态分析、测试执行、编译构建、部署、合规检查 |
| 人类 Gate | 全部场景 | 需求确认、设计评审、代码评审、发布审批、应急判定 |

---

## 设计规律

### 规律 1：Stage 之间串行，Step 之间并行

```
Stage 1 ──→ Stage 2 ──→ Stage 3
  │并行│      │并行│      │并行│
  S1.1       S2.1         S3.1
  S1.2       S2.2         S3.2
  S1.3       S2.3         S3.3
```

Stage 是阶段边界，必须全部完成才能进入下一个 Stage。Step 是最小调度单元，同一 Stage 内可以并行执行。

### 规律 2：Gate 出现在"不可逆"操作之前

| 不可逆操作 | 前置 Gate |
|-----------|----------|
| 发布到生产环境 | 生产发布审批 |
| 基础设施变更 | 变更审批 |
| 确认需求范围 | 需求确认 |
| 确认技术方案 | 设计评审 |
| 确认修复方案 | 修复方案确认 |

### 规律 3：AI Agent 做"创造性"工作，传统 CLI 做"确定性"工作

| 工作性质 | Agent 类型 | 例子 |
|---------|-----------|------|
| 分析、设计、编码 | 本地 AI CLI | 需求解析、技术方案、代码编写、日志分析 |
| 检查、测试、构建 | 传统 CLI | lint、单元测试、Docker 构建、合规检查 |
| 扫描、检测 | 远程 API | 安全扫描、漏洞扫描 |
| 审批、决策 | 人类 Gate | 需求确认、设计评审、发布审批 |

### 规律 4：每个 Step 的输入输出都是结构化的

没有"让 AI 自由发挥"的 Step。每个 Step 都有：

- **Task**：明确要做什么
- **Context**：在什么条件下做
- **Workspace**：在哪里做
- **Artifact**：交付什么（必须是结构化的）

### 规律 5：失败处理策略因场景而异

| 失败类型 | 处理策略 | 典型场景 |
|---------|---------|---------|
| 临时性故障（服务不可用） | Retry | Trivy API 超时 → 重试 |
| 业务 Bug（代码有问题） | Rollback + 人工修复 | 单元测试失败 → 退回编码 |
| 格式错误（输出不规范） | 人工介入 | Artifact 校验失败 → 通知运维 |
| 破坏性失败（基础设施变更失败） | 自动回滚 | Terraform Apply 失败 → Destroy |

### 规律 6：审计不可跳过

无论场景紧急程度如何，审计记录始终不可跳过。即使在 P0 安全应急场景下可以跳过部分 Gate，审计记录仍然必须写入。

---

## 修订历史

| 版本 | 日期 | 修改内容 | 作者 |
|------|------|---------|------|
| v1.0 | 2026-05-04 | 初始版本 | Foundry Team |
