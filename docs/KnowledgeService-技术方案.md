# KnowledgeService 技术方案

> 更新日期：2026-10-15  
> 现行目标架构；不代表对应模块已实现或完成生产验证  
> 需求来源：[产品需求文档](KnowledgeService-产品需求文档.md)；实施顺序：[实施与验收计划](KnowledgeService-实施与验收计划.md)

## 1. 结论与架构原则

采用模块化单体、Capability Contract、可替换 Provider 和异步加工架构，构建独立企业级知识服务。完整设计知识资产、身份治理、内容加工、检索问答、质量运营和运行保障，但不把每个模块拆成服务。

默认部署包含 API、摄取 Worker、PostgreSQL＋pgvector、对象存储和 OIDC 身份服务；本地企业联调使用 Keycloak，线上可替换为兼容 OIDC 的企业 IdP。API 与 Worker 同代码库、不同进程；高风险解析在受资源限制的子进程或容器执行，不与 API 共用执行进程。

核心边界：

- PostgreSQL 保存业务事实、授权、版本、任务、清单、证据和审计；检索索引可以重建，不成为治理权威。
- 对象存储保存不可变原文件、网页快照和大型解析产物；数据库保存对象标识、版本和指纹。
- Runtime 和公共契约不依赖数据库驱动、模型 SDK、框架对象或平台内部类型。
- 后台任务至少一次执行，通过幂等、租约和 fencing 防止重复提交；不承诺外部模型调用恰好一次。
- 检索前过滤、外发前检查、交付前重新授权；不能只靠前端隐藏或检索后剔除。
- 自研领域与流程，窄接口使用成熟组件；不引入一个包办摄取、Agent、记忆和权限的黑盒框架。

## 2. 系统结构与依赖方向

```text
知识门户 / 知识工作台 / 治理控制台 / 外部应用
                         ↓
              HTTP API + 身份认证 + 限流
                         ↓
 Application：资产、发布、检索、回答、治理、质量用例
                         ↓
           Domain + Capability Contracts
                         ↑
 Providers：数据库、对象存储、身份、解析、模型、检索、连接器

 Composition Root：配置并注入默认 Provider
 Worker：领取可靠任务，调用相同应用用例
 Scheduler：创建同步／失效／清理任务，不直接修改业务结果
```

Domain 不导入 FastAPI、ORM、模型 SDK 或 OrchestrationPlatform。Application 依赖契约，Provider 实现契约，装配层组织集成。HTTP DTO 与领域对象分离，避免把持久化字段直接暴露给调用者。

### 2.1 模块职责

| 模块 | 负责 | 不负责 |
| --- | --- | --- |
| Identity & Governance | 主体映射、成员、授权、委托、审计规则 | 生成检索向量或回答 |
| Knowledge Catalog | 库、目录、资产、元数据、来源和生命周期 | 存储后端特有索引语法 |
| Ingestion & Publishing | 同步、解析、分块、构建、审核绑定和发布 | 自行扩大文档 ACL |
| Retrieval & Evidence | 查询计划、授权召回、融合、重排、证据预算 | 将候选直接当最终答案 |
| Grounded QA & Delivery | 回答、引用校验、历史和交付控制 | 任意文件访问、工具执行 |
| Quality Operations | 反馈、评测、对照实验和修复验证 | 绕过资源授权读取所有知识 |
| Operations | 配额、任务运行、告警、清理和恢复 | 将技术日志作为正文存储 |

## 3. 默认技术栈与扩展条件

