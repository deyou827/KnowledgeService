# ADR-0001 S1 技术基线与对象存储选型

状态：已采纳（2026-10-15）；2026-10-16 调整对象存储决策，按参考项目固定发布版采用 MinIO 兼容维护分支。集成基线已选定，不代表生产安全放行。

## 结论

S1 采用 Python + FastAPI + PostgreSQL，默认实现与版本固定规则见 [ADR-0002](0002-reference-backed-defaults.md)，精确依赖以本项目锁文件为准。对象存储沿用 RAGFlow v0.27.2 的固定部署配置，采用 `docker.io/pgsty/silo:RELEASE.2026-08-06T00-00-00Z` 作为 S1b 集成基线；这是 PGSTY 维护的 MinIO 分支，不是 MinIO 官方发行版。不再前置开展 SeaweedFS 等产品的对比选型，保留正常接入验收和生产安全门禁。

## 背景

PRD 的 V-01、V-02 要求在 S1 前确定试点范围、对象存储、OIDC 会话方案和依赖版本。

## 决策

### V-01 试点范围

- 单租户试点，容量档位采用 PRD 9.3：100 个已注册用户、10 个并发搜索、10,000 份文档／100,000 个片段。该档位用于测试设计，不是容量承诺。
- 测试夹具仍使用至少两个租户、每租户两个空间，验证隔离。
- 角色、知识类型、审核和有效期策略沿用 PRD 第 3、7 节默认值；试点组织的真实角色清单待用户提供，不阻塞 S1a。

### V-02 依赖版本

采用 [ADR-0002 第 3 节](0002-reference-backed-defaults.md#3-s1-基础默认实现) 的默认组件和参考版本规则。撤销原“查询时最新版”表，不将各包独立最新版或“发布满两周”当作稳定组合依据。

S1a 依据固定参考版本、兼容性与安全修复生成本项目锁文件；pytest、import-linter 等开发工具一并锁定。S1b 固定基础设施镜像 digest。产品／组件选择已经确定，锁文件与真实接入验收是工程交付，不能伪称本轮文档修改已经完成。

### 会话方案

沿用技术方案 6.1：OIDC Authorization Code + PKCE，API 侧会话（BFF），浏览器使用 HttpOnly、Secure、SameSite Cookie，状态变更接口加 CSRF 防护。服务端会话存储、默认期限与退出行为见 ADR-0002 第 3.2 节。S1a 用测试签发 JWT 夹具，S1b 接入本地 Keycloak。

### 对象存储

决策原则：以参考开源项目的正式发布配置为主要依据，优先稳定够用和维护简单，不为替换而替换。采用固定版本，不使用 `latest`、nightly 或 RC；不把“大厂项目采用”表述成相同负载下的稳定性证明。

2026-10-16 核对到的部署配置：

| 参考版本 | 实际镜像 | 本项目取舍 |
| --- | --- | --- |
| WeKnora v0.8.2 | `quay.io/minio/minio:RELEASE.2025-09-07T16-13-09Z` | 历史官方发行版参照，不作为新项目默认基线 |
| WeKnora v0.8.3 | `pgsty/minio:RELEASE.2026-08-04T00-00-00Z` | 已转向 PGSTY 维护分支，但仍使用更名前镜像 |
| RAGFlow v0.27.2 | `pgsty/silo:RELEASE.2026-08-06T00-00-00Z` | 采用其固定发布配置作为集成基线 |

RAGFlow 参考提交为 `a024bea0cd93f39e6652a42bf84dd20c55bc560b`，文件为 `docker/docker-compose-base.yml`；WeKnora v0.8.2 参考提交为 `3e8b0bfc80b845b2d4b2ed683994748741450a97`，文件为 `docker-compose.yml`。服务仍叫 `minio` 不代表镜像仍由 MinIO 官方维护；PGSTY 项目于 2026-08-06 更名为 Silo，应区分上游项目、维护分支和镜像发布者。

部署实施时将所选镜像解析为 digest 并固定，记录平台架构；本文未拉取镜像，不声称镜像可用性、digest 或运行效果已验证。不自动追随参考项目升级，也不凭 GitHub 的 latest 标记选用名称包含 RC 的版本。

S1b 只做正常集成验收：不可变修订写入、校验和、鉴权代理读取、失败重试、重启保留、删除与恢复清单对齐。这验证本项目是否接对、配对，不另立存储产品评测阶段。业务内容版本由 DocumentRevision 与不可变对象映射定义，不与存储原生版本号混为一谈。下载仍走知识服务鉴权代理，不照搬参考项目的长效预签名链接或默认凭据。

维护与安全边界：MinIO 官方社区仓库已声明不再维护；PGSTY Silo 是独立维护分支，不受 MinIO 官方背书。核对时 Silo 的 README／CHANGELOG 明确说明部分修复尚未进入已发布 Server，其中 SN-2026-011 涉及未签名请求头改变已签名 PUT 的语义，最新公开 Server 也受影响。因此本次选定的是参考发布版集成基线，不是无漏洞或生产安全保证。S1b 使用隔离网络和合成资料，不向客户端分发存储凭据或预签名写入链接；这些措施不等同于漏洞修复。正式企业数据／生产接入前必须确认适用安全公告的修复发行版或经确认的处置方案，并更新版本与 digest，不能因参考项目采用而跳过。AGPLv3 许可适用性也需在正式交付前确认。

对象存储通过 `BlobStorage` Capability 隔离（技术方案第 5 节），测试用内存替身必须标注，不替代真实链路验收。

## 放弃的备选

- SeaweedFS、Garage、RustFS：暂不引入，也不开展前置对比；仅在默认实现出现明确不可接受缺口时重新评估，不以未经证实的“成熟度最高”或能力缺失直接定案。
- 无维护承诺的 MinIO 官方历史发行版：保留参考，不作为新项目默认基线。
- 本地文件系统：只作测试替身，不满足企业验收的对象存储链路。
- 先写代码再选存储：对象存储决定不可变写入和恢复清单设计，晚选会推翻 S1b。

## 来源

- [RAGFlow 固定提交部署配置](https://github.com/infiniflow/ragflow/blob/a024bea0cd93f39e6652a42bf84dd20c55bc560b/docker/docker-compose-base.yml)
- [WeKnora v0.8.2 固定提交部署配置](https://github.com/Tencent/WeKnora/blob/3e8b0bfc80b845b2d4b2ed683994748741450a97/docker-compose.yml)
- [WeKnora v0.8.3 部署配置](https://github.com/Tencent/WeKnora/blob/v0.8.3/docker-compose.yml)
- [PGSTY Silo 2026-08-06 发布说明](https://github.com/pgsty/silo/releases/tag/RELEASE.2026-08-06T00-00-00Z)
- [PGSTY Silo 维护与发行边界（固定提交）](https://github.com/pgsty/silo/blob/00a375c4e22bfb1e9e684b0bd01ecd972f4448d0/README.md)
- [PGSTY Silo 未发布修复（固定提交）](https://github.com/pgsty/silo/blob/00a375c4e22bfb1e9e684b0bd01ecd972f4448d0/CHANGELOG.md)
- [MinIO 官方维护状态](https://github.com/minio/minio)
- 服务依赖与解析组件的固定参考来源见 ADR-0002。

对象存储来源于各项目官方仓库，核对日期为 2026-10-16；发布配置证明该项目采用了具体版本，不证明本项目完成部署、性能、安全或恢复验收。
