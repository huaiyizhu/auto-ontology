# Demo、验证与模型导出

前置步骤见[运行与数据初始化](./运行与数据初始化.md)。以下区分“固定测试模型已导入”和“完整 AI 问数已验证”。

## 1. 建议的 Demo 顺序

| 阶段 | 操作 | 验证点 |
| --- | --- | --- |
| 数据源 | 展示 pagila 和 chinook | 两个真实可连接的数据源，不是网页里的模拟结果 |
| 目录 | 展示 pagila.public 的 payment、customer、address、city、country | 字段类型、主外键与实际数据库对应 |
| 语义层 | 展示 Term、ColumnAttribute、SQL Attribute、自定义分析 | 业务概念绑定到具体表、字段或表达式 |
| 问数 | 输入指定数据源和历史日期的问题 | 生成 SQL、正确源库、日期过滤、关联和汇总 |
| 核对 | 在数据库客户端运行同一 SQL | 行数、金额、排名与界面一致 |
| 模型交换 | 导出原生 YAML，打开 data_layer / semantic_layer | 能追溯概念到物理字段，了解不是数据备份 |

建议问题逐步增加复杂度：

1. “在 pagila 中，一共有多少条付款记录？”
2. “在 pagila 中，2022 年各月的付款总额是多少？”
3. “在 pagila 中，2022 年 1 月至 7 月，按客户地址所属国家汇总付款金额，哪个国家最高？”
4. “在 chinook 中，按账单国家汇总发票金额，列出前五名。”

这些是验证问题，不承诺当前手工语义层可覆盖每一种问法。
Pagila 与 Chinook 的“国家”定义不同：前者此例为客户地址国家，后者可使用账单国家。
不能在未说明口径的情况下把两者当成同一个指标。

本次检查到 `pagila.public.payment` 有 599 条记录，日期为 2022 年 1 月至 7 月，总金额 2502.01。
这是本次样本状态，不是每个版本的固定断言；重新初始化或修改数据后应重查。
当前日期为 2026 年，“上个月”不应自动被改成数据中最近有记录的月份。

## 2. 用 SQL 建立结果基准

在 `pagila` 执行：

```sql
SELECT count(*) AS payments,
       min(payment_date) AS first_payment,
       max(payment_date) AS last_payment,
       sum(amount) AS total_amount
FROM public.payment;
```

国家付款金额排名基准：

```sql
SELECT co.country,
       sum(p.amount) AS payment_amount
FROM public.payment AS p
JOIN public.customer AS cu ON cu.customer_id = p.customer_id
JOIN public.address AS a ON a.address_id = cu.address_id
JOIN public.city AS ci ON ci.city_id = a.city_id
JOIN public.country AS co ON co.country_id = ci.country_id
WHERE p.payment_date >= TIMESTAMPTZ '2022-01-01 00:00:00+00'
  AND p.payment_date < TIMESTAMPTZ '2022-08-01 00:00:00+00'
GROUP BY co.country
ORDER BY payment_amount DESC, co.country
LIMIT 5;
```

这里“金额”是 `payment.amount` 的汇总，“国家”取客户地址，时间使用付款时间及 UTC 边界。
查询分区父表 `payment` 即可，不要再额外 UNION 所有 payment 子分区，否则可能重复计算。
如业务口径要求税前、退款后、租赁发生日期或其他币种，需要另行建模。

建议保留一次 Demo 的问题、生成 SQL、源库、执行时间、结果和手工核对记录。
本次只确认了索引检索和用户后续问数可运行，不应写成所有查询路径均通过测试。

## 3. 检查向量索引与就绪情况

```bash
docker compose exec -T postgres psql -U postgres -d auto_ontology -c \
  'SELECT database_name, label, count(*) FROM vdb.semantic_layer GROUP BY database_name, label ORDER BY 1, 2;'
```

若表尚未创建，说明索引写入尚未走到创建步骤，不是这个检查命令本身有问题。

本次补建后有 19 条语义向量，涉及 Term、ColumnAttribute、SqlAttribute、CustomAnalysis。
后端实际检索“销售额”时返回 14 个候选，包含 pagila / chinook 数据库作用域。
条数仅代表索引存在；仍需验证候选相关性、阈值、权限和最终 SQL。

使用 MCP 时先验证连接和 `check_readiness`；MCP 为只读接口。
API 状态读取走 Web 的 3000 端口，并使用登录会话或用户 API Token。
直接访问内部 FastAPI 不能当成对外权限验证流程。

## 4. 如何导出模型

在网页打开 **Settings → Import / Export**（路径 `/settings/import-export`）：

1. 在 Export Model 选择格式。
2. 选择要导出的数据库。
3. 下载 YAML；Ossie 若包含 ontology，则下载 ZIP。
4. 检查导出范围、表与字段、术语、SQL 和关联；必要时保留导出时间、版本和变更说明。

Web API 为 `POST /api/model/export`，需要 `modelInterchange:export` 权限。
请求体示例：

```json
{"databases": [], "format": "auto_ontology", "include_ontology": false}
```

`databases` 是目录数据库 **ID** 列表，不是 `pagila` 这样的名称；空列表表示全部数据库。
通过用户菜单的 API Tokens 创建令牌后可调用；令牌继承用户权限，不能绕过导出权限。
请求字段与认证方式以 [OpenAPI](../openapi/auto-ontology-api.json) 为准。