| 层 | 默认方向 | 替换或扩展条件 |
| --- | --- | --- |
| API | Python、FastAPI、Pydantic | 锁定版本后以 OpenAPI／契约测试控制升级 |
| 数据访问 | SQLAlchemy＋psycopg、Alembic | 复杂检索 SQL 可显式实现，不强塞进 ORM |
| 事实存储 | PostgreSQL | 单一权威来源，暂不维护第二套生产 Repository |
| 全文检索 | 中文分词 Provider＋PostgreSQL 全文检索 | 用中英文、术语和授权过滤样本验证；不将其自动称为 BM25 |
| 向量检索 | pgvector，先精确再评估 ANN | 以目标容量、召回和过滤测试决定索引类型 |
| 原文件 | S3 兼容对象存储，产品待 V-02 验证 | 本地文件仅作测试替身，不替代企业验收的对象存储链路 |
| 任务与事件 | PostgreSQL 任务表、Outbox、独立 Worker | 吞吐和故障域证明需要后再引入队列中间件 |
| 身份 | OIDC；本地 Keycloak | 企业 IdP 通过标准契约接入，不绑定其内部表或专有 SDK |
| 前端 | React、TypeScript、Ant Design、TanStack Query | 按门户／工作台／控制台组织，不复制平台业务组件 |
| 模型 | 独立 Answer／Embedding／Rerank Provider | OpenAI-compatible 为候选默认接口，本地／外部由外发策略决定 |
| 观测 | 结构化日志、指标、Trace 契约 | OpenTelemetry 作为出口候选，后端选型按部署确定 |
| 部署 | 本地 Compose，应用无状态化接口 | 多副本、托管数据库和独立搜索后端由实测驱动 |

依赖版本、解析库和对象存储产品必须用官方资料与样本验证后锁定。默认后端只实现一套；OpenSearch／Milvus 属于可替换的派生索引 Provider，不同时上线形成长期双写。

## 4. 领域模型与版本关系

| 对象 | 责任与关键约束 |
| --- | --- |
| Tenant / Workspace | 企业与业务空间；所有资源关联带 Scope，平台 project 显式映射 |
| Principal / Group / Membership | 已验证主体、用户组和成员关系；外部主体用 issuer＋subject 标识 |
| RoleBinding / AccessPolicy | 资源动作授权、文档限制、策略版本；不把邮箱或客户端请求当授权事实 |
| KnowledgeBase / Folder / MetadataSchema | 治理容器、导航和元数据约束；目录首版无独立 ACL 继承 |
| Source / Connector / SyncCursor | 来源配置、同步范围、健康与游标；密钥仅保存引用 |
| Document | 稳定逻辑资产；类型为文件、网页、知识页面或 FAQ；持有治理状态 |
| DocumentRevision | 不可变内容版本、来源快照、作者、指纹；原文变化才产生内容修订 |
| ParseArtifact | 某修订的解析／清洗产物、配置与位置映射；人工修正生成新产物 |
| ChunkSet / Chunk | 绑定解析产物与分块策略的不可变片段；不覆盖历史片段 |
| Publication / Approval | 被批准的内容修订与解析产物、有效期和策略；审核不等于已上线 |
| IndexVersion / Manifest | 特定库的可查询清单，绑定发布对象、片段、模型和策略 |
| IngestionJob / JobAttempt | 输入快照、幂等、阶段、租约、fencing 和安全错误码 |
| QueryPlan | 主体授权上下文、选库、各库索引快照、查询策略、预算和配置 |
| RetrievalCandidate | 通道内候选与排名；不等于有资格进入模型的证据 |
| EvidenceSet / EvidenceItem | 实际送入模型的全部证据；保存来源、版本、位置、指纹的引用，不保存正文副本，回读时按引用重取并重新授权 |
| Answer / Citation | 结构化状态与正文；引用只能指向该 Answer 的 EvidenceSet |
| Feedback / QualityIssue | 用户授权共享的反馈及处理状态，不默认复制问答正文 |
| EvaluationDataset / Run | 受控数据、标签、模型、策略、索引和指标快照 |
| AuditEvent / UsageLedger / OutboxEvent | 治理事件、用量与可靠异步传播，均无正文和凭据 |

### 4.1 四种变化不能混为一个版本

1. 内容变化：新 DocumentRevision。
2. 解析方式或人工修正变化：新 ParseArtifact，原文件仍不变。
3. 分块变化：新 ChunkSet。
4. 模型、检索方式或可查询集合变化：新 IndexVersion／Manifest。

