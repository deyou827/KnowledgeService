# ADR-0002 开源参考驱动的默认实现与阶段输入

状态：已采纳（2026-10-16）。本记录确定默认实现与配置原则，不表示代码、依赖锁文件或运行验收已完成。

2026-10-20 修订：按 [ADR-0004](0004-rag-component-architecture.md) 采用 LlamaIndex 摄取转换、LangChain 模型适配与 Ragas 评测；真实 Embedding／混合检索提前至 S3，原有身份、存储、安全与恢复规则不变。

## 1. 结论

以固定发布版的开源实现为主要依据，按模块采用成熟组件，不再将解析、分词、会话、任务和部署方式作为无边界选型任务。参考项目未覆盖或不符合现有架构的部分，给出明确的本项目默认值，不冒称来自大厂生产实践。

三个状态分开记录：

- **技术选择已定**：默认 Provider、库、接口和参数有明确答案；出现具体不兼容或不可接受缺口时才重新决策。
- **接入验收待完成**：生成锁文件、固定镜像 digest、契约测试和恢复测试属于开发交付，不等于重新选型。
- **业务确认待提供**：实际身份环境、数据外发许可、模型资源与预算、正式保留期等不能由参考项目代替授权。

本阶段只收敛方案，位置是 KnowledgeService 的实现基线，不改变平台 Runtime。平台仍通过 RemoteKnowledgeProvider 和服务契约接入；不新增基础设施服务，不启动部署或真实模型调用。

## 2. 参考版本与证据边界

| 主要参考 | 固定版本／提交 | 本次实际检查的内容 | 使用边界 |
| --- | --- | --- | --- |
| PAI-RAG | v0.4.3；`fb06fc594692ad02afaf65b5cee96eb1f0a4b7d0` | `pyproject.toml`、`poetry.lock` | Python 服务与 LlamaIndex 依赖参考；仅采用摄取转换，不整体引入其业务模型和检索后端 |
| WeKnora | v0.8.2；`3e8b0bfc80b845b2d4b2ed683994748741450a97` | `docreader/pyproject.toml`、`docreader/uv.lock`、PDF／DOCX／HTML／Markdown／Excel 解析文件 | 按格式处理和原文定位；不复制 Go 主服务、OCR 调用和库内部 monkey patch |
| RAGFlow | v0.27.2；`a024bea0cd93f39e6652a42bf84dd20c55bc560b` | 部署配置、`pyproject.toml`、DOCX／Excel／PPT 解析文件、`rag/nlp/rag_tokenizer.py` | 对象存储沿用 ADR-0001；Office 库选型可借鉴，不搬入完整 DeepDoc／Infinity 运行时 |

证据只证明对应源码或依赖存在，不证明本项目的安全、性能、兼容性和 UI 已验收。尤其 RAGFlow 该版 tokenizer 委托给 `infinity.rag_tokenizer`，不能将它描述为本项目将直接采用的 jieba 实现。

## 3. S1 基础默认实现

| 项目 | 默认选择 | 依据与取舍 |
| --- | --- | --- |
| Python | 3.12 系列 | 位于 PAI-RAG v0.4.3 声明的 `>=3.11,<3.13` 范围内；不因 RAGFlow 使用 3.13 而迁移整套运行环境 |
| API／模型 | FastAPI＋Pydantic 2 | 参考 PAI-RAG 的服务依赖；保留本项目 DTO 与领域对象分离 |
| 数据访问 | SQLAlchemy 2.0 系列＋psycopg 3＋Alembic | SQLAlchemy／Alembic 参考 PAI-RAG；psycopg 3 是本项目已定方向，不谎称参考项目的 psycopg2 锁文件能直接复用 |
| 数据库／检索扩展 | PostgreSQL 17 系列＋pgvector 0.8 系列 | 本项目默认部署组合；只用 PostgreSQL 全文与向量能力，不增加参考项目中的 MySQL、Infinity、Elasticsearch 或 ParadeDB 依赖 |
| 身份 | Keycloak 26 系列；标准 OIDC | 保留 ADR-0001 方向；实际企业 IdP 可替换，不因参考项目使用其他身份系统而改变服务契约 |
| OIDC 客户端／JWT | Authlib 负责协议交互，PyJWT 负责服务令牌验证 | 本项目标准协议适配选择；禁止手写签名算法或关闭 issuer／audience 校验，不将浏览器 ID Token 混作通用服务令牌 |
| S3 客户端 | boto3，经 BlobStorage Provider 包装 | RAGFlow 依赖清单包含 boto3；领域层不暴露 SDK 响应、bucket 或服务端品牌 |
| 对象存储服务端 | ADR-0001 的固定集成镜像 | 不在本记录维护第二份镜像版本；安全放行限制继续有效 |
| 事务／任务 | PostgreSQL Unit of Work、任务表、Outbox，独立 Worker | 本项目一致性要求；同一用例共享事务，Provider 不自行提前提交；不引入 Celery／Redis 队列 |
| 观测 | 结构化日志＋OpenTelemetry 标准出口 | 先记录关联标识、耗时、错误码、队列积压与费用，不引入完整外部观测平台作为启动条件 |

