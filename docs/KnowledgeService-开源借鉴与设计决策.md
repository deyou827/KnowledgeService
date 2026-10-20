# KnowledgeService 开源借鉴与设计决策

> 更新日期：2026-10-20
> 目的：把参考项目的设计转化为本产品的需求、模块、交互和验收，不复制整套应用  
> 项目入口：[现行方案与开发入口](README.md)

## 1. 结论与证据边界

优先参考 WeKnora 的知识组织与使用入口、RAGFlow 的解析透明度与人工干预、PAI-RAG 的企业服务化与治理方向。借鉴不等于依赖：本产品保留自己的领域模型、权限、任务、证据和交付契约。

第 3 节产品能力矩阵依据官方仓库 README 与公开架构说明。2026-10-16 补充了固定版本的依赖与解析源码抽查，采用范围、默认组件和可配置初值见 [ADR-0002](adr/0002-reference-backed-defaults.md)，存储版本见 ADR-0001。未完成整套源码安全审计、实际部署或浏览器交互体验，因此“官方描述”不证明其权限、可靠性和用户体验满足本方案。UI 交互建议是我们的设计转化，不声称逐屏复刻其实现。

引用的 main 分支会变化。采用具体实现时以 ADR 中的 commit／release 和文件为依据，并遵守相关许可；产品介绍不作为依赖锁或“最新版本”证明。以开源默认实现为主要依据，不因尚未完成本项目接入测试而重复展开多产品选型。

## 2. 参考资料

| 编号 | 官方资料 | 使用范围 |
| --- | --- | --- |
| R1 | Tencent／WeKnora 仓库 README | 知识组织、FAQ／Wiki、引用与任务等产品能力描述 |
| R2 | WeKnora 架构概览 | 主服务、解析服务、后台任务和后端适配的职责 |
| R3 | infiniflow／ragflow 仓库 README | 深度解析、模板化分块、可视化干预和引用描述 |
| R4 | aigc-apps／PAI-RAG 仓库 README | 多租户、RBAC、观测和 API 能力描述 |

产品介绍地址（2026-10-16 核对）；本轮固定源码抽查另见第 3.1 节：

- R1：`https://github.com/Tencent/WeKnora`
- R2：`https://github.com/Tencent/WeKnora/blob/main/website-docs/02-architecture/01-overview.md`
- R3：`https://github.com/infiniflow/ragflow`
- R4：`https://github.com/aigc-apps/PAI-RAG`

## 3. 借鉴矩阵

| 参考与官方描述 | 解决的问题 | 本产品采用的设计 | 对应需求／验证 |
| --- | --- | --- | --- |
| WeKnora：文档、FAQ、Wiki 等知识形态，R1 | 文件堆积无法覆盖人工知识维护 | 统一资产模型，基础知识页面与 FAQ 进入同一治理链 | FR-02／06；页面／FAQ 撤权与引用测试 |
| WeKnora：目录组织、引用定位、任务观察，R1 | 用户不知道资料在哪、答案从哪来、处理到哪一步 | 库工作台、原文定位侧栏、可恢复任务中心 | FR-02／03／08；完成导入到引用导航 |
| WeKnora：主服务、解析和异步处理分工，R2 | 大文件加工影响在线查询 | API 与 Worker 分进程，解析资源隔离，依赖以契约注入 | FR-03／11；解析故障不拖垮 API |
| RAGFlow：模板化分块与可视化人工干预，R3 | 解析错误只能反复上传或盲调参数 | 原文／解析／片段对照，修正为新产物，重建并重新审核 | FR-05／06；原文不被覆盖、旧引用不混用 |
| RAGFlow：可追踪引用与检索加工，R3 | 聊天答案无法核对、加工质量不可见 | EvidenceSet、引用定位、检索实验室与策略对照 | FR-07／08／10；结构与语义分别评测 |
| PAI-RAG：多租户、RBAC、观测与服务接口，R4 | 应用接口、组织治理和运行管理相互混杂 | 身份治理独立、Search／Answer 分离、审计和质量运营 | FR-01／11／12；隔离与接口契约验证 |

以上仅概括官方描述，不从“有 RBAC”“有引用”等功能名称推断实现安全。自己的权限矩阵、事务规则与故障用例仍必须独立设计和验证。

