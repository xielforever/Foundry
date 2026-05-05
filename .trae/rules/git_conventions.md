---
alwaysApply: false
description: 进行 Git 提交、创建分支、版本控制操作时使用此规则，通过 #Rule 引用
---

## Git 提交信息规范

提交信息格式：

```
<type>(<scope>): <subject>

<body>
```

### 类型 (type)

| 类型 | 用途 |
|------|------|
| `feat` | 新功能 |
| `fix` | 修复 |
| `docs` | 文档 |
| `design` | 设计文档 |
| `refactor` | 重构 |
| `test` | 测试 |
| `chore` | 构建/工具 |

### 范围 (scope)

| 范围 | 含义 |
|------|------|
| `agent` | Agent 抽象 |
| `executor` | Executor 执行层 |
| `task` | Task 数据模型 |
| `artifact` | Artifact 数据模型 |
| `harness` | Harness 集成 |
| `audit` | 审计 |
| `registry` | Agent 注册发现 |
| `failure` | 失败处理 |
| `rollback` | 回滚机制 |
| `flow` | 流程模板 |

### 示例

```
docs(agent): Add Agent Executor architecture design document
design(task): Design Task data model specification
feat(harness): Implement Harness Step plugin integration
```

## 分支规范

| 分支 | 用途 |
|------|------|
| `main` | 主线，始终可发布 |
| `feature/*` | 功能开发分支 |
| `fix/*` | 修复分支 |
| `docs/*` | 文档分支 |

**禁止 AI 自动提交代码，除非用户明确要求。**

## 版本标签规范

| 规则 | 说明 |
|------|------|
| 标签格式 | `vX.Y.Z`（语义化版本号） |
| 设计文档阶段 | 不打 tag，版本号仅在文档头中管理 |
| 编码阶段里程碑 | 每个里程碑完成后打 tag，如 `v0.1.0`（首个可运行版本） |
| 发布版本 | 正式发布时打 tag，如 `v1.0.0` |

## 变更范围规范

| 变更类型 | 要求 |
|---------|------|
| 设计文档变更 | 必须在修订历史中记录，涉及跨文档同步时声明同步范围 |
| 代码变更 | 必须追溯到对应的设计文档，PR 描述中引用设计文档路径 |
| 规则文件变更 | 影响所有后续工作，需在提交信息中明确说明变更原因 |