向量模型切换不伪造原文修订；人工修改解析文本不覆盖原始文件。审核绑定内容修订与解析产物，因此加工结果发生实质变化必须重新审核。仅重建兼容索引可复用批准内容，但保留完整构建记录。

人工知识页面使用 Markdown／受控结构化内容；FAQ 使用问题、标准回答和来源关联。它们进入同一版本、发布、权限与证据链路，不维护权限薄弱的旁路回答表。

## 5. Capability 与 Provider 契约

| Capability | 核心接口 | 默认 Provider／后续替换 |
| --- | --- | --- |
| Identity | authenticate、resolve_principal、validate_delegation | OIDC／本地 Keycloak；企业 IdP |
| Authorization | require、authorized_documents、check_delivery | PostgreSQL 当前授权；后续策略引擎适配 |
| KnowledgeCatalog | create_asset、revise、set_policy、archive | PostgreSQL Repository |
| SourceAccess | discover_changes、fetch_snapshot、checkpoint | 受控 URL 集合；后续企业连接器 |
| BlobStorage | put_immutable、get、delete、verify | S3 兼容对象存储；测试内存替身 |
| Parsing | parse → ParseArtifact | 各格式适配器；后续 OCR／外部解析 |
| Chunking | chunk → ChunkSet | 标题／段落／表格感知；策略可替换 |
| Publication | submit_review、approve、publish、withdraw | 本地领域规则与 PostgreSQL 事务 |
| Embedding | embed_documents、embed_query | 独立真实模型；声明用途、维度和距离 |
| Indexing | build、validate、publish、retire | PostgreSQL／pgvector；OpenSearch／Milvus |
| Retrieval | lexical_search、vector_search、fuse、rerank | 全文＋向量＋RRF；重排按收益启用 |
| Evidence | pack、validate、check_dependencies | 授权、去重、预算、版本与快照校验 |
| Answer | generate → structured_draft | 配置的真实模型；无工具或文件权限 |
| Delivery | finalize、read_answer、read_citation | 当前权限与有效发布检查，失败关闭 |
| Quality | submit_feedback、evaluate、compare | 固定集与人工标注；评测工具可适配 |
| Operations | reserve_quota、record_usage、emit_audit | 数据库配额／用量／Outbox；外部观测出口 |

契约必须说明输入输出、能力声明、Scope、超时、取消、可重试错误、资源释放和实现版本。必要过滤不支持时明确失败，不静默放宽权限。测试替身必须标识，不能把 Mock 通过当真实集成通过。

## 6. 身份、授权与服务委托

### 6.1 登录和主体

浏览器使用 OIDC Authorization Code＋PKCE。默认采用 API 侧会话／BFF 方式：Token 留在服务端，浏览器使用 HttpOnly、Secure、SameSite 会话 Cookie，状态变更接口增加 CSRF 防护；开发回环例外显式配置。具体客户端注册、退出和会话期限在 V-02 固定。

验证签名、算法允许名单、issuer、audience、有效期和适用客户端；支持 JWKS 轮换。使用 issuer＋subject 映射本地主体，邮箱只作可变属性。IdP 故障时不回退为匿名管理员。

组成员变化通过受控同步或管理接口落库，记录同步延迟；本地撤权即时生效。来自 IdP 的撤权生效上限由会话检查及同步机制决定，部署前必须明确，不能声称任意远端变化实时可知。

### 6.2 资源授权

资源读取须同时满足 Tenant、Workspace、知识库和文档规则，以及当前发布／有效期约束。普通成员文档 ACL 只收窄；Owner 可读本库内容但不自动读取他人问答。平台管理员没有隐含知识超级读权限。

Repository 查询显式带 Scope，复合外键／唯一约束阻止跨域关联；RLS 可作为后续纵深防御，不代替用例授权。搜索候选、计数、自动补全、缓存键、导出和后台任务均使用相同规则。

ACL、成员或治理变化更新相应 policy_epoch。缓存以身份、范围、策略代次和数据版本分区，敏感正文默认不进入共享缓存；交付始终读取当前授权。Outbox 驱动清理派生缓存，但消息延迟不延迟权威拒绝。

