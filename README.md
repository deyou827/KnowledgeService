# KnowledgeService

独立企业级知识库产品，提供知识门户、知识工作台、治理控制台和对外知识 API，通过明确契约服务 OrchestrationPlatform 及其他应用。

## 开发入口：先读方案

**[现行方案与阅读顺序](docs/README.md)** 是项目内的统一开发入口，产品需求、技术方案、开源借鉴、实施与验收计划正文均位于本仓库 `docs/`。

项目文档随代码一起维护，使用相对链接，不依赖个人知识库路径。开发前必须阅读当前阶段相关方案，具体要求见 [AGENTS.md](AGENTS.md)。

## 当前状态

项目初始化阶段：仅建立职责边界、目录和实施顺序，尚无可运行服务、正式接口契约或部署配置。

平台侧基线位于 `../OrchestrationPlatform`，对应提交 `c7e1223`。平台现有知识问答是本地词法检索与摘录基线，不代表独立服务已完成；不在此项目直接导入平台的 Python 模块或共享平台数据库。

## 职责边界

| 模块 | 职责 |
| --- | --- |
| KnowledgeService | 资料来源、解析、文档版本、索引发布、权限过滤检索、问答、引用与证据校验、受控历史回读 |
| OrchestrationPlatform | 平台入口、Agent/Run 编排、平台身份与项目上下文、平台自身模型配置、知识服务 Provider 适配；不拥有知识服务内部模型配置 |
| 服务契约 | 认证身份与作用域、资源标识、请求响应、错误码、幂等与版本语义 |

服务必须自行验证调用方身份、Tenant／Workspace 与资源权限；平台 Project 显式映射到 Workspace，不能信任调用方自报的作用域。共享服务凭据不自动等于终端用户知识访问权限；委托方案按现行技术方案与阶段门禁确认。

独立项目不要求马上采用复杂部署。先通过本地 Docker Compose 验证单服务闭环，再决定生产部署和存储规模。

## 目录

```text
docs/            PRD、技术方案、设计依据、实施计划、开发入口与 adr/
contracts/       对外 OpenAPI 与兼容性变更记录
src/             knowledge_service 包：domain、capabilities、application、providers、api、worker
tests/           architecture、unit、security、integration、fault、evaluation 分层测试
deploy/          本地 Compose 与后续部署配置
web/             独立前端职责与目录入口，尚无应用实现
migrations/      显式迁移作业职责入口，尚无迁移脚本
```

`web/` 与 `migrations/` 当前只有职责说明；应用子目录、迁移脚本和 `cli/` 在对应阶段按需建立，不预建空包。依赖方向与完整结构见[技术方案 2.2](docs/KnowledgeService-技术方案.md)。

## 开发依据

产品范围、技术选型、需求编号与阶段门禁统一以 [现行方案](docs/README.md) 为准，不在这里维护另一份实施顺序。

默认架构为模块化单体、PostgreSQL＋pgvector、对象存储、OIDC／Keycloak 和独立加工 Worker；LlamaIndex 负责摄取转换，LangChain 负责模型适配，Ragas 辅助评测，业务与授权检索由项目控制。理由与边界见 [ADR-0004](docs/adr/0004-rag-component-architecture.md)，最终架构图及目录规划见[技术方案第 2 节](docs/KnowledgeService-技术方案.md#2-系统结构与依赖方向)。依赖精确版本、模型配置及验收仍须按阶段完成，不代表已有可运行实现。

M0 核心链路、V1 企业试点版、后续平台集成是不同交付边界，不得混称。每阶段先确认范围，再实现和验证；不把模拟测试或引用合法率当作真实问答质量验收。

项目相关的核心设计、架构决策和重要结论维护在本仓库 `docs/`；跨项目通用学习可另记个人知识库，不复制项目方案正文。
