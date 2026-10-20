# 独立知识门户前端

技术选型为 React、TypeScript、Ant Design、TanStack Query。这里只建立前端职责入口，尚无 package.json、依赖锁或可运行页面；实际实现按 S3b 推进，不从平台复制业务组件。

目标组织为 `src/app/`（路由、认证入口和装配）、`src/features/`（documents、search、chat、citations、debug、governance）和 `src/shared/`（通用 UI 与 API 客户端）。完整结构以[技术方案 2.2](../docs/KnowledgeService-技术方案.md#22-代码结构与依赖规则)为准，子目录随实际代码建立。

前端仅消费服务契约：BFF 管会话，后端执行授权；不持有模型密钥、不自行检索数据库、不长期复制答案兜底。SSE 只传过程进度与结果引用，完整正文经 GET Answer 校验交付，历史与引用按当前权限回读。

按文档列表／处理状态 → 单库搜索问答 → 引用回链 → 检索调试 → 治理交互渐进实现，不先建设完整控制台。常规验证使用开发编译与必要测试，不默认生产构建。