### 6.3 服务账户与委托

服务身份访问自身明确授权的资源；代表用户调用需可验证、短期、受众限定的委托。有效权限取服务允许动作、用户当前资源权限和委托范围的交集。不接受任意 user_id／X-Project-ID 作为身份。

独立服务 API 在 V1 验证服务身份与委托。委托协议（签发方、受众、claims、权限映射）属 V-07，须在 S5 实现服务委托前确定；S1 的 OpenAPI 只预留凭据位置与声明形态，不实现签发。平台侧适配在 S7 接入。没有用户委托时不能自动继承某管理员权限。

## 7. 来源同步与知识摄取

### 7.1 可靠输入

上传／同步 → 校验身份、目标与配额 → 写不可变对象 → 短事务建立修订意图、任务及 Outbox → 返回 `202 + job_id`。

幂等键限定 Scope、资源和动作，绑定规范化请求指纹；相同键相同输入返回同一任务，不同输入返回冲突。更新必须显式指定文档及预期修订，同名文件不隐式合并。去重对象不共享权限。

对象上传与数据库不能假设共享事务。未入库对象通过受控孤儿回收处理；任务只处理已确认对象及校验和。URL 首次抓取保存快照，后续解析重试复用快照，显式同步才产生新抓取。

### 7.2 Worker 链路

领取任务及租约 → 抓取／校验快照 → 隔离解析 → 清洗 → 位置映射 → 分块 → 全文表示／向量 → 构建候选索引 → 等待审批及发布条件 → 发布。

Job 使用 queued／running／succeeded／failed／cancelled，Stage 独立记录 fetch／parse／clean／chunk／embed／index／publish。Worker 心跳续租，提交携带 fencing token；失去租约的旧 Worker 不得提交。重试有上限、退避和人工处理队列，取消只在安全边界生效。

外部模型可能重复调用，记录 attempt 与费用；预算预留、对账及告警控制重复费用，不声称恰好一次。

### 7.3 增量连接器

首版 URL 集合连接器支持来源配置、定时／手动同步、HTTP 条件请求和内容指纹；不支持条件请求的来源抓取后比较指纹。游标在发现事件与任务持久化后推进，而不是在发现前推进；任务失败仍可独立恢复。

每次同步产生安全摘要：发现、新增、更新、未变、待确认失效、失败。403、超时、单次 404 与明确删除分开；未确认失效按库策略告警或暂停交付。显式从受管集合移除来源产生 tombstone。

后续企业连接器必须声明是否支持增量、删除、源 ACL、权限主体映射和检查点。源授权无法可靠映射时禁止自动发布，不将服务账户可见范围自动分享给全库。

### 7.4 解析和人工修正

各格式 Provider 输出结构化段落、标题、表格和位置：Markdown 行号、PDF 页码、DOCX 段落、XLSX 工作表与单元格、PPTX 页序、网页快照与段落。保留原始定位与派生映射；表格分块携带列头。

人工修正生成 ParseArtifact 新版本，记录作者、原因和原文关联，不改 Blob。新产物触发片段和索引重建，必要时重审。扫描 PDF 明确返回需要 OCR；OCR 属于后续能力，不把空解析标为成功。

## 8. 审核、原子发布与派生一致性

发布前检查：内容与解析产物已获当前策略批准、索引构建完成、任务租约有效、资产未删除或撤回、有效期允许、预期修订未冲突。

发布短事务锁定知识库，基于最新 Manifest 合并本次文档变更，使用 CAS 检查文档更新序号，切换 Publication 与活动索引指针，并提交任务成功和 Outbox。不同文档并发更新不能以旧全库清单相互覆盖；同文档慢任务不能覆盖较新更新。

新索引发布前旧批准版本继续服务；更新失败不破坏旧版。紧急撤回、删除、归档和到期由权威状态直接阻止交付，不等待重建。过期检查使用时间条件，定时任务只负责整理状态和索引，不能作为唯一安全控制。

