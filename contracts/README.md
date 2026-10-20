# 服务契约

本目录是知识服务对外的唯一契约位置：`openapi/` 存放 `/v1` OpenAPI，`CHANGELOG.md` 记录兼容性变更。目前尚未生成正式 OpenAPI 文件或实现端点。

资源设计、错误结构、幂等、关联标识和兼容性规则见[技术方案第 11 节](../docs/KnowledgeService-技术方案.md)，这里不重复维护。

首批字段、状态、接口与 SSE 设计输入见[核心链路详细设计第 8 节](../docs/KnowledgeService-核心链路详细设计.md#8-首批-api-规格)，操作与交付权威仍以 ADR-0003 为准。S1a 将其转成正式 schema、正负样例和契约测试；文档表格不等同于可执行 OpenAPI。

约定：

- OpenAPI 是契约测试的依据，由 S1 落地；变更须同步更新 `CHANGELOG.md`。
- `/v1` 内只增不改，弃用先公告并保留时间窗。
- 契约只描述对外资源，不暴露数据库字段、模型 SDK 或平台内部类型。
- Python 侧的 Capability Protocol 属于 `src/knowledge_service/capabilities/`，不放在这里。
