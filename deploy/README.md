# 本地部署

后续建立 Dockerfile、Docker Compose 与无敏感值的环境变量示例。当前尚无可执行部署配置。

本地企业联调的组成见[技术方案第 14 节](../docs/KnowledgeService-技术方案.md)：API、Worker、PostgreSQL＋pgvector、对象存储、Keycloak。默认仅绑定 loopback 地址，凭据运行时注入，数据目录与源码分离。数据库迁移是显式一次性作业，放在顶层 `migrations/`，不在应用启动时执行。