### 3.1 版本如何固定

参考项目的锁文件是来源证据，不是本项目的完整安装组合。PAI-RAG v0.4.3 锁文件中的 FastAPI `0.115.2`、Pydantic `2.12.5`、SQLAlchemy `2.0.44`、Alembic `1.17.2` 和 Uvicorn `0.31.1` 仅记录为上游基线，不能未经安全与兼容处理原样宣布为本项目生产版本。

S1a 在上述技术范围内生成本项目依赖锁文件，记录所有精确版本与制品哈希；S1b 固定镜像补丁版本、平台架构与 digest。已知安全问题优先采用兼容修复版本；如需越过已定主／次版本边界，记录具体原因，而不是重新全面选型。禁止 `latest`、nightly、RC 和复制多个项目的整份锁文件。

本轮没有生成或解析本项目依赖锁文件，也没有固定 PostgreSQL／Keycloak 的补丁镜像，因此不能称“完整可部署版本组合已锁定”。这些是对应阶段的工程交付，不再列为等待用户选择产品的事项；不以“发布满两周”替代兼容性和安全判断。

### 3.2 会话默认配置

采用 Authorization Code＋PKCE（S256）、API 侧 BFF、PostgreSQL 服务端会话。浏览器只持有不含 Token 的随机会话标识；会话 Cookie 默认 HttpOnly、Secure、SameSite=Lax，状态变更加 CSRF 校验，OIDC 校验 state／nonce。禁止把访问令牌存入客户端可读会话或 localStorage。

本项目可配置初值：会话空闲 30 分钟、绝对期限 8 小时，不能超过身份提供方或组织更严格的期限。退出立即撤销本地会话，再尝试 IdP 退出；IdP 不可用不能阻止本地退出。认证成功不替代每次资源授权。开发回环地址的 Secure Cookie 例外须显式启用，不能带入正式部署。

## 4. S2／S4 解析与输入默认实现

S2 只实现 Markdown 核心链路；其他 P0 格式按 S4 分批接入，不因默认库已确定而提前全部实现。

默认以 LlamaIndex Core 包装解析结果的转换与切分，下列格式库继续负责实际解析。Application 通过项目契约控制各阶段，框架不拥有文档权威、模型配置、发布或独立向量库；转换输出回到项目 ParseArtifact／Chunk 模型，具体边界见 ADR-0004。

| 格式 | 默认组件 | 输出与边界 | 来源类型 |
| --- | --- | --- | --- |
| Markdown | markdown-it-py，保留 token 行范围 | 标题、段落、表格与原始行号；嵌入 HTML 不执行 | PAI-RAG 锁文件包含该库；用于行号定位是本项目适配 |
| DOCX | python-docx | 按正文顺序提取段落／表格和位置；不承诺 Word 页码，不执行宏或加载外部关系 | WeKnora／RAGFlow 解析代码 |
| XLSX | openpyxl，只读提取 | 工作表、单元格范围与表头；不执行公式或下载外链；公式无缓存值时明确标识，不伪造计算结果 | RAGFlow Excel 代码使用 openpyxl；定位约束由本项目补齐 |
| PPTX | python-pptx | 幻灯片序号、文本框／表格；不将截图理解当普通文本解析 | RAGFlow PPT 解析代码 |
| 文本 PDF | pypdfium2，隔离进程内按页提取 | 保留页码和位置；原生文本不足／混合扫描页面显式报告需 OCR，不将部分提取冒充完整成功 | WeKnora PDF 代码；不带入其 Go 侧 OCR 链路 |
| 受控公开网页 | HTTPX 抓取不可变快照，BeautifulSoup4＋lxml 清理与结构提取 | 保留快照、标题、段落序号；不执行 JavaScript，不自动跟踪站外链接 | WeKnora HTML 代码采用 BeautifulSoup／lxml；受控抓取由本项目实现 |