同库 Manifest 只能引用适配该索引模型和策略的派生产物。更换模型时构建整套兼容候选集合，不把不同模型向量混进同一空间。

Outbox 与业务修改同事务，消费者按事件 ID 幂等处理，按资源序号防止乱序恢复旧状态。外部搜索后端将来采用准备索引、校验覆盖、切换路由模式；失败候选不能成为活动索引。

## 9. 授权检索与 EvidenceSet

### 9.1 调用链

验证主体与查询范围 → 固定 QueryPlan → 获取各库已发布快照 → 构造有权且有效文档集合 → 词法／向量召回 → 去重与 RRF → 可选重排 → 证据预算 → 外发检查 → EvidenceSet。

跨库检索使用 `knowledge_base_id → index_version` 映射，不要求多个库共享一个索引 ID。EvidenceItem 分别绑定来源库、发布对象、修订、解析产物、ChunkSet、片段和位置；EvidenceSet 绑定同一 Scope 和 QueryPlan。

首版跨库限于同 Workspace，选中库必须全部有权。隐藏库不出现在选择器和计数中；请求不可见资源返回不泄漏存在性的统一错误。

### 9.2 检索正确性

- 授权、有效期、删除和元数据筛选作用于每路检索的 Top-K 之前，不先全库截断再过滤。
- pgvector 首先用精确检索建立过滤基线；ANN 启用前单独评估高过滤比例、候选不足和召回损失。
- 中文分词、术语词表、归一化及查询策略版本化，固定样本比较；全文排名与向量距离不直接相加。
- RRF 使用通道排名，跨库也采用明确的候选预算和排名融合策略，不比较不同模型空间的原始距离。
- 每个向量通道用其索引绑定的模型生成查询向量；模型不可用时明确失败，不用新模型查询旧向量。
- 混合检索或跨库默认不静默部分成功。降级必须由调用方明确允许，响应列出实际通道／范围与 warning，评测单独统计。
- 重排是可替换增强；未经质量收益和延迟评测不默认开启。

### 9.3 证据规则

证据必须非空、当前授权、属于指定快照，且来源关系可被 Manifest 验证。同片段去重，不同内容冒用同 ID 报错。扩展父段或邻接片段时再次授权并计入预算。

按实际目标模型 tokenizer 计算或估算上下文预算，预留提示、问题和回答空间；近似估算须标明单位。全部送入模型的内容都进入 EvidenceSet，包括 FAQ、标题和补充上下文，避免只有引用段落被审计。

向外部 Embedding、Rerank、Answer Provider 发送数据前检查外发策略；后台摄取还要验证任务的当前授权／服务身份。Prompt Injection 不能获得任意 SQL、工具或文件执行权。

API 返回的每个 EvidenceItem 带 `untrusted_content: true`，契约明确调用方必须把证据当数据而不是指令。平台 Agent 把证据交给自己的模型时同样适用，列为 15.1 的接入约束。

## 10. 回答、引用与受控交付

EvidenceSet → Answer Provider 结构化草稿 → 引用结构校验 → 当前权限与有效发布重查 → 保存 Answer／Evidence／Citation → 最终交付。

状态统一为 answered／no_answer／clarification_needed／failed。空证据直接无答案；相关但不足允许拒答；范围或业务条件不明确可以请求澄清；索引、模型和引用故障均为失败，不伪装成无答案。

结构校验检查引用 ID、证据绑定、修订、位置和指纹，失败草稿不展示。结构合法不能证明断言得到语义支持，固定评测另行验证。第一版流式仅用于进度事件，完整正文通过校验后发送。

### 10.1 历史安全依赖

交付检查全部 EvidenceSet，而非只检查可见 Citation。任一送入模型的内容撤权、删除、过期、被撤回或修订失效，整条答案与引用不可交付。问答记录还须检查提问者、原查询库范围及共享许可，避免列表摘要泄漏。

与答案无关的文档更新不使答案失效。原文未变、只有索引重建时，保存的证据快照仍有效且当前批准策略允许才可回读；重新 Top-K 搜索不是验证历史引用的方式。