### 3.1 独立服务契约的固定源码依据

2026-10-19 针对身份与权限边界抽查以下官方固定版本文件。它们证明实现模式或功能支持范围，不证明本项目已接入，也不代表整库审计。

| 参考文件 | 实际观察 | 采用与不采用 |
| --- | --- | --- |
| WeKnora v0.8.2：`internal/middleware/rbac.go`、`website-docs/03-features/01-tenant-auth.md` | 机器 API Key scope 与 JWT 用户角色分开处理；机器分支交给 APIKeyGate；部分用户角色检查受 rollout 开关控制 | 采用机器主体与用户分离；不把 API Key 当用户委托，不复制关闭强制权限检查的上线过渡模式 |
| WeKnora 同版：`internal/handler/rbac_lookups.go` | 资源 owner lookup 中显式复查 tenant，避免仅凭 creator 匹配跨域放行 | 采用资源解析后的 Scope 复查；不照搬其组织／租户关系或跨租户超级用户特例 |
| RAGFlow v0.27.2：`api/apps/restful_apis/document_api.py` | 文档路由采用认证、租户上下文注入，上传路径进一步调用 `check_kb_team_permission` | 采用入口认证与业务资源权限分层；单一路径不证明所有历史／图片／引用端点有本项目要求的完整保护 |
| Keycloak 26.2.0：`docs/guides/securing-apps/token-exchange.adoc` | Standard V2 为受支持功能，覆盖同 realm 内部令牌交换；Legacy 的其他模式仍为 Preview | 默认采用单跳标准交换与认证客户端 profile，不依赖完整 actor 委托链、用户冒充或跨 realm 交换 |
| RFC 8693 第 1.1、2、4.1 节 | 区分 subject 与 actor，定义 Token Exchange 和 act 声明语义 | 用于解释协议边界；不把支持交换自动等同于支持完整委托链 |

固定来源定位：

- `https://github.com/Tencent/WeKnora/blob/3e8b0bfc80b845b2d4b2ed683994748741450a97/internal/middleware/rbac.go`
- `https://github.com/Tencent/WeKnora/blob/3e8b0bfc80b845b2d4b2ed683994748741450a97/internal/handler/rbac_lookups.go`
- `https://github.com/Tencent/WeKnora/blob/3e8b0bfc80b845b2d4b2ed683994748741450a97/website-docs/03-features/01-tenant-auth.md`
- `https://github.com/infiniflow/ragflow/blob/a024bea0cd93f39e6652a42bf84dd20c55bc560b/api/apps/restful_apis/document_api.py`
- `https://github.com/keycloak/keycloak/blob/26.2.0/docs/guides/securing-apps/token-exchange.adoc`
- `https://www.rfc-editor.org/rfc/rfc8693`

Keycloak 26.2.0 是功能语义参考，不是生产补丁版本锁定或安全背书；部署按 ADR-0002 固定维护补丁与 digest。本项目的 Operation、全证据回读、历史归属与失败关闭设计见 [ADR-0003](adr/0003-platform-decoupling-contract.md)，属于需求驱动的本地决策，不宣称直接复制了参考项目实现。基础技术栈和存储不因本轮契约优化重新选型。

### 3.2 最终组件选型与相对优势

2026-10-20 本轮以官方固定源码与组件源码补证。下表区分“上游确有此能力”和“本项目决定如何组合”，不宣称上述项目共同采用 LlamaIndex＋LangChain＋Ragas，也不以公司背景代替稳定性或安全验收。