WeKnora 的 PDF 实现明确处理 PDFium 进程全局并发限制，本项目按已有高风险解析隔离原则每个解析进程串行调用，不在 API 内增加全局锁处理文档。DOCX 解析库内部 monkey patch 不照搬；只在确有失败样本时作窄适配并留回归用例。所有组件均输出本项目 ParseArtifact／位置模型，不泄漏参考项目 Document 类型。

WeKnora 锁文件中的 python-docx `1.2.0`、openpyxl `3.1.5`、python-pptx `1.0.2` 是组件版本参考，lxml 等解析依赖仍须随本项目锁文件落实适用安全修复，不因为参考项目锁过就忽略公告。

本项目开发初值（不是抄自参考项目的生产参数）：单文件 50 MiB、Office 容器累计解压 250 MiB／10,000 项、解析单任务 120 秒／1 GiB 内存；网页响应 10 MiB、请求总预算 30 秒、最多 3 次重定向。Worker 每实例先取 2 个任务，租约 120 秒、心跳 30 秒，失去租约禁止提交。受限解析必须能被终止，不只依赖业务层超时返回。

压缩检查还须限制压缩比、拒绝路径穿越和外部实体；普通 ZIP 上传不作为新格式开放。网页仅允许已批准 HTTP(S) 来源，每跳重新做地址与网络访问检查，禁止内网、回环、链路本地和云元数据目标；不能以放宽 SSRF 限制解决抓取失败。

## 5. S3／S4 检索与模型默认实现

按 ADR-0004 调整：S3 接入真实 Embedding、授权单库词法／向量／混合检索与真实回答，完成可配置 Rerank 接入和开关对照；S4 扩展格式与跨库。词法链路可先作为开发检查点，但不能单独代表 M0 混合检索验收。

- **中文词法**：jieba `0.42.1`＋受控业务词典作为默认分词 Provider；索引和查询使用同一归一化、词典版本与搜索分词规则，关闭 HMM 作为确定性初值。按分词序列构造 PostgreSQL `simple` 全文表示，查询参数化，不将用户输入直接当 tsquery 语法。jieba 来源是自身开源项目，不宣称来自上述参考项目的相同检索链。
- **向量与融合**：pgvector 精确检索＋词法检索＋RRF；每路先取 30 个有权候选，RRF 常数 60，融合后最多 10 条再按模型预算打包。以上是本项目可配置初值，不是已证明的召回／延迟最优值；ANN 和 Rerank 不默认开启。
- **分块**：标题／段落优先，表格重复必要表头；初始目标 512 tokens、重叠 64 tokens。以实际 Embedding tokenizer 和上下文限制为准，不能把字符数冒称 token 数；超长表格按行切分且保留位置，不截断后默默丢弃内容。
- **模型接口**：通过 LangChain 按需集成包实现 Answer／Embedding 两个独立 Provider；Rerank 使用合适集成或直接 API，特殊协议才增加直接 SDK／HTTPX 适配。首条回答链路用结构化草稿和完整正文交付，不开放工具执行。兼容接口不等于某个云厂商或默认允许外发；优先配置已有获授权模型服务，不下载大模型、不默认购买 API。
- **评测**：Ragas 经 Evaluation Provider 参与 S3 固定样本回归；确定性召回／引用／权限测试独立运行。记录评判模型和指标版本，调用也受外发和预算控制；不默认外接托管评测服务。
- **模型参数**：模型 ID、部署地址、维度、距离、tokenizer 与策略版本绑定索引；必要配置缺失时返回能力未配置，不回退到模拟向量或公开云。首次调用前确认模型环境、预算和数据外发；这是业务环境输入，不是 Provider 选型仍未定。
- **调用可靠性**：连接超时初值 5 秒，Embedding 请求总预算 60 秒，Answer 120 秒。采用 ADR-0003 第 5.5 节唯一恢复规则：只有确认未发送或 Provider 明确未接纳且允许重试，才最多重试 2 次，退避并遵循适用的 Retry-After，所有尝试共享用例截止时间及额度。429／5xx／超时本身不证明可安全重试；结果未知默认停止自动重新生成，已声明的上游幂等／查询恢复另按 Provider 契约执行。SDK 内部重试不得绕过总预算。鉴权、输入和引用校验失败不盲重试，费用未知独立记录，不保证外部调用恰好一次或自动对账成功。

这些初值在固定评测集上校准；不调整权限、引用或失败关闭规则以换取质量分数，也不以组件已有采用记录免除接口测试。

## 6. S4—S7 治理、运行与集成默认行为