“证据快照”指保存的引用与指纹（文档、修订、解析产物、片段、位置、指纹），不是正文副本；回读时按引用从权威存储重取并重新授权。引用目标因保留策略被物理清理时，答案转为不可交付。管理员纠正错误解析时，可显式撤销旧 ParseArtifact 的交付资格。

授权与读取形成短事务一致视图，必要时对 policy_epoch 再校验；保证的是交付授权决策时刻，不保证撤回已发出的网络字节。对象下载通过鉴权代理，严格撤权场景不使用长效预签名链接绕过服务。

保存后至实际响应前发生政策变化时仍需最后交付检查；接口不返回未检验的数据库缓存。取消或超时的生成保留安全状态，不留下可读草稿。

## 11. API 与前端边界

API 使用 `/v1`，正式 OpenAPI 在 S1 落地，分页、稳定排序、时间格式、请求大小和错误契约纳入测试。以下为资源设计，不声称已实现。

契约规则：

- 兼容性：`/v1` 内只增加可选字段和新端点，不删除或改变既有字段语义；新增枚举值须声明调用方应容错；弃用先公告并保留不少于一个版本周期的时间窗，由契约测试保护。
- 幂等：凡创建任务或产生模型费用的请求（`/imports`、`/sources/{id}/sync`、`/answers`）都接受幂等键，语义同 7.1，并在额度预留前去重，避免重试重复计费。
- 关联：请求可携带调用方的 `correlation_id`（平台填 run_id 或 run_step_id），原样写入 AuditEvent 和响应，不参与授权；服务自身 request_id 仍独立生成。
- 证据：EvidenceItem 带 `untrusted_content` 标记，见 9.3。

| 资源／用例 | 接口方向 | 约束 |
| --- | --- | --- |
| 身份与空间 | `/workspaces`、`/memberships`、`/groups` | Scope 来自已验证身份，不信任请求自报 |
| 知识库与目录 | `/knowledge-bases`、`/folders` | 角色与动作授权；修改使用预期版本 |
| 文档／页面／FAQ | `/documents`、`/documents/{id}/revisions` | typed payload，内容不可变修订 |
| 导入与同步 | `/imports`、`/sources`、`/sources/{id}/sync` | 202＋任务，幂等键与配额 |
| 解析与修正 | `/documents/{id}/artifacts`、`/artifacts/{id}/corrections` | 新产物，不原位覆盖 |
| 审核与发布 | `/reviews`、`/publications`、`/documents/{id}/withdraw` | 绑定修订与产物，审计和 CAS |
| 搜索／问答 | `/search`、`/answers` | 选库列表、过滤、策略；授权跨库 |
| 历史／引用 | `/answers/{id}`、`/answers/{id}/citations` | 全证据与当前授权检查 |
| 反馈／评测 | `/feedback`、`/evaluations` | 共享许可和数据集授权 |
| 运行治理 | `/jobs`、`/usage`、`/audit-events`、`/model-configs` | 单独管理权限，不返回密钥 |

错误结构为 `error.code / error.message / request_id / retryable`，不返回原始堆栈。401 身份缺失；403 已知资源的操作禁止；404 不存在或不可见；409 幂等／版本冲突；413 超限；422 输入非法；429 配额／限流；502／503 上游故障／不可用。超时使用明确错误码与约定状态。

同步回答中 no_answer／clarification_needed 为业务结果，不用错误码伪装。模型失败返回非 2xx 和安全关联标识；异步任务的资源状态与 HTTP 查询成功分开解释。

前端以门户、工作台、控制台分域，服务器状态交给查询层管理。权限变化、空间切换、退出和失效通知清理敏感缓存；重新验证期间隐藏正文，不使用 stale 数据回退。批量动作逐项反馈，不能暗示跨文件原子成功。

## 12. 质量运营、可观察性与成本

反馈默认存分类和关联 ID；用户明确允许后才共享正文，处理者仍需资料权限。质量问题关联资产、策略、修复版本和回归结果，不把员工问答直接复制进公共评测集。