| 核对来源 | 实际证据 | 本项目采用与限制 |
| --- | --- | --- |
| PAI-RAG v0.4.3 固定提交的 `pyproject.toml` | 包含 `llama-index-core = ^0.14.8` 及多种模型／向量后端适配 | 采用 LlamaIndex 摄取转换；该范围不是本项目锁定版本，不搬入完整索引与业务体系 |
| RAGFlow v0.27.2 固定提交的 `README.md` | 描述深度解析、模板分块、切片可视化与可追踪引用 | 采用解析预览、修正、定位与质量门禁；首版格式库并不等价于 DeepDoc 全部能力 |
| WeKnora v0.8.2 固定提交的 `README.md` | 描述解析、向量化、检索、推理模块化及切片编辑／历史能力 | 采用模块边界与检索调试思路；不照搬 Go 主服务、Agent 和连接器全集 |
| LlamaIndex 官方 `IngestionPipeline` 源码 | 提供 transformations、cache、docstore、vector_store 等组合入口 | 仅采用转换；默认不启用内容缓存和框架权威存储，项目负责发布与删除 |
| LangChain 官方 `langchain-openai` README | 明确该集成包通过官方 `openai` SDK 实现接入 | LangChain 作为模型 Provider 实现，不重复维护相同 SDK 封装，不保证任意兼容端点全功能一致 |
| Ragas 官方 Faithfulness 指标文档 | 检查回答断言是否被检索上下文支持 | 辅助质量回归，不当作事实真实性、安全性或业务准确率的单一证明 |

本轮直接读取的来源（后三项为可变分支，仅作为组件能力依据，安装前须锁定实际制品）：

- `https://github.com/aigc-apps/PAI-RAG/blob/fb06fc594692ad02afaf65b5cee96eb1f0a4b7d0/pyproject.toml`
- `https://github.com/infiniflow/ragflow/blob/a024bea0cd93f39e6652a42bf84dd20c55bc560b/README.md`
- `https://github.com/Tencent/WeKnora/blob/3e8b0bfc80b845b2d4b2ed683994748741450a97/README.md`
- `https://github.com/run-llama/llama_index/blob/main/llama-index-core/llama_index/core/ingestion/pipeline.py`
- `https://github.com/langchain-ai/langchain/blob/master/libs/partners/openai/README.md`
- `https://github.com/vibrantlabsai/ragas/blob/main/docs/concepts/metrics/available_metrics/faithfulness.md`

#### 解决的痛点与验收证据

| 痛点 | 设计措施 | 相对直接整套引入的预期优势 | 如何证明／代价 |
| --- | --- | --- | --- |
| 上传成功但内容不可用 | 隔离解析、LlamaIndex 转换、定位与解析预览 | 规则与首批企业资料一致，可追踪失败原因 | 固定格式样本、位置保真、OCR 显式报告；复杂版面能力初期少于成熟完整产品 |
| 接一个模型就重复写适配 | LangChain 模型 Provider，Rerank 独立接口 | 可替换模型而不改领域契约 | 真实端点契约测试；框架与 SDK 升级仍有维护成本 |
| 关键词不命中或语义结果不相关 | 授权向量＋全文＋RRF＋可选 Rerank | 单一可解释链路，便于同集调参 | Recall@K、拒答、延迟与成本对照；不预先承诺质量更高 |
| 撤权后历史或引用仍泄露 | 项目控制 ACL、EvidenceSet 与交付检查 | 与本项目历史权威要求直接一致 | 跨租户、撤权、删除、全证据回读测试；不据此声称上游不安全 |
| 多框架重复索引、模型与重试 | 一套发布事实、一个模型入口、一处恢复策略 | 减少状态分歧和重复外部调用风险 | 架构检查、故障注入与 Attempt 审计；组合适配需自己维护 |
| 修改参数后效果靠感觉 | 固定评估集、Ragas、确定性回归、人工抽检 | 改动效果可比较，指标能关联配置版本 | 记录失败样本和评判成本，不用单个总分替代验收 |
| 基础设施过多、平台强耦合 | 模块化单体＋同库任务／索引，平台仅 API 调用方 | 当前规模下减少部署组件与平台依赖 | 独立运行、备份恢复与容量测试；大规模不保证优于专用搜索引擎 |

这些是针对本项目需求的预期适配优势，不是性能排名或已实现成果。直接部署成熟产品通常在开箱即用、格式／连接器覆盖和完整交互上更有优势；本项目选择更可控的契约与生命周期，因此承担实现、运维及验证责任。

具体组件决策与放弃的方案见 [ADR-0004](adr/0004-rag-component-architecture.md)，不因依赖尚未锁定而重新启动全面选型。

