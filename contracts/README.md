# 服务契约

本目录是知识服务对外的唯一契约位置：`openapi/` 存放 `/v1` OpenAPI，`CHANGELOG.md` 记录兼容性变更。目前尚未定义正式端点。

资源设计、错误结构、幂等、关联标识和兼容性规则见[技术方案第 11 节](../docs/KnowledgeService-技术方案.md)，这里不重复维护。

约定：

- OpenAPI 是契约测试的依据，由 S1 落地；变更须同步更新 `CHANGELOG.md`。
- `/v1` 内只增不改，弃用先公告并保留时间窗。
- 契约只描述对外资源，不暴露数据库字段、模型 SDK 或平台内部类型。
- Python 侧的 Capability Protocol 属于 `src/knowledge_service/capabilities/`，不放在这里。