## 5. 导出的 model 究竟是什么

这里的 model 是**目录与语义定义文档**，不是 DeepSeek/Cohere 模型权重，也不是数据库完整备份。
原生格式的顶层是：

```yaml
data_layer:
  databases: []
  foreign_keys: []
  joins: []
semantic_layer:
  terms: []
  semantic_fks: []
  sql_attributes: {}
  custom_analyses: []
zones: []
```

此片段只解释结构；空对象不能拿来当作完整恢复文件。

| 部分 | 导出内容来源 |
| --- | --- |
| databases / schemas / tables / columns | 已持久化目录，含名称、类型、描述、主键、可空、唯一性和样本值等 |
| foreign_keys / joins | 已持久化物理关联和 Join 定义 |
| terms / columns_attributes | 已保存的术语及字段映射、术语代表的表 |
| semantic_fks | 已保存的语义外键 |
| sql_attributes | 已保存的派生 SQL 定义及关联 |
| custom_analyses | 已保存的可复用分析 SQL 及关联 |
| zones | 当前文档结构支持的区域信息；不要将其等同于完整权限策略备份 |

并非所有应用对象都在该格式中，例如聊天记录、用户、会话、模型密钥、完整权限状态和 PQL 分析不属于上述原生模型定义。
业务表的完整记录和向量数组也不是模型备份内容，但字段 `sample_values` 可能含业务样本，分发文件前应审查。

你打开的 `auto-ontology-model.yaml` 属于原生格式。
当前模型来源是：fixture 源库 → 目录采集 → `seed_graph_fixture` 的手工定义 → 已保存语义层 → 导出。
如果以后执行了自动语义编译或人工编辑，导出会反映当时存储中的结果，而不再只反映初始 fixture。

## 6. 代码中的生成逻辑

导出链路：

```text
界面选择数据库 ID、格式
  → Next.js /api/model/export 权限网关
  → FastAPI model_interchange/router.py
  → service._export_document
  → DAL 校验数据库 ID、读取目录/语义行
  → assemble_export_document 组装引用与作用域
  → 原生 YAML 序列化，或转换为 Ossie
```

导出本身**不会重新调用 LLM 生成语义**。
它查询内部存储，按数据库范围过滤，组装 ID 引用；SQL 字段关联使用已有关系和 SQL 列解析辅助构建。
导出会尝试从连接器获得数据库方言，失败时回退到保存的连接设置，因此源库暂时不可达不一定阻止导出。

自动语义编译是另一条流程：从目录和数据特征生成/持久化术语、映射与关系。
这些生成结果可能受模型、提示词和数据状态影响；固定 fixture 的手工定义用于避免这种不确定性。
导出文件不包含足以重演每次推理的完整提示词、调用历史和生成谱系。

代码入口：

- [前端导出界面](../../frontend/app/settings/import-export/ImportExportView.tsx)
- [导出服务与格式转换](../../auto_ontology/server/model_interchange/service.py)
- [DAL 读取和组装](../../auto_ontology/dal/model_interchange.py)
- [文档字段定义](../../auto_ontology/server/model_interchange/schemas.py)
- [固定测试语义定义](../../dev_tools/fixtures/seed_graph_fixture.py)
- [自动语义处理代码](../../auto_ontology/semantic/)

## 7. 原生与 Ossie 的区别

- **auto_ontology**：适合本项目内部交换，保留术语代表多张表的原生关系。
- **ossie**：面向 Apache Ossie 交换。当前转换把一个术语投影到一张表：保留其列属性最多的表，去掉其他表的属性，并过滤受影响的语义外键。因此可能有信息损失。
- **ossie + include_ontology**：ZIP 包含 `semantic_model.yaml`、`ontology.yaml`、`mapping.yaml`，映射通过相对文件名引用前两者。

Ossie 的全局指标命名空间无法表达重名 CustomAnalysis 时，导出会拒绝，而不是静默覆盖。
跨格式转换后要重新核对表映射、字段、SQL 和关系，不能承诺无损往返。

## 8. 导入与恢复边界

导入支持原生和 Ossie YAML，接口为 `POST /api/model/import`。
`replace` 默认 true，`embed` 默认 true；导入可能替换现有模型，也可能触发对外 Embedding 调用。
先导出备份，在隔离环境导入，再重新导出对比对象、引用和 SQL，最后做正反例问数验证。
导入成功响应不等于业务口径正确、数据源已连接或所有向量都已就绪。

模型文件不携带完整源库连接配置；在新环境仍需独立配置数据源、账号、模型服务和认证。
需要完整恢复应用时另做 PostgreSQL、SQLite 文件及必要配置的备份，不能只依赖 YAML。

## 9. Demo 的验收标准

- 页面、登录、两类迁移及长期运行服务正常。
- 数据源测试成功，目录表字段与源库一致。
- 语义对象有明确业务含义、物理映射和指标口径。
- Embedding 有效且维度一致，索引非空，检索返回正确数据库的候选。
- SQL 执行结果与直接查询一致；不存在数据时诚实返回空结果。
- Rerank、跨库问题、中文语义匹配等未验证能力明确标注，不混入已验证项。
- YAML 可审阅，知道它来自手工 fixture、自动编译还是人工修改。