评测固定数据、标签、模型、策略、索引及授权上下文，分别报告解析保真、Recall@K、证据覆盖、引用结构、语义支持、拒答、延迟和费用。先比较全文／向量／混合，再测试重排或图扩展。

Trace 用 request_id、job_id、answer_id、evaluation_id 及调用方 correlation_id 关联，日志只记录数量、状态、版本、耗时和安全错误。指标覆盖任务积压、租约超时、来源新鲜度、检索空命中、模型失败、交付拒绝、预算和备份结果。指标标签避免携带问题正文及高敏感资源名称。

配额在提交时预留、完成后结算，失败／取消释放未用额度；外部成本超时不确定时记录待对账，不无限重试。组织、空间和应用限流分别生效，防止单一连接器或用户耗尽 Worker。

## 13. 安全、保留与恢复

### 13.1 不可信输入

- 抓取限制协议、端口、公开地址、每次 DNS 和重定向目标，防 DNS 重绑定；连接到已验证地址并验证目标主机，不只做字符串黑名单。
- 上传限制文件大小、解压大小、条目数量、页数、内存、CPU 和时间，禁止宏／脚本；解析进程限制网络与文件访问范围。
- 原文 HTML／Markdown 净化，危险链接不执行；下载以服务端对象 ID 定位，不接受磁盘路径。
- 模型端点配置与密钥使用属管理权限，低权限用户不能把已存凭据重定向到任意地址。
- 普通技术日志不含问题、原文、Token 或密钥；审计出口的权限与保留策略独立。

### 13.2 保留与清理

保留策略分别覆盖 Blob、解析／分块、向量、历史、评测、审计和备份。删除先写 tombstone 与策略代次，立即阻止交付，再异步清理派生数据。清理前检查当前索引、活动任务、保留义务和受控历史依赖。历史答案只保存证据引用，片段被物理清理后答案不可交付，因此保留策略要单独定义历史答案与其证据的保留期，二者不得错配。

法律保留等特殊策略属于部署治理扩展，不能默认为所有企业已具备。备份中的数据按期限清理；产品不承诺删除一条记录即从所有历史备份即时抹除。

### 13.3 备份恢复

数据库与对象存储各自备份，并保存可对齐的恢复清单、对象版本和校验和。试点可在受控暂停写入窗口创建一致备份；生产连续恢复需设计数据库日志与对象版本保留的协调策略。

恢复顺序：隔离环境 → 恢复数据库／对象 → 检查引用完整性与 tombstone／授权状态 → 重建必要派生索引 → 验证身份、读取、删除和回读 → 开放流量。

恢复旧备份可能丢失备份后的撤权／删除记录，必须从独立保留的治理日志或经确认的现态补齐；无法证明安全时维持停服，不直接开放旧权限。RPO／RTO 必须通过演练测量，指标起点见 PRD 9.3。

## 14. 部署、升级与索引迁移

本地企业联调 Compose 包含 Web／API、Worker、PostgreSQL＋pgvector、对象存储、Keycloak；服务默认绑定受控本机网络，密钥运行时注入，数据库与对象存储不暴露公网。

迁移为显式一次性作业，API／Worker 不并发改表。健康分存活、就绪和能力诊断：就绪检查数据库、迁移与必要依赖；模型连接不靠高频付费请求探活。

升级采用兼容性检查与 expand／migrate／contract 原则，先验证备份和回滚边界；不能保证不可逆数据迁移可直接回滚二进制。Worker draining 停止领取新任务，已有任务完成或安全释放租约。

索引后端或模型迁移：固定目标模型／策略 → 从权威内容构建 → 追平更新、删除、撤权 → 验证质量、覆盖、权限和延迟 → 原子切换路由 → 有限回滚窗口 → 清理旧索引。回滚仍受当前权限与 tombstone 约束。

部署扩展顺序由实测决定：先调整 API／Worker 并发与资源隔离，再评估数据库／对象存储高可用，最后按检索压力引入独立搜索后端。模块独立不要求独立服务。

## 15. 与平台集成及后续演进

