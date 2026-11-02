# Auto Ontology 中文试用指南

本文根据本仓库代码和 2026-11-02 的本机试用过程编写。命令从仓库根目录执行。
当前机器使用 Docker PostgreSQL 的宿主机端口 **5433**，避免与 Mac 已有的 PostgreSQL 5432 冲突。
文档不包含实际密钥、密码和 Azure 资源名；更换机器时请自行配置。

- [运行、配置与数据初始化](./运行与数据初始化.md)
- [Demo、验证与模型导出](./Demo与模型导出.md)
- [技术架构、本体与问数链路](./技术架构与问数链路.md)

## 项目做什么

Auto Ontology 在业务数据库和 AI Agent 之间构建目录与语义层，让自然语言问数有可检索的业务上下文。
它不是训练大模型的工具，也不是替代业务数据库的数据仓库。

主要流程是：连接数据源 → 读取目录与数据特征 → 构建语义定义 → 建立向量索引 → 理解问题 → 检索相关定义 → 生成和校验 SQL → 查询源数据库 → 返回结果。
查询仍访问连接的业务数据库；应用内部保存目录、语义定义、索引和应用状态。

| 层 | 内容 | 用途 |
| --- | --- | --- |
| 数据目录 | 数据库、Schema、表、字段、类型、主外键、样本值等 | 知道数据在哪里、如何关联 |
| 语义层 | Term、ColumnAttribute、SQL Attribute、语义关系、自定义分析 | 知道业务含义及计算口径 |
| 向量索引 | 目录和语义文本的 Embedding | 将“销售额”等问题词匹配到相关对象 |
| 问数流程 | 问题解析、候选检索、SQL 生成与校验、执行 | 将自然语言转为可核对的查询结果 |

“生成了 SQL”不等于“答案正确”：需要同时核对时间、金额定义、关联路径、粒度、数据源和结果。
认证或标记为 certified 也不能替代业务审核。

## 适合试用的场景

1. **自然语言问数**：电影租赁收入、国家排名、客户付款汇总；验证多表关联和日期条件。
2. **语义目录探索**：从业务术语查看字段和派生表达式，理解表之间的关联。
3. **统一指标定义**：把收入、金额、订单数等口径绑定到字段或 SQL，减少重复解释。
4. **模型迁移与审阅**：导出 YAML，审阅变更，再在隔离环境导入和验证。
5. **Agent 集成**：通过 MCP 提供只读语义发现和问数能力；写入走 Web `/api/...` 网关。

跨数据库目录可以共存，但不应据此承诺任意跨库 SQL 联邦查询。
当前范围过滤也不能被当成所有目录采集、采样、Embedding 环节均已覆盖的安全边界。

## 组件与数据库

| 组件 | 作用 | 本机访问 |
| --- | --- | --- |
| Next.js | 网页、登录、权限、API 网关 | http://localhost:3000 |
| FastAPI | 问数和目录、语义 API | http://localhost:3001 |
| Ingestion service | 数据采集、向量化、语义编译调度 | http://localhost:3002 |
| PostgreSQL + pgvector | 应用存储及向量存储 | localhost:5433（本次配置） |
| pgAdmin | 数据库管理工具 | http://localhost:5050 |

| 数据库 | 当前作用 | 来源 |
| --- | --- | --- |
| `auto_ontology` | 内部存储：`public` 目录及语义表、`frontend` 用户会话等表、`vdb` 向量表 | 应用迁移及运行时创建 |
| `pagila` | PostgreSQL 示例业务库：电影、租赁、客户、付款、国家等 | `seed_fixtures` |
| `chinook` | SQLite 示例业务库：音乐商店 | `dev_tools/fixtures/sql/chinook.sqlite` |

这些是测试数据，不是用户自己的企业业务数据。应用目录和业务数据分开存储。
本机 PostgreSQL（5432）与 Docker PostgreSQL（5433）是两个独立实例，账号也不共享。

## 两条试用路线

- **固定测试模型**：用仓库 fixtures 创建源库，再写入手工定义的语义层。可复现，适合演示和测试，但必须补建语义向量。
- **自动编译模型**：通过界面连接业务数据库，采集目录，再运行语义编译。需要可用的聊天与 Embedding 服务，应审核生成结果。

不要把 `seed_graph_fixture` 的手工语义层介绍成 LLM 自动发现的结果。

## 当前本地改动与限制

本次试用在原始 NVIDIA 默认配置之外增加了：

- 两个 fixture 入口自动加载根目录 `.env`，已导出的环境变量优先。
- Compose 的 `DOCKER_CONNECTION_STRINGS` 可覆盖容器内的数据源；只读挂载 fixtures SQL 目录至 `/fixtures`。
- `EMBED_API_STYLE=azure-cohere` 适配 Azure Cohere 的 `query/document` 参数。
- `EMBED_VECTOR_SIZE=1536` 适配本次 Azure `embed-v-4-0` 返回的维度；默认仍为 2048。

这些配置属于当前工作区的实现，不应假定上游所有版本都支持。
DeepSeek 聊天请求本次已能完成问题解析；客户端仍按模型名前缀选择，不能据此宣称所有 Azure 模型都完全兼容。
未配置专用 Rerank 时，重排流程仍需单独验证；注释配置不会关闭该功能。

## 参考入口

- [项目 README](../../README.md)、[官方 Helm 部署](../../DEPLOYMENT.md)
- [Setup skill](../../skills/nvidia-ontology-setup/SKILL.md)
- [Management skill](../../skills/nvidia-ontology-management/SKILL.md)
- [Query skill](../../skills/nvidia-ontology-query/SKILL.md)
- [REST API 规范](../openapi/auto-ontology-api.json)、[MCP 说明](../mcp.md)