本轮采用方式进一步收敛：参考机器／用户分离和后端资源授权，明确三类身份入口；保留 PostgreSQL＋Worker，区分 Operation、Job 和 Attempt；以执行／回答／交付三类状态表达结果。取消请求标记、外部结果未知停止自动重调、费用待核对与全证据回读均为本项目规则，不宣称参考项目已完整实现。无需引入 Celery 或新的工作流引擎，也不增加未经固定版本核对的组件依赖。

### 3.3 核心链路详细设计的源码依据

2026-10-20 为[核心链路详细设计](KnowledgeService-核心链路详细设计.md)补查下列固定文件。组件提交是本轮代码观察快照，不是正式稳定发行选择或本项目安装版本；没有安装、运行或完成整库安全审计。下面只记录可观察的实现，不将组合方案伪称某一家开源产品原样采用。

| 编号 | 固定来源与定位 | 实际观察及设计转化 |
| --- | --- | --- |
| D1 | PAI-RAG `fb06fc594692ad02afaf65b5cee96eb1f0a4b7d0`；`integrations/pairag-file/pairag/file/nodeparsers/positional_markdown_parser.py` 的 `Section.to_chunk` 与子块处理 | TextNode metadata 保存 page_bbox／token_count；超长块存在按 token 比例推算 bbox 的处理。本项目采用位置映射，但必须标注估算精度 |
| D2 | RAGFlow `a024bea0cd93f39e6652a42bf84dd20c55bc560b`；`rag/svr/task_executor.py` 的 `build_chunks`、`embedding`、`set_progress` | 任务分段加工、进度及取消检查与向量计算分开。本项目保留阶段可观察性，租约、审核等待和发布事务是本地设计 |
| D3 | WeKnora `3e8b0bfc80b845b2d4b2ed683994748741450a97`；`internal/application/service/knowledgebase_search_fusion.go` 的 `fuseWithRRF` | 通道内排名从 1 开始，按片段 ID 合并并计算加权 RRF；采用算法，不直接复制模型／后端原始分数归一化 |
| D4 | LlamaIndex `619d74664e8404612038bc15a1a40c686037277c`；`llama-index-core/llama_index/core/ingestion/pipeline.py` | 未传 transformations 时默认 SentenceSplitter＋Settings.embed_model；disable_cache 默认为 false。项目必须显式传纯转换列表并关缓存，不依赖默认模型 |
| D5 | LangChain `afafd55d34cb11b794781fd0f7bef9127c633f84`；`libs/partners/openai/langchain_openai/chat_models/base.py`、`embeddings/base.py` | chat 把 max_retries 传给底层客户端；embedding 默认重试为 2，提供 check_embedding_ctx_length 控制。项目显式 max_retries=0，并把文本长度／批次控制留在业务调用层 |
| D6 | pgvector `v0.8.0`；`README.md` 的 Indexing、Hybrid Search、不同维度 FAQ | 默认精确近邻；全文可配 RRF／cross-encoder；vector 列可不固定维度。项目查询按配置与维度隔离，先不上 ANN |
| D7 | PostgreSQL `REL_17_0`；`doc/src/sgml/ref/select.sgml` 的锁定子句 | SKIP LOCKED 适用于队列表多消费者，但不是通用一致读取。本项目只用于任务领取 |
| D8 | Ragas `298b68274234c060deacab3cf5fb52aa3a20e885`；`docs/concepts/metrics/available_metrics/faithfulness.md` | 指标检查 response 的断言被 retrieved context 支持的比例；不能据此证明原文事实正确或权限安全 |

可追溯来源：