平台通过 RemoteKnowledgeProvider 调用稳定 API，传递受众正确的服务身份／用户委托，显式映射 project 与 Workspace。返回 answer_id、EvidenceSet 标识、引用及关联 ID；平台在自己的 RunStep 记录调用，不把知识任务改名成平台 Run。

平台交付需同时满足知识授权和会话披露范围。历史消息应通过受控回读或失效状态展示，不能将答案复制进永久可见缓存。先验证一条“检索／回答／撤权／回读”链路，再扩展 Bot 和工作流入口。

### 15.1 接入模式与平台契约差异

两种接入模式的保证强度不同，契约和文档必须分开命名：

- 服务端回答：平台调用 `/answers`，由服务生成、校验并交付，全证据检查完整有效。首条链路采用此模式。
- 平台端生成：平台只调 `/search`，用自己的模型生成。证据离开服务边界后，全证据检查只覆盖服务存档的 EvidenceSet；平台须保存 `evidence_set_id`，展示前回读校验，并接受已交付内容无法撤回。

平台现有 `knowledge_qa` 契约（`OrchestrationPlatform` 提交 `c7e1223`）与本服务的差异，S7 前核对，由平台侧适配，不让服务迁就：

| 平台契约现状 | 本服务 | 处理 |
| --- | --- | --- |
| 作用域为 tenant_id＋project_id，EvidenceSet 绑定单个 knowledge_base_id 与 index_version_id | Tenant／Workspace／KB，跨库为 KB→index_version 映射 | project 显式映射 Workspace；首条链路限单库，跨库需先扩展平台契约 |
| 答案状态含 `retrieval_degraded` | 四种状态，默认不静默降级 | 仅调用方显式允许降级时，适配器才映射为 `retrieval_degraded` |
| `StoredKnowledgeAnswer` 持久化 `RetrievalResult.snippet`，且要求 `run_id` | 证据只存引用，回读重新授权 | 远程 Provider 模式下平台不持久化服务证据正文，只存 answer_id、evidence_set_id、引用；`run_id` 作为 correlation_id 传入 |
| `KnowledgeAnswerStore` 只有 save／get，无交付重查与失效状态 | 回读必须重查权限与证据 | 平台为知识消息新增回读校验和“已失效”展示状态 |
| `KnowledgeQuestion.actor_subject_id` 由平台自报 | 不接受自报身份 | 平台经 V-07 的委托协议传递，服务不信任该字段 |
| IngestionJob 阶段与状态集合不同 | 服务独立维护任务模型 | 平台不镜像服务任务，只引用 job_id |

接入约束：平台把证据交给自己的模型时，必须按 `untrusted_content` 当数据处理，不得作为指令拼接。

Wiki 链接、Entity／Relation／Claim／Ontology 未来作为有来源和版本依赖的派生产物加入，图扩展候选仍要授权并进入 EvidenceSet。人工页面与 FAQ 已属于 V1，不需要等待 GraphRAG。LangChain、LangGraph 等只在存在明确组件收益时通过 Provider 适配，不成为公共模型或治理事实来源。

## 16. 验证策略

1. 领域／契约测试：Scope、版本、状态、幂等、错误、引用与全证据依赖。
2. 真实基础设施：PostgreSQL 迁移／事务／并发、对象存储一致性、OIDC 登录／轮换／会话。
3. 故障测试：租约过期、重复任务、并发发布、发布中删除、模型超时、索引不完整、Outbox 重放。
4. 安全矩阵：身份、角色、用户组、各资源端点、服务委托、SSRF、解析隔离及提示注入。
5. 质量评测：固定集比较策略和真实模型，不用 Mock 或结构合法率代替效果。
6. 产品端到端：导入、预览、审核、发布、搜索、问答、引用、反馈修复、撤权及恢复。

验收结果引用 PRD 的 FR 与实施计划的门禁。前端常规修改以开发编译、必要 lint 和相关测试为准，不默认生产构建；浏览器验收优先复用已有页面。模型评测需预先确认外发范围与预算。
