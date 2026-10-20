# 服务实现

后续在 `src/knowledge_service/` 下按 `domain`、`capabilities`、`application`、`providers`、`api`、`worker` 和 `composition.py` 组织，依赖方向与规则见[技术方案 2.2](../docs/KnowledgeService-技术方案.md)。

布局职责已确定，完整子目录以技术方案 2.2 为唯一规划。`application` 按摄取、资产、检索、回答、治理、质量及模型调用控制分片；`providers` 按 PostgreSQL、存储、身份、解析、LlamaIndex 转换、LangChain 模型和 Ragas 评测分片。框架对象不得进入领域与对外契约，适配器不得互相导入。

目录在对应阶段按需建立，不预建空包；S1a 生成实际包、依赖锁与架构检查。API／Worker 通过装配根复用应用用例，不独立维护模型配置和重试。当前没有可运行入口。