- D1：`https://github.com/aigc-apps/PAI-RAG/blob/fb06fc594692ad02afaf65b5cee96eb1f0a4b7d0/integrations/pairag-file/pairag/file/nodeparsers/positional_markdown_parser.py`
- D2：`https://github.com/infiniflow/ragflow/blob/a024bea0cd93f39e6652a42bf84dd20c55bc560b/rag/svr/task_executor.py`
- D3：`https://github.com/Tencent/WeKnora/blob/3e8b0bfc80b845b2d4b2ed683994748741450a97/internal/application/service/knowledgebase_search_fusion.go`
- D4：`https://github.com/run-llama/llama_index/blob/619d74664e8404612038bc15a1a40c686037277c/llama-index-core/llama_index/core/ingestion/pipeline.py`
- D5：`https://github.com/langchain-ai/langchain/blob/afafd55d34cb11b794781fd0f7bef9127c633f84/libs/partners/openai/langchain_openai/chat_models/base.py`
- D5：`https://github.com/langchain-ai/langchain/blob/afafd55d34cb11b794781fd0f7bef9127c633f84/libs/partners/openai/langchain_openai/embeddings/base.py`
- D6：`https://github.com/pgvector/pgvector/blob/v0.8.0/README.md`
- D7：`https://github.com/postgres/postgres/blob/REL_17_0/doc/src/sgml/ref/select.sgml`
- D8：`https://github.com/vibrantlabsai/ragas/blob/298b68274234c060deacab3cf5fb52aa3a20e885/docs/concepts/metrics/available_metrics/faithfulness.md`

上述源码阅读只形成详细设计依据；真实接口、格式样本、并发故障与质量回归仍是对应阶段的验收工作。

## 4. 架构决策

| 决策 | 采用理由 | 不采用／暂缓 |
| --- | --- | --- |
| 模块化单体＋独立加工进程 | 领域清晰、故障隔离，同时控制部署成本 | 不照搬参考项目全部服务拓扑 |
| 事实存储与派生索引分离 | 授权、版本和生命周期不随搜索后端迁移 | 不让向量数据库成为业务数据库 |
| 不可变原文＋可修正派生产物 | 兼顾人工纠错与来源可追溯 | 不直接编辑历史片段覆盖证据 |
| 多种知识形态统一治理 | 页面、FAQ 与文件都需要权限、发布与引用 | 不先开发完整 Wiki 协同编辑器 |
| 授权检索与全证据交付检查 | 模型可能受未引用内容影响，不能只检查引用 | 不把参考项目的引用 UI 当安全证明 |
| 搜索、问答、管理三类入口 | 普通读者不应理解索引参数才能找到知识 | 不把所有操作堆在聊天窗口 |
| 质量问题到修复回归闭环 | 优化依据真实失败而非功能数量 | Rerank 接入并对照后决定启用；不默认引入图谱或 Agent 搜索 |

## 5. 交互转化要求

以下为本产品的设计要求，待实现时通过原型与用户场景验收：

1. 导入使用分步向导：来源、目标与权限、策略预览、提交、任务；上传完成不等于发布。
2. 文档详情分原文、解析、片段、版本，点击片段能定位原文；纠错保留来源和审计。
3. 搜索门户先呈现范围、筛选和结果；检索通道分数、预算等高级信息留给实验室。
4. 答案旁提供引用侧栏，可核对具体段落和修订；校验失败隐藏正文，不显示伪造确定性分数。
5. 治理状态和加工状态分开：审核通过但构建失败、更新失败但旧版可用都能解释清楚。
6. 批量操作展示逐项结果，失败可恢复；同步健康与文档有效性不能混为一个绿色图标。
7. 反馈明确共享范围，处理人有权才能看问题与证据；质量运营不绕过隐私边界。

## 6. 正式实现前的参考验证

每个将要采用的模块记录：用户问题 → 官方证据 → 固定版本 → 可观察行为 → 本产品取舍 → 验收用例。不得把未体验的截图或功能介绍写成已完成产品评审。

| 研究任务 | 验证重点 | 截止阶段 |
| --- | --- | --- |
| 资料导入与解析修正体验 | 状态层次、定位、纠错、失败恢复是否真正闭环 | S2 页面设计前 |
| 知识组织与引用体验 | 目录／标签／页面／FAQ 的关系，引用打开方式 | S3／S4 对应页面设计前 |
| 权限与企业协作源码抽查 | 后端过滤位置、组变更、下载／历史／引用授权 | S1 契约设计时，不等到上线 |
| 加工与发布源码抽查 | 幂等、租约、并发发布、删除后任务行为 | S2 实现前 |
| 质量运营与调试体验 | 反馈是否能落到资料修复和策略回归 | S5 设计前 |

这些任务是有边界的设计验证，不授权自动安装整套参考产品、下载大模型或发送企业资料。需要真实体验时先采用合成样本和受控环境；基础组件引入需单独检查许可证、版本和维护情况。