| 事项 | 默认行为 | 正式环境需要的输入 |
| --- | --- | --- |
| 来源同步 | 公开 URL 白名单；每 24 小时同步，可手动触发；优先 ETag／Last-Modified，再比较内容指纹；同一来源不重叠运行 | 获授权来源清单和实际新鲜度要求 |
| 来源失效 | 超时／403／单次 404 记录异常，不当作删除；连续失败且超过 48 小时未成功确认时暂停该来源正文交付；恢复成功后仍需当前发布与权限检查 | 可配置更严格期限，不默认放宽 |
| 资料审核 | 开发夹具至少覆盖免审和强制审核两种库；正式新库默认人工审核、作者不得自审；获授权 Owner 可显式调整并审计 | 实际人员／角色映射；任何自动发布策略需业务确认 |
| 有效期 | 新资料默认无自动到期时间，但仍受撤回、来源失效和权限限制；支持显式有效期 | 组织要求的强制有效期，不把无期限当业务已同意 |
| 保留与清理 | 逻辑删除立即阻止交付；未配置并确认正式保留期前，不自动物理删除企业原文或证据依赖；开发仅使用合成资料 | 原文、历史、审计、备份分别的保留期；正式数据接入前必须确认，不以无限保留替代决策 |
| 备份与恢复 | 试点每日备份；短暂暂停新写入并排空相关写任务，生成数据库与对象对齐清单，校验后恢复；备份置于独立故障域 | 实际备份目标、加密／密钥责任、容量和服务窗口；RPO／RTO 仍以 PRD 目标及演练为准 |
| 服务委托 | S5 按 ADR-0003 同 realm 标准 Token Exchange 单跳 profile 接入；用户与认证客户端分开映射，权限取交集；不依赖 Preview／冒充 | 正式签发方／受众／客户端、claims 映射与撤销传播配置；不依赖平台旧身份实现 |
| 平台回读 | 首条链路采用服务端 Answer；平台只存服务标识与引用，展示前受控回读，失效答案不交付 | 平台侧按技术方案 15.1 实现适配，不改变服务领域模型 |

同步周期、超时和开发参数可在部署配置中修改并审计，不属于需要每次重开选型会的架构问题。生产安全修复、数据外发和组织保留期不能由开发默认值代替授权。

## 7. 阶段完成条件与剩余事项

V-01—V-07 编号仍以 PRD 为准，不新增第二套待办编号。默认实现确定后，剩余工作是：

1. S1a／S1b 形成精确依赖锁、镜像 digest、迁移和真实登录／对象存储链路；对象存储的安全门禁见 ADR-0001。
2. S2／S4 将已选组件封装为 Provider，补本项目位置映射、权限、资源限制和异常回归；不默认部署整套参考产品。
3. 模型首次调用前取得实际模型资源、数据外发许可和费用预算；正式数据接入前确认人员、保留与备份责任。
4. S5／S7 对接真实身份环境与平台；S6 执行容量、质量、恢复验收，不把设计数值写成已达到的性能承诺。

## 8. 官方来源

- [PAI-RAG 固定提交依赖声明](https://github.com/aigc-apps/PAI-RAG/blob/fb06fc594692ad02afaf65b5cee96eb1f0a4b7d0/pyproject.toml) 与 [锁文件](https://github.com/aigc-apps/PAI-RAG/blob/fb06fc594692ad02afaf65b5cee96eb1f0a4b7d0/poetry.lock)。
- [WeKnora 固定提交解析目录](https://github.com/Tencent/WeKnora/tree/3e8b0bfc80b845b2d4b2ed683994748741450a97/docreader/parser) 与 [锁文件](https://github.com/Tencent/WeKnora/blob/3e8b0bfc80b845b2d4b2ed683994748741450a97/docreader/uv.lock)。
- [RAGFlow 固定提交解析目录](https://github.com/infiniflow/ragflow/tree/a024bea0cd93f39e6652a42bf84dd20c55bc560b/deepdoc/parser)、[依赖声明](https://github.com/infiniflow/ragflow/blob/a024bea0cd93f39e6652a42bf84dd20c55bc560b/pyproject.toml) 与 [tokenizer 入口](https://github.com/infiniflow/ragflow/blob/a024bea0cd93f39e6652a42bf84dd20c55bc560b/rag/nlp/rag_tokenizer.py)。
- [jieba v0.42.1 官方说明](https://github.com/fxsjy/jieba/blob/v0.42.1/README.md)。

来源核对于 2026-10-16。本轮为有边界的源码与依赖抽查，不是整套产品的运行审计；引用实现与选择组件不授权复制许可证不兼容的代码。
