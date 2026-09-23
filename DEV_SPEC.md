<!-- Dev specification for RAG-Math-QB. -->
# Developer Specification (DEV_SPEC) — RAG-Math-QB

> 版本：0.2 — 分层颗粒度重写版，地基层与核心业务层任务已细化至可执行程度

## 目录

- 项目概述
- 核心特点
- 任务颗粒度与 TODO 管理规范
- 技术选型
- 测试方案
- 系统架构与模块设计
- 项目排期
- 可扩展性与未来展望

---

## 1. 项目概述

RAG-Math-QB 是一个**数学错题智能题库系统**：围绕"标准教学步骤 + 错因标签"做结构化检索与推荐，解决传统题库只按题面文字相似度搜题、抓不到学生真正错在哪一步的问题。

### 职责边界（重要，决定本项目做什么、不做什么）

本项目与未来的"辅导讲师 Agent"（独立项目 `Intelligent-Tutoring-Agent/`）职责二分：

- **RAG-Math-QB（本项目）**：被动的存储 + 检索服务。管理标准步骤库、错因标签库、题目库；对外暴露结构化查询/提议接口。**不包含任何对话式推理、多轮澄清、步骤拆解、路径判断的 LLM 逻辑。**
- **辅导讲师 Agent（未来项目，不在本仓库范围内）**：负责和学生的自然语言交互、引导学生澄清"错在哪一步"、对未见过的题型做步骤拆解分析、判断学生这次走的是哪条证明路径。它作为 MCP Client 调用本项目暴露的 MCP tools。

这个边界决定：本项目的每一个功能模块都应该问自己"这是检索/存储能力，还是推理/对话能力？"——后者一律不做，交给辅导讲师 Agent。**图片识别是一个容易混淆边界的例外**：把图片转换成结构化题目数据（OCR/Vision LLM 感知能力）属于本项目；但学生发起"帮我看这道错题"这个交互请求、以及后续的理解和讲解，属于辅导讲师 Agent（见 §3.1 和 `docs/DECISIONS.md`）。

完整决策推理过程见 `docs/DECISIONS.md`。

### V1 覆盖范围

- **学科**：初中数学
- **章节**：几何证明类（如相似三角形、全等三角形、圆的相关证明题）。选择理由：该章节在"标准步骤 + 错因"规范化上技术难度最高——题目通常涉及图形（图片处理复杂度高）、且同一题可能存在多条被认可的标准证明路径（步骤路径非线性，"标准步骤"这个抽象概念本身更难定义）。用最难的场景压力测试架构，验证通过后扩展到其他章节（如代数类）是降维，不是加维。
- **数据结构可扩展性**：字段命名和约束不写死为数学专属（如使用 `chapter_code`/`subject_code` 而非 `math_chapter_code`），为未来跨学科扩展留出空间，但 V1 本身不做任何跨学科的额外设计投入。

### V1 明确不做

- 不做对话式错因澄清（属于辅导讲师 Agent）
- 不做步骤拆解的多轮推理交互（属于辅导讲师 Agent）
- 不做"学生这次走的是哪条证明路径"的动态判断——多路径数据本身是审核录入阶段的静态数据，RAG 只做检索匹配，不做路径推理
- 不做 REST API（触发条件：出现非 MCP Client 的独立调用方，见 §3.3 和 `docs/DECISIONS.md`）
- 不做 AI 生成内容的自动合并入库（一律走 review queue，审核前对检索完全不可见）

---

## 2. 核心特点

### 多媒介题目录入，感知与交互分离

题目来源可能是学生拍照上传的图片、老师/管理员批量扫描的纸质题库、或手动文字录入。三种来源统一转换为结构化 `Question` 记录后入库，复用同一套下游检索/存储逻辑。图片走 Vision LLM 转结构化文字（类比通用 RAG 项目的 Image Captioning 思路，但目标是提取题目结构而非生成描述）。

**图片识别的职责边界**：图片 → 结构化题目数据的转换能力（感知层）属于本项目，作为一个可被调用的 MCP tool 暴露；但"学生发起拍照提问"这个交互入口、以及后续的理解/对话/讲解，属于辅导讲师 Agent——辅导讲师 Agent 作为 MCP Client 调用本项目的图片转结构化 tool，拿到结构化数据后才开始自己的推理工作。学生不直接对接本项目。

录入触发方式包括学生即时提交（经辅导讲师 Agent 中转）和后台管理员批量导入，共享同一套 Ingestion Pipeline。

### 题目与标准步骤的多对多关联（多路径共存模型）

由于 V1 覆盖几何证明类题目，同一道题可能存在多条都被认可的标准证明路径。数据模型上，`questions` 与 `canonical_steps` 不是一对一/一对少，而是通过多对多关联表支持"一题多套标准步骤序列（`step_sequence`）"。

**关键约束**：路径数据必须是审核录入阶段产生的静态数据（由出题人录入，或 AI 提议候选路径后经人工审核确认），本项目在检索时**绝不**动态推理或判断"学生这次走的是哪条路径"——这类判断属于语义理解/推理能力，越界进入辅导讲师 Agent 的职责范围。数据结构复杂度的提升不等于职责越界；只要路径本身是静态录入的候选集合，本项目的工作依然是纯粹的检索匹配。

### 错因标签的独立建模

错因标签（misconception tags）不归属于任何特定标准步骤，而是作为全局独立表存在，通过关联表与标准步骤建立多对多关系（同一个错因标签可以在多个不同步骤下都是典型错因，如"符号错误"可能同时出现在"去括号"和"移项"两个步骤）。

### Step-aware Hybrid Retrieval

检索不只是题面语义相似度匹配，而是综合标准步骤匹配、错因标签匹配、章节、难度等维度做混合检索与重排。具体排序权重公式属于待解锁任务（见 §6 待解锁任务区块），需要真实数据和老师反馈驱动迭代，不在本阶段拍板。

### Review-gated 知识增长

AI 从新题目/错例中挖掘出的候选标准步骤、候选错因标签、候选题目-步骤路径映射，在老师审核通过前，**对线上检索/推荐功能完全不可见**。这是本项目区别于"AI 生成内容直接生效"类系统的核心信任边界，必须严格执行，不留灰色地带。

### 双接口服务

- MCP Server：面向辅导讲师 Agent 的程序对程序接口
- Streamlit Dashboard：面向老师的可视化管理界面（增删改题目、审核候选内容），部署到可远程访问的服务器

---

## 3. 任务颗粒度与 TODO 管理规范

> 本章节是方法论规范，`auto-coder` skill 在解析本文档、执行阶段 6 排期任务前应先理解本章节的约定。完整决策背景见 `docs/DECISIONS.md`「方法论决策：DEV_SPEC 的分层颗粒度与 TODO 待解锁区块规范」一节。

本项目业务需求仍在演进中，不采用"全篇统一精细度"的写法，而是按内容的确定程度分层：

- **地基层**（工程骨架、数据模型、可插拔接口抽象）：已确定、后续不太会变的部分，写到函数签名级别的精细度。
- **核心业务层**（Ingestion、检索、Rerank、MCP tools）：方向已定但部分参数未定，写清楚接口契约和验收标准，不把具体实现算法/参数钉死。
- **待定层**：现在无法确定的内容，不写入正式任务，转入 §6 末尾的「⏸ 待解锁任务」区块。

**待解锁任务区块的规则**：

1. 不使用正式任务编号（A1/B1/C1 这类），因此 `auto-coder` 的 Find Task 扫描逻辑**不会**认领这些条目。
2. 每条待解锁任务必须写清楚三件事：这个任务本来要做什么 / 为什么现在无法开始（缺什么前置条件）/ 依赖哪个已完成的正式任务或哪个外部输入（真实数据、老师反馈等）才能解锁。
3. 当满足解锁条件时，由人工将其转正、编入 §6 正式排期表并赋予编号，不由 `auto-coder` 自动转正。

这个规范的目的：避免把还没想清楚的设计硬编造成"看起来确定"的样子（虚假精确），也避免完全跳过不写导致排期表出现依赖关系空洞。

---

## 4. 技术选型

### 4.1 数据摄取流水线

**目标：** 构建统一、可配置、可观测的题目摄取流水线，把图片、PDF、手动表单三种来源统一转换为结构化 `Question` 记录，供检索层消费。该能力应是可重用的库模块，供后台批量导入脚本、Streamlit Dashboard、学生即时提交入口（经辅导讲师 Agent 中转）共同调用。

- **自研 Pipeline 框架（设计定位与 §4.4 可插拔架构一致，不依赖 LlamaIndex 等第三方 RAG 框架）**：
  - 采用自定义抽象接口（`BaseLoader`/`BaseTransform`/`BaseEmbedding`/`BaseVectorStore`，无 `BaseSplitter`——本项目不做 Chunking，见下方说明），实现完全可控的可插拔架构。
  - 支持可组合的 Loader → Transform → Embed → Upsert 流程（比通用文档 RAG 少一环 Splitter），便于实现可观测的流水线。
  - 与主流 Embedding provider 有良好适配，架构中统一使用 Chroma 作为向量存储（与 §4.4 保持一致）。

设计要点：

- **不做 Chunking（切分）**：与通用文档 RAG 不同，本项目不引入 Splitter 层。原因见 `docs/DECISIONS.md`「数据摄取架构：不做 Chunking，但需要"多题目边界识别"」——单道题目的长度远低于 embedding 窗口限制，不存在需要切分的技术约束；检索与存储的最小单元就是一整道题。
- **明确分层职责**：
  - Loader：负责把原始输入（图片/PDF/表单）解析为统一的 `Question` 对象列表（类型定义集中在 `src/core/types.py`）。三种来源的输入形态本质不同（文件路径 / 结构化字典 / 图片二进制），因此 `BaseLoader.load()` **不强制统一参数签名**，只约束返回值必须是 `List[Question]`——即使输入只对应一道题，也返回长度为 1 的列表，保持下游 Pipeline 处理逻辑统一。
  - `PdfLoader.load(file_path)`：从 PDF 题库文档中识别题目边界并抽取多道题目（核心任务是"识别有几道题、边界在哪"，而非文档转 Markdown）。
  - `ImageLoader.load(image_data, source_hint)`：从图片中识别并抽取题目，供学生实时拍照（`source_hint="student_realtime"`，经辅导讲师 Agent 中转调用，见职责边界章节）与老师批量扫描（`source_hint="batch_scan"`）两种场景共用。**本 Loader 只做感知转换，不涉及理解学生意图、对话、错因判断**（那属于辅导讲师 Agent 的职责，见 `docs/DECISIONS.md`「图片识别能力归属」）。
  - `ManualEntryLoader.load(form)`：接收 Streamlit Dashboard 手动录入的结构化表单，无需解析逻辑，恒定返回长度为 1 的列表。
- **摄取记录（`ingestion_records`）与 `questions` 表分离**：记录来源媒介类型（image/pdf/manual）、原始文件引用、导入时间、导入批次、导入人（学生/管理员）。设计理由见 `docs/DECISIONS.md`「来源/摄取元数据独立建表」——题目表保持纯业务属性，摄取元数据是独立的生命周期/查询维度。
- **触发入口**：学生即时提交（经辅导讲师 Agent 中转、单条）、后台管理员批量导入（多条，附带批次记录），两者复用同一 Pipeline，仅入口不同。
- **Review-gate 过滤**：新识别出的题目默认 `review_status="pending"`，在老师审核通过前不参与任何检索（对应硬规则，见 `docs/DECISIONS.md`「Review-Gate 规则」）。
- **Dedup & Normalize（去重与归一化）**：在写入向量库前运行去重检查——与通用文档 RAG 一致，防止重复索引，具体机制见下方"前置去重"与"幂等性设计"两节。

**前置去重（Early Exit / File Integrity Check）**（与通用文档 RAG 一致，直接复用；对本项目同样必需——老师重复上传同一份 PDF 题库或同一张错题图片，不应重新触发一遍完整的识别+向量化流程）：

- **机制**：在解析文件/图片前，计算原始输入的 SHA256 哈希指纹。
- **动作**：检索 `ingestion_history` 表，若发现相同哈希且状态为 `success` 的记录，直接跳过后续所有处理（识别、向量化），实现零成本增量更新。
- **表结构**：

```sql
CREATE TABLE ingestion_history (
    input_hash    TEXT PRIMARY KEY,   -- SHA256(原始文件/图片内容)
    source_type   TEXT NOT NULL CHECK(source_type IN ('image', 'pdf', 'manual')),
    status        TEXT NOT NULL CHECK(status IN ('success', 'failed', 'processing')),
    processed_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    error_msg     TEXT,
    question_count INTEGER            -- 该次摄取识别出的题目数
);
CREATE INDEX idx_status ON ingestion_history(status);
```

- **查询逻辑**：`SELECT status FROM ingestion_history WHERE input_hash = ? AND status = 'success'`。命中则跳过。
- **手动录入（`ManualEntryLoader`）不适用此机制**——表单提交没有"原始文件"概念，每次提交视为新内容，去重交由 `questions` 表本身的内容层面幂等（见下方幂等性设计）处理。

> **📌 持久化存储架构统一说明**
>
> 本项目在多个模块中采用 **SQLite** 作为轻量级持久化存储方案，避免引入重量级数据库依赖，保持本地优先（Local-First）的设计理念：
>
> | 存储模块 | 数据库文件 | 用途 | 表结构关键字段 |
> |---------|-----------|------|---------------|
> | **文件完整性检查** | `data/db/ingestion_history.db` | 记录已处理输入的 SHA256 哈希，实现增量摄取 | `input_hash`, `status`, `processed_at` |
> | **图片索引映射** | `data/db/image_index.db` | 记录题目关联图形文件的 `image_id → 文件路径` 映射，支持图片检索与引用 | `image_id`, `file_path`, `question_id` |
> | **BM25 索引元数据** | `data/db/bm25/` | 存储倒排索引和 IDF 统计信息（未来可扩展用 SQLite） | 当前使用 pickle，可迁移至 SQLite |
>
> **设计优势**：
> - **零依赖部署**：无需安装 MySQL/PostgreSQL 等数据库服务，`pip install` 即可运行——对应本项目"个人可维护、可部署到教育机构本地环境"的定位。
> - **并发安全**：WAL（Write-Ahead Logging）模式支持多进程安全读写（学生提交与管理员批量导入可能并发发生）。
> - **持久化保证**：摄取历史和索引映射在进程重启后自动恢复，避免重复计算。
> - **架构一致性**：所有 SQLite 模块遵循相同的初始化、查询与错误处理模式，便于维护与扩展。
>
> **升级路径**：当题库规模扩展至多机构、分布式部署场景时，可通过统一的抽象接口将 SQLite 替换为 PostgreSQL 或 Redis，无需修改上层业务逻辑。

**Transform 阶段：结构清洗（仅 `PdfLoader`/`ImageLoader` 场景，`ManualEntryLoader` 跳过）**：

- 对识别出的题面文本做规则去噪（剔除页眉页脚、乱码、多余空白），确保入库的 `Question.stem` 是自包含、干净的文本——与通用文档 RAG 的"智能重组"对应，但不涉及"合并被物理切断的段落"（不适用，本项目没有跨块合并的场景）。
- **语义元数据注入（Semantic Metadata Enrichment，仅 `PdfLoader` 批量场景，题目量较大时才有意义）**：在基础元数据（章节、来源）之上，利用 LLM 为每道题目自动生成 `topic_tags`（知识点标签，如"相似三角形""平行线判定"），注入 `metadata` 字段，用于支撑 Dashboard 题库浏览器的筛选与统计维度。与通用文档 RAG 的 `Title`/`Summary`/`Tags` 三件套不同，本项目题目本身已有 `stem`（相当于自带摘要）、`chapter_code`（相当于自带分类），不需要额外生成标题和摘要，只需要这一层知识点标签补充。
- 原子化与幂等：单道题目的清洗失败不阻塞同批次其他题目的处理，失败题目单独标记 `metadata.transform_failed=true`，进入待审核队列由老师判断。

**Embedding 阶段（双路向量化）**：

- **差量计算（Incremental Embedding / Cost Optimization）**：在调用 Embedding API 前，计算 `Question.stem` 的内容哈希（`content_hash`）。仅对数据库中不存在的新内容哈希执行向量化计算；若题目文本未变但其他字段（如 `difficulty`）被老师编辑，直接复用已有向量，避免重复计费。
- **核心策略**：为支持高精度的混合检索（Hybrid Search），系统对每道题目并行执行双路编码计算：
  - **Dense Embeddings（语义向量）**：调用 Embedding 模型生成高维浮点向量，捕捉题面的深层语义关联——对应 §4.2 Dense Route 依赖的向量来源，解决"标签相同、具体条件不同"的检索难题。
  - **Sparse Embeddings（稀疏向量）**：利用 BM25 编码器生成稀疏向量（关键词权重），捕捉精确的关键词匹配信息——对应 §4.2 Sparse Route 依赖的索引来源，解决专有名词（如特定几何术语）查找问题。
- **批处理优化**：向量化计算采用 `batch_size` 驱动的批处理模式（而非逐题调用 API），最大化吞吐并减少网络往返（RTT）——老师批量导入一份 PDF 题库可能一次产生几十道新题，逐题调用会显著拖慢摄取速度。

**Upsert & Storage（索引存储）**：

- **存储后端**：统一使用向量数据库（Chroma）作为存储引擎，同时持久化 Dense Vector、Sparse Vector 以及 Transform 阶段生成的富 metadata。
- **All-in-One 存储策略**：执行原子化存储，每条 `QuestionRecord` 同时包含：
  1. **Index Data**：用于计算相似度的 Dense Vector 和 Sparse Vector。
  2. **Payload Data**：完整的题目原始内容（`stem`/`answer`）及 metadata。

  机制优势：检索命中 `question_id` 后能立即取回题面、答案等完整信息，无需额外查库操作（Lookup），保障检索阶段的毫秒级响应——对应辅导讲师 Agent 实时对话场景对检索延迟的要求。

**幂等性设计（`Question.id` 生成算法）**：

- `id = hash(source_type + source_ref + content_hash)`——`source_type`/`source_ref` 来自 `ingestion_records`，`content_hash` 是题面内容哈希。同一来源、同一内容重复摄取产生相同 `id`，写入时采用 Upsert 语义，确保不产生重复记录。
- 手动录入场景 `source_ref` 为空，`id` 退化为 `hash("manual" + content_hash)`——意味着两位老师各自手动录入完全相同的题面会被判定为同一题目并合并，这是刻意设计（避免重复题目污染题库），而非缺陷。
- **原子性保证**：以 Batch 为单位进行事务性写入（一批题目要么全部成功入库，要么全部回滚），确保索引状态的一致性——避免批量导入中途失败导致部分题目有向量无 metadata、或有 metadata 无向量的不一致状态。

**题目生命周期管理（`QuestionManager`，对应通用文档 RAG 的 `DocumentManager`，支撑 Dashboard 题库浏览器/摄取管理页）**：

- 独立模块 `src/ingestion/question_manager.py`，负责跨存储的协调操作：
  - `list_questions(chapter_code?, review_status?) -> List[QuestionInfo]`：列出题目及统计信息（关联标准步骤数、摄取时间、来源类型）。
  - `get_question_detail(question_id) -> QuestionDetail`：获取单道题目的详细信息（题面、答案、metadata、关联图片、`ingestion_records` 溯源信息）。
  - `delete_question(question_id) -> DeleteResult`：协调删除跨存储的关联数据：
    1. **VectorStore** — 删除该题目的 dense/sparse 向量
    2. **`ingestion_records`** — 删除对应摄取记录
    3. **`ingestion_history`** — 若该题目是某次摄取的唯一产出，移除处理记录，使原始输入可重新摄取
    4. **图片文件**（若有）— 删除关联的题目图形文件
  - `get_chapter_stats(chapter_code?) -> ChapterStats`：返回章节级统计（题目数、待审核数、存储大小）。

**Pipeline 进度回调**（与通用文档 RAG 一致，直接复用，支撑 Dashboard 摄取管理页的实时进度条）：

```python
def run(self, source: Any, source_type: str,
        on_progress: Callable[[str, int, int], None] | None = None) -> IngestionResult:
```

- 回调签名：`on_progress(stage_name: str, current: int, total: int)`。
- 各阶段（load / transform / embed / upsert）处理每个 batch 时调用回调。`on_progress` 为 `None` 时行为不受影响，不影响 CLI 和测试场景。

**存储层接口扩展**（支撑 `QuestionManager` 的删除操作）：

- `BaseVectorStore` 新增 `delete_by_id(question_id: str) -> bool`
- `BM25Indexer` 新增 `remove_document(question_id: str) -> None` — 移除指定题目的倒排索引条目，对应 `delete_question` 步骤中"删除该题目的 dense/sparse 向量"里 sparse 一侧的具体接口
- `FileIntegrityChecker`（封装 `ingestion_history` 表操作）新增 `remove_record(input_hash: str) -> None` 和 `list_processed() -> List[dict]`

**数据类型分层（对应通用文档 RAG 的 Document → Chunk → ChunkRecord 三层，本项目因不做 Chunking 而简化为两层）**：

- **`Question`**（Loader 的直接产出，识别结果，尚未向量化）：
  - `id: str`（基于内容哈希生成，确定性 ID，同一题目重复识别产生相同 ID，支撑幂等摄取）
  - `stem: str`（题面文本）
  - `answer: Optional[str]`（可为空，允许后续人工补充）
  - `metadata: Dict[str, Any]`，至少包含：`chapter_code`（必填）、`subject_code`（必填，V1 恒为 `"math"`）、`difficulty`、`review_status`（`"pending"` | `"approved"`）、`source_ref`（指向 `ingestion_records` 的外键，见下）
  - `image_ref: Optional[str]`（几何题关联的图形文件路径，若题目本身含图形；对应通用文档 RAG 的 `metadata.images`，但本项目里图片是题目本身的一部分而非文档中的插图，因此提升为顶层字段而非塞进 metadata 列表）
- **`QuestionRecord`**（Embedding 之后、写入向量库前的最终存储记录，对应通用文档 RAG 的 `ChunkRecord`）：
  - 继承 `Question` 的全部字段
  - `dense_vector: Optional[List[float]]`
  - `sparse_vector: Optional[Dict[str, float]]`
  - 无需 `start_offset`/`end_offset`/`source_ref`（这些字段服务于 Chunk 在原文档中的定位，本项目题目本身就是完整单元，不存在"在更大文档中的位置"这个概念）

**`ingestion_records` 表字段**（与 `questions` 表分离，见 `docs/DECISIONS.md`「来源/摄取元数据独立建表」）：

```sql
CREATE TABLE ingestion_records (
    id            TEXT PRIMARY KEY,
    question_id   TEXT NOT NULL,      -- FK -> questions.id
    source_type   TEXT NOT NULL CHECK(source_type IN ('image', 'pdf', 'manual')),
    source_ref    TEXT,               -- 原始文件路径/引用，manual 来源可为空
    source_hint   TEXT,               -- 'student_realtime' | 'batch_scan'，仅 image 来源使用
    imported_by   TEXT NOT NULL,      -- 'student' | 'admin'
    imported_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    batch_id      TEXT                -- 批量导入场景下的批次标识，单条提交为空
);
CREATE INDEX idx_question_id ON ingestion_records(question_id);
CREATE INDEX idx_batch_id ON ingestion_records(batch_id);
```

**多题目边界识别机制**（`PdfLoader`/`ImageLoader` 从一份输入中识别多道题目时的定位机制，对应通用文档 RAG 的图片占位符+位置信息设计，但服务的目的不同——通用文档 RAG 定位的是"图片在文本流中的位置"，本项目定位的是"每道题在原始输入中的边界"）：

- 每道被识别出的题目，在 `metadata` 中记录 `source_position`：`{"page": int, "sequence": int}`（`page` 为原始文档/图片的页码，无分页场景可省略；`sequence` 为该页内的题目序号，从 0 开始），用于后续排查"这道题识别得对不对"时定位回原始输入。
- 识别失败或置信度过低的候选，不静默丢弃，而是以 `review_status="pending"` 且 `metadata.recognition_confidence` 低于阈值的形式入库，交由老师在待审核队列页判断是否为有效题目——避免识别错误导致数据永久丢失、无法追溯。

### 4.2 检索流水线

**目标：** 给定辅导讲师 Agent 传入的结构化错因信息，检索并排序出对症的练习题。本模块实现核心的检索引擎，采用"多阶段过滤（Multi-stage Filtering）"架构——先用结构化条件层层收窄候选范围，再用语义排序精确排出对症程度。与通用文档 RAG 不同，本模块的输入不是"待消歧的自由文本查询"，而是**已经结构化的错因描述**——标准步骤 ID、可选的错因标签 ID、章节、难度——上游的语义理解、路径判断、自由文本归一化工作已由辅导讲师 Agent 完成，本模块只负责"拿结构化条件找题、排序"。

设计要点：

- **核心假设**（对应通用文档 RAG 的"输入已消歧"假设）：输入的 `ProcessedQuery` 已由上游辅导讲师 Agent 完成语义理解与结构化——具体来说，Agent 已经把学生的自由文本错误描述归一化为 `canonical_step_id`（可能还完成了多路径判断，决定学生的错误对应哪条候选路径中的哪个节点，但这个判断结果本身不会作为参数直接传入，因为本模块的路径匹配是基于命中的 `canonical_step_id` 反查候选题目关联的路径，而非接收一个"路径 ID"参数）。本模块**不做任何语义理解层面的二次确认**——如果传入的 `canonical_step_id` 不存在于 taxonomy 中，直接返回空结果 + 明确错误信息，不做模糊匹配或猜测。这个假设是职责边界的直接延伸（见 `docs/DECISIONS.md`「职责边界」），必须在这里显式写出，因为它决定了本模块的输入校验策略——校验"格式是否合法"，不校验"语义是否合理"（语义合理性是上游 Agent 的职责）。
- **Query 输入契约**：`ProcessedQuery` 完整字段：
  - `canonical_step_id: str`（必填）
  - `misconception_tag_id: Optional[str]`
  - `chapter_code: Optional[str]`
  - `difficulty: Optional[int]`
  - `top_k: int`（默认值为待解锁任务，当前占位 10）
  - `exclude_question_ids: List[str]`（可选，用于排除学生已经做过的题，默认空列表）
  - `reference_stem: Optional[str]`（学生当前做错的这道题的题面文本，辅导讲师 Agent 若持有则传入，用于驱动 Dense Route 语义检索；不提供时 Dense Route 跳过，仅用 Sparse + 结构化过滤，检索质量会降级但不阻断流程——见下方 Hybrid Search Execution）
  - 无 `keywords`/`expanded_terms` 字段——这两个字段服务于自由文本查询的关键词提取/同义词扩展，本项目的核心结构化字段不需要这层转换。
- **查询转换与扩张策略（对应通用文档 RAG 此节，本项目判定不适用）**：原版此节包含 Keyword Extraction（NLP 提取关键实体生成稀疏检索 Token）和 Query Expansion（同义词/别名扩展）两个机制，目的是把自由文本查询转化为适合检索的形式。本项目的核心结构化字段不需要这层转换；但 `reference_stem`（若提供）仍需要基础的分词/关键词提取供 Sparse Route 使用——这部分复用与通用文档 RAG 相同的 NLP 处理，只是触发条件从"总是执行"变成"仅当 `reference_stem` 存在时执行"。
- **Metadata Filtering 完整策略**（与通用文档 RAG 一致，原则为"先解析、能前置则前置、无法前置则后置兜底"）：
  - **硬过滤 / 前置（Pre-filter）**：`review_status != approved` 的题目、以及 `chapter_code`/`canonical_step_id` 不满足的候选，在 Dense/Sparse 检索阶段之前直接排除，不进入候选集——因为这些是索引层面可精确支持的结构化字段，前置能缩小候选集、降低成本。对应 `docs/DECISIONS.md`「技术选型验证」中确立的"结构化过滤 + 语义排序"分工原则。
  - **硬过滤 / 后置（Post-filter，safety net）**：`misconception_tag_id`（可选字段，题目数据里可能缺失该标注）在 Rerank 前统一做后置过滤——若题目未标注错因标签，默认"宽松包含"（missing→include），不因标注缺失而误杀本该召回的候选，避免因数据标注不完整导致漏检。
  - **软偏好（Soft Preference）**：`difficulty` 不做硬过滤（学生错因对应的标准步骤，难度相近但不完全相等的题目依然有练习价值），而是作为 Rerank 阶段的排序信号之一参与加权（对应"难度匹配"这一分项，见下方 Rerank 权重）。
- **Hybrid Search Execution（双路混合检索）**：
  - **并行召回（Parallel Execution）**：
    - **Dense Route**：仅当 `reference_stem` 存在时执行——计算 `reference_stem` 的 Embedding → 检索向量库（Cosine Similarity，检索范围是已通过结构化过滤的候选集合）→ 返回 Top-N 语义候选。`reference_stem` 缺失时跳过此路，直接进入 Sparse Route 结果。这一路捕捉"标签相同、具体条件不同"的语义差异（几何题的图形构造差异即典型场景），对应 `docs/DECISIONS.md`「技术选型验证」六点论证中的第一点。
    - **Sparse Route**：对 `reference_stem`（若有）做关键词提取 → BM25 算法检索倒排索引 → 返回 Top-N 关键词候选；若 `reference_stem` 也缺失，Sparse Route 退化为仅按结构化字段过滤，不做关键词打分（此时排序完全依赖 Rerank 阶段的结构化匹配度）。
    - Top-N 数量：与 Rerank 的 Top-M 同为待解锁任务的可调参数（见 §8 末尾），当前占位 20。
  - **结果融合（Fusion）**：
    - 采用 RRF（Reciprocal Rank Fusion）算法，不依赖 Dense/Sparse 各路分数的绝对值（两路量纲不同，直接比较无意义），而是基于排名的倒数进行加权融合，平滑因单一模态缺陷导致的漏召回。
    - 公式：`Score = 1 / (k + Rank_Dense) + 1 / (k + Rank_Sparse)`，`k` 可配置。
    - **单路降级场景**：`reference_stem` 缺失导致 Dense Route 未执行时，Fusion 直接使用 Sparse Route 排名（不套用 RRF 公式，因为只有一路数据）。
- **多路径匹配加分（本项目特有设计）**：候选题目可能关联多条标准步骤路径（`step_sequence`，见 §7 数据模型的多对多关系）。若同一题目的多条路径都命中传入的 `canonical_step_id`，视为该题对这个错误步骤更有代表性，给予小幅加分；命中单条路径的题目不因"存在其他不相关路径"而受影响。**加分幅度必须受控（远小于标准步骤匹配/错因匹配等主信号权重）**，避免"路径数量多"本身压过"是否真正对症"这个核心排序目标——具体加分系数为待解锁任务（见 §8 末尾），当前用小值占位。**路径本身是审核录入阶段的静态数据，本模块只做路径命中判断与加分，不做路径推理**（硬边界，见 `docs/DECISIONS.md`）。
- **Rerank（精排）**：
  - 候选集按标准步骤匹配度、错因标签匹配度、题面语义相似度、章节匹配、难度匹配、多路径命中加分共同加权排序，具体权重公式为待解锁任务（见 §8 末尾），当前用等权重占位跑通链路。
  - 可插拔后端：None（直接用 Fusion 排名）/ Cross-Encoder / LLM Rerank，与通用可插拔架构一致（见 §4.4）。候选数量参数（Top-M，与通用文档 RAG 一致，本项目规模更小仍沿用同一档位）：Cross-Encoder 默认对 Top-M=10~30 的候选执行精排；LLM Rerank 候选数更小（M≤20），控制成本与稳定性，要求输出严格结构化格式（JSON 格式的排序后 `question_id` 列表）。
  - **Fallback 语义**：精排后端不可用/超时/失败时，必须回退到 Fusion 阶段排名，返回结果需显式标记是否使用了 Fallback 及原因，不能静默降级（面向可观测性，便于后续排查排序质量问题）。
- **输出契约**：`RetrievalResult` 完整字段（每个检索阶段——Dense/Sparse/Fusion/Rerank——都以此类型作为候选的统一表示，各阶段只更新 `score`/`stage_scores`，不改变类型结构）：
  - `question_id: str`
  - `score: float`（当前阶段的综合分数）
  - `stage_scores: Dict[str, float]`（各阶段独立分数的留痕，如 `{"dense": 0.8, "sparse": 0.6, "fusion": 0.72}`，用于 §4.5 Query Trace 展示"Dense vs Sparse 对比"）
  - `matched_path_ids: List[str]`（命中的 `step_sequence` 路径 ID 列表，长度 >1 时对应多路径命中加分场景）
  - `metadata: Dict[str, Any]`（题目的 `chapter_code`/`difficulty` 等，供上游过滤/展示，不需要重新查库）

  最终返回给调用方的结果在 `RetrievalResult` 基础上补充：`score_breakdown`（各排序信号的分项得分，含多路径加分项）、`why_recommended`（简短说明命中原因，供辅导讲师 Agent 或 Dashboard 展示）。不允许只返回裸分数——对应"排序质量是核心商业价值"的结论（见 `docs/DECISIONS.md`「技术选型验证」）。

### 4.3 MCP 服务设计

**目标：** 设计并实现一个符合 Model Context Protocol 规范的 Server，作为结构化题库上下文提供者，供辅导讲师 Agent（MCP Client）调用。

设计要点：

- **核心设计理念**：
  - 协议优先：严格遵循 MCP 官方规范（JSON-RPC 2.0），只做协议合规实现，不写死专属于某个 Client 的非标准扩展——这为未来接入除辅导讲师 Agent 外的其他合规 MCP Client 保留了空间，但**本阶段不设计多租户/多调用方的权限治理机制**（触发条件与 REST API 一致：出现真实的第二调用方时才设计，见 `docs/DECISIONS.md`「对外接口形态」）。
  - 排序理由透明：对应通用文档 RAG 的"引用透明"理念，本项目的等价物是 `score_breakdown` + `why_recommended`（见 §4.2），而非文档的 `source_file`/`page`/`chunk_id`。具体结构上，`search_questions` 返回的每道题目在 `structuredContent` 中采用统一格式：
    ```json
    {
      "questions": [
        {
          "question_id": "q_8f3a2b",
          "score": 0.87,
          "score_breakdown": {
            "canonical_step_match": 0.9,
            "misconception_match": 0.8,
            "dense_similarity": 0.75,
            "multi_path_bonus": 0.05
          },
          "why_recommended": "该题标准步骤与错因标签均命中，题面语义与学生原题高度相似",
          "matched_path_ids": ["path_012"]
        }
      ]
    }
    ```
    同时在 `content` 数组第一项提供 Markdown 格式的人类可读摘要（题目列表 + 简短推荐理由），保证辅导讲师 Agent 无论是否解析 `structuredContent` 都能拿到可用信息——与通用文档 RAG"始终在 content 第一项提供纯文本兜底"的原则一致。
  - 单一确定 Client：不需要像通用文档 RAG 那样为不同 Client（Copilot vs Claude Desktop）设计差异化的降级/适配策略，因为当前唯一的 Client 就是辅导讲师 Agent，其能力边界由本项目团队自行定义。
  - **开箱即用（Zero-Config for Client）**：辅导讲师 Agent 端接入本 Server 无需任何特殊配置，只需在其 MCP 配置中添加启动命令即可使用全部工具——这是 Stdio Transport 的天然优势，也延续到工具设计层面：所有工具的参数和返回值都是自解释的结构化数据，不需要额外的接入文档或握手流程。
  - **多模态友好（Multimodal-Ready）**：返回格式同时支持文本与图像内容类型（见下方"返回内容设计"），几何题的图形是本项目的核心场景之一，多模态支持不是预留扩展，而是当前就要用到的能力。
- **传输协议**：Stdio（本地子进程通信），与通用文档 RAG 设计一致——无需网络端口/鉴权，数据不经网络，`stdout` 仅输出合法 MCP 消息，日志统一走 `stderr`。
- **SDK 选型**：优先采用 Python 官方 MCP SDK（`mcp`），复用其 `@server.tool()` 声明式定义方式，不自行实现协议底层细节。
- **SDK 备选方案**：若未来需要深度定制 HTTP 行为（自定义中间件、复杂鉴权流程）或出现独立于辅导讲师 Agent 的第二调用方（触发 REST API 的场景，见 `docs/DECISIONS.md`「对外接口形态」），可考虑 FastAPI + 自定义协议层；权衡是开发成本更高，需自行实现能力协商（Capability Negotiation）、错误码映射，且需持续跟进协议版本更新。本阶段无此需求，官方 SDK 已充分满足。
- **协议版本协商**：跟踪 MCP 最新稳定版本，在 `initialize` 阶段完成 Client/Server 能力协商，确保兼容性，与通用文档 RAG 一致，不做改动。
- **Tools 设计**：Server 通过 `tools/list` 向辅导讲师 Agent 注册可调用的工具函数，设计遵循"单一职责、参数明确、输出丰富"原则——每个工具只做一件事（查询/检索/转换/提议/反馈五类不混合），参数语义清晰不复用，返回值携带足够信息支撑调用方决策（如 `score_breakdown`），不要求调用方二次查询补全上下文。按职责分五类，共七个工具。

| 类别 | 工具名称 | 功能描述 | 典型输入参数 | 输出特点 |
|---|---|---|---|---|
| 查询 | `lookup_canonical_steps` | 查询某章节下的候选标准步骤列表 | `chapter_code` | 标准步骤列表（含所属路径信息） |
| 检索 | `search_questions` | 核心检索入口，见 §4.2 | `canonical_step_id`, `misconception_tag_id?`, `chapter_code?`, `difficulty?`, `top_k?` | 含 `score_breakdown`/`why_recommended` 的题目列表 |
| 图片感知转换 | `ingest_question_from_image` | 图片转结构化题目，见 §4.1 | `image_data`, `source_hint` | 识别出的题目列表（`review_status=pending`） |
| 提议写入 | `propose_canonical_step` | 提议新标准步骤 | 候选步骤内容 | 确认信息 + 提议记录 ID |
| 提议写入 | `propose_misconception_tag` | 提议新错因标签 | 候选标签内容 | 确认信息 + 提议记录 ID |
| 提议写入 | `propose_question_mapping` | 提议题目与标准步骤的映射，粒度覆盖单步骤与整条路径 | `question_id`, `step_ids`（长度 1 为单步骤映射，长度 >1 为路径提议） | 确认信息 + 提议记录 ID |
| 反馈回流 | `submit_recommendation_feedback` | 记录学生/老师对某次推荐结果的反馈，作为未来 Rerank 权重迭代的数据来源 | `case_id`（对应某次检索结果）, `question_id`, `feedback`（有帮助/无帮助等） | 确认信息 |

  - 所有"提议写入"与"反馈回流"类工具**只返回简洁确认（成功与否 + 对应记录 ID），不返回审核状态或详细内容**——调用方（辅导讲师 Agent）不需要在当前对话轮次追踪审核进度，这是本项目与通用文档 RAG 工具设计的一个刻意简化。
  - 所有提议写入类工具的返回内容，一律写入 `review_queue`（见 §7 数据模型），`review_status=pending`，审核通过前对检索完全不可见（硬规则，见 `docs/DECISIONS.md`「Review-Gate 规则」）。
- **返回内容设计**：默认返回类型为 TextContent（Markdown），保证最低兼容性；当检索结果关联题目图片时（几何题的图形），追加 ImageContent（Base64 编码），供支持渲染的 Client 使用，不支持的 Client 可降级为纯文本展示。

### 4.4 可插拔架构

**目标：** 定义清晰的抽象层与接口契约，使核心组件能够独立替换与升级，避免技术锁定。与通用文档 RAG 一致，不做改动——这一层是纯技术范式，与业务场景无关。

**术语说明**：本节中的"提供者（Provider）"、"实现（Implementation）"指的是完成某项功能的**具体技术方案**，而非传统 Web 架构中的"后端服务器"。例如，LLM 提供者可以是远程的 Azure OpenAI API，也可以是本地运行的 Ollama；向量存储可以是本地嵌入式的 Chroma，也可以是云端托管的 Pinecone。本项目作为本地 MCP Server，通过统一接口对接这些不同的提供者，实现灵活切换。

设计原则（与通用文档 RAG 相同）：

- **接口隔离**：为每类组件定义最小化抽象接口，上层业务逻辑仅依赖接口，不依赖具体实现。
- **配置驱动**：通过 `config/settings.yaml` 指定各组件的具体后端，代码无需修改即可切换实现。
- **工厂模式**：工厂函数根据配置动态实例化对应实现类，一处配置、处处生效。
- **优雅降级**：首选后端不可用时，自动回退到备选方案或安全默认值。

**通用结构示意（适用于 LLM/Embedding、向量数据库、精排后端等各可插拔组件）**：

```
业务代码
  │
  ▼
<Component>Factory.get_xxx()  ← 读取配置，决定用哪个实现
  │
  ├─→ ImplementationA()
  ├─→ ImplementationB()
  └─→ ImplementationC()
      │
      ▼
    都实现了统一的抽象接口
```

各组件抽象（沿用原版设计，具体默认 Provider 为待解锁任务，见 §8 末尾）：

- **LLM / Embedding 提供者**：`BaseLLM`（`chat(messages) -> response`）、`BaseEmbedding`（`embed(texts) -> vectors`），统一屏蔽不同 Provider 的认证与请求格式差异。

  | 提供者类型 | 典型场景 | 配置切换点 |
  |---------|---------|-----------|
  | **Azure OpenAI** | 企业合规、私有云部署、区域数据驻留 | `provider: azure`, `endpoint`, `api_key`, `deployment_name` |
  | **OpenAI 原生** | 通用开发、最新模型尝鲜 | `provider: openai`, `api_key`, `model` |
  | **DeepSeek / 其他云端** | 成本优化、特定语言优化 | `provider: deepseek`, `api_key`, `model` |
  | **Ollama / vLLM（本地）** | 完全离线、隐私敏感、无 API 成本——教育机构本地部署场景下的重要选项 | `provider: ollama`, `base_url`, `model` |

  - **中间层取舍**：对于生产可靠性要求更高的场景，可在 Provider 抽象基础上增加统一的重试、限流、日志中间层；本项目当前规模（单机构/小流量）暂不实现，这里仅记录思路，留待真实并发压力出现时再引入。
- **Vision LLM 提供者**：`BaseVisionLLM`，支持文本+图片多模态输入，供 §4.1 `ImageLoader` 调用。这是本项目相对通用文档 RAG **优先级更高**的一环——通用文档 RAG 里 Vision LLM 只用于可选的图片描述增强，本项目里它是图片题目录入这一核心摄取路径的必需依赖。

**设计模式：抽象工厂模式**——向量数据库、精排后端等检索层组件的可插拔性依赖两层设计：（1）自研的统一抽象接口（`BaseVectorStore` 等），不同实现只需遵循相同接口即可无缝替换；（2）工厂函数路由（如 `vector_store_factory.py`），根据 `settings.yaml` 配置自动实例化对应实现，做到"改配置不改代码"。

- **向量数据库**：`BaseVectorStore`（`.add()`/`.query()`/`.delete()`），默认 **Chroma**。相比 Qdrant/Milvus/Weaviate 等需要 Docker 容器或分布式架构支撑的方案，Chroma 采用嵌入式设计，`pip install chromadb` 即可使用，无需额外部署数据库服务——契合本项目"教育机构本地可部署"的定位（与 §4.1 SQLite 选型的"本地优先"理念一致）。
- **向量编码策略**：可选纯稠密编码（Dense Only，仅语义向量，适合通用场景）、纯稀疏编码（Sparse Only，仅关键词权重，适合精确匹配）、双路编码（Dense + Sparse，为混合检索提供数据基础）。本项目采用**双路编码**（具体机制见 §4.1 Embedding 阶段），为 §4.2 的 Hybrid Search 提供数据基础；`reference_stem` 缺失时 Dense 向量本身不生成，属于查询时的降级，不影响摄取阶段仍对题面执行双路编码。
- **召回策略**：可选纯稠密召回（仅语义向量匹配）、纯稀疏召回（仅 BM25 关键词匹配）、混合召回（Dense+Sparse 并行+融合）、混合召回+精排。本项目采用**混合召回+精排**（完整机制见 §4.2 Hybrid Search Execution + Rerank），架构上预留了纯稠密/纯稀疏降级路径——这正是 `reference_stem` 缺失时 Dense Route 跳过、系统仍可用的技术基础。
- **精排后端（Reranker）**：见 §4.2，None / Cross-Encoder / LLM Rerank 三种，工厂路由 + Fallback 语义。
- **不需要 Splitter 抽象**：与通用文档 RAG 不同——本项目不做 Chunking（见 §4.1），因此不需要 `BaseSplitter`/`SplitterFactory` 这一层。通用文档 RAG 在这一层通常需要在固定长度切分/递归字符切分/语义切分/结构感知切分四种策略间权衡；本项目题目本身就是完整检索单元，这四种策略均无适用对象，不存在"选哪种分块"的决策。

**评估框架**：与通用文档 RAG 不同，本项目**主用自定义检索指标**（Hit Rate、MRR 等，衡量排序质量），不引入 Ragas——Ragas 的核心指标（Faithfulness、Answer Relevancy）面向"生成式回答质量"评测，本项目不做生成式回答，没有可用的评测对象。完整分析见 `docs/DECISIONS.md`「评估框架选型」。`BaseEvaluator` 抽象接口本身仍保留，暴露 `evaluate(query, retrieved_questions, ground_truth) -> metrics`（相比原版 `evaluate(query, retrieved_chunks, generated_answer, ground_truth)` 少一个 `generated_answer` 参数——本项目不做生成式回答，没有"生成内容"这个评测对象），为未来接入其他检索类评估指标留出扩展空间，但默认实现只需覆盖自定义检索指标。评估模块设计为**组合模式**，可同时挂载多个 Evaluator 并行执行、汇总结果——当前仅挂载自定义检索指标一项，但接口层面已支持未来追加其他评估维度而无需改动调用方代码。

**配置驱动示例**（`config/settings.yaml` 关键字段，与通用文档 RAG 结构一致，字段内容按本项目调整）：

```yaml
llm:
  provider: TODO  # azure | openai | ollama | deepseek，具体选型为待解锁任务
vision_llm:
  provider: TODO  # 图片题目录入依赖，优先级高于通用文档 RAG 场景
embedding:
  provider: TODO
vector_store:
  backend: chroma
retrieval:
  sparse_backend: bm25
  fusion_algorithm: rrf
  rerank_backend: none  # none | cross_encoder | llm
evaluation:
  backends: [custom_metrics]  # 不含 ragas，理由见 docs/DECISIONS.md
dashboard:
  enabled: true
  port: TODO  # 待解锁任务，见 §8 末尾
  traces_dir: ./logs
```

**切换流程**：

1. 修改 `settings.yaml` 中对应组件的 `backend`/`provider` 字段。
2. 确保新后端的依赖已安装、凭据已配置。
3. 重启服务，工厂函数自动加载新实现，无需修改业务代码。

### 4.5 可观测性与可视化管理平台

**目标：** 针对检索系统常见的"黑盒"问题，设计全链路追踪体系与可视化管理平台，覆盖 Ingestion（摄取）与 Query（检索）两条链路，同时承载 review-gate 审核这一本项目特有的核心人工流程。

设计理念（与通用文档 RAG 一致，不做改动）：双链路追踪、透明可回溯、低侵入性（`TraceContext` 显式调用）、轻量本地化（结构化日志 + Streamlit Dashboard，零外部依赖）、动态组件感知（Dashboard 基于 Trace 的 `method`/`provider` 字段动态渲染，换组件不用改 Dashboard 代码）。

**追踪数据结构（相对通用文档 RAG 的调整）**：

**基础信息字段（两类 Trace 共有）**：

- `trace_id: str`（请求/摄取唯一标识）
- `trace_type: Literal["query", "ingestion"]`
- `timestamp: datetime`
- `total_latency: float`（端到端总耗时）
- `error: Optional[str]`（异常信息，若有）

- **Query Trace**：阶段沿用 query_processing → dense → sparse → fusion → rerank，与 §4.2 一致。基础信息额外含 `processed_query: ProcessedQuery`（见 §4.2，取代通用文档 RAG 的 `user_query` 自由文本字段，因为本项目输入已结构化）。各阶段记录内容：

  | 阶段 | 记录内容 |
  |---|---|
  | query_processing | 输入的 `ProcessedQuery` 字段值、耗时 |
  | dense | Top-N 候选 `RetrievalResult` 列表（含 `stage_scores.dense`）、embedding provider、耗时 |
  | sparse | Top-N 候选 `RetrievalResult` 列表（含 `stage_scores.sparse`）、耗时 |
  | fusion | 融合后的统一排名（`RetrievalResult` 列表）、algorithm（rrf）、耗时 |
  | rerank | 重排后最终排名、backend、**`used_fallback: bool`、`fallback_reason: Optional[str]`**（§4.2 硬要求）、耗时 |

  汇总指标：`top_k_results: List[str]`（最终返回的题目 ID 列表）。去掉通用文档 RAG 的 `answer_faithfulness`（依赖生成环节，本项目不做生成回答）；`context_relevance` 类信号改为"题目与结构化错因条件的匹配度"，计算对象是 §4.2 `score_breakdown` 里的各分项。

- **Ingestion Trace**：阶段为 load → transform（仅图片场景，Vision LLM 感知转换）→ embed → upsert，**不含 split 阶段**（本项目不做 Chunking，见 §4.1）。基础信息额外含 `source_ref: str`（对应 `ingestion_records.source_ref`）、`source_type: str`。各阶段记录内容：

  | 阶段 | 记录内容 |
  |---|---|
  | load | 识别出的题目数、来源类型（method: pdf_loader / image_loader / manual_entry_loader）、耗时 |
  | transform | Vision LLM provider、处理的图片数、耗时（仅图片来源场景执行，其余来源跳过此阶段） |
  | embed | embedding provider、dense + sparse 编码耗时 |
  | upsert | 存储后端（method: chroma）、upsert 数量、幂等跳过数、耗时 |

  汇总指标：`total_questions: int`、`skipped: int`（review-gate 判定重复/已存在而跳过的数量）、`failed: int`。

**技术方案**：结构化日志（JSON Lines，`logs/traces.jsonl`）+ 本地 Streamlit Dashboard，与通用文档 RAG 一致——零外部依赖、单用户单机场景不需要分布式追踪。

**实现架构**：

```
检索/摄取 Pipeline
    │
    ▼
Trace Collector（TraceContext 显式调用）
    │
    ▼
JSON Lines 日志文件（logs/traces.jsonl）
    │
    ▼
本地 Web Dashboard（Streamlit）
    │
    ▼
按 trace_id 查看各阶段详情与性能指标
```

**`TraceContext` 机制**（与通用文档 RAG 完全一致，直接复用，因为这是纯工程范式，不涉及业务差异）：

1. **创建**：Pipeline/检索入口处创建 `TraceContext` 实例，生成唯一 `trace_id`，记录基础信息。
2. **阶段记录**：`TraceContext.record_stage(stage_name: str, method: str, details: dict, latency_ms: float)`——各阶段执行完毕后调用，记录该阶段的具体实现与细节数据（划分原则见下）。
3. **结束**：调用 `TraceContext.finish()`，序列化为 JSON，追加写入 `traces.jsonl`。
4. **调用约定**：显式调用模式——不强制、不会因未调用而报错，但依赖各可插拔组件的实现者在核心逻辑执行后主动调用 `record_stage()`。好处是代码透明，代价是需要开发者自觉遵守约定（与通用文档 RAG 的取舍一致）。

**阶段划分原则**：

- **Stage 是固定的通用大类**：`query_processing`/`dense`/`sparse`/`fusion`/`rerank`/`load`/`transform`/`embed`/`upsert`，不随具体可插拔实现变化。
- **具体实现是阶段内部的细节**：`record_stage()` 中通过 `method` 字段记录采用的具体方法（如 `bm25`、`cross_encoder`），通过 `details` 字段记录方法相关的细节数据。
- 这样无论底层可插拔组件怎么替换（换 Reranker、换 Embedding Provider），阶段结构保持稳定，Dashboard 展示逻辑无需调整——这是「阶段划分」与「组件实现」解耦的具体机制，对应 §4.4 可插拔架构的设计原则在可观测性层面的延伸。

**Dashboard 页面设计**：七页面，相对通用文档 RAG 的六页面做了三处调整——移除"评估后端选择 Ragas/Custom"（本项目评估只有 custom_metrics）、移除 Splitter 相关配置展示（不适用）、新增"待审核队列"页承载 review-gate 流程。

| 页面 | 核心功能 | 与通用文档 RAG 的差异 |
|---|---|---|
| 1. 系统总览 | 组件配置卡片、数据资产统计（题目数/章节分布）、最近 Trace 时间 | 组件卡片不展示 Splitter 配置 |
| 2. 题库浏览器 | 浏览已入库题目（题面/答案/关联标准步骤/难度），按章节/难度筛选 | 对应通用文档 RAG 的"数据浏览器"，浏览对象从文档 Chunk 换成完整题目（无 Chunk 概念） |
| 3. 数据摄取管理 | 触发三种 Loader（PDF/图片/手动表单）摄取，进度条，文档删除 | 摄取入口从单一文件上传扩展为三种来源 |
| 4. 待审核队列 | Tab A：`review_queue` 待审提议（标准步骤/错因标签/题目路径映射），approve/reject/edit；Tab B：已批准的 taxonomy（标准步骤库/错因库）只读浏览，供审核时对照 | **本项目新增**，通用文档 RAG 无对应页面（无人工审核流程） |
| 5. 评估面板 | Tab A：`submit_recommendation_feedback` 收集的反馈汇总；Tab B：运行自定义检索指标（Hit Rate/MRR）、历史趋势 | 去除 Ragas 选项；新增反馈汇总 Tab；当前功能较薄，排序权重仍是待解锁任务 |
| 6. Ingestion 追踪 | 摄取历史、阶段耗时瀑布图、失败详情 | 阶段列表随 §4.1 调整（无 split） |
| 7. Query 追踪 | 查询历史、Dense/Sparse 对比、Rerank 前后排名变化、Fallback 触发记录 | 新增 Fallback 触发的显式展示 |

**Dashboard 与 Trace 的数据关系**：页面 6/7 读取 `traces.jsonl`；页面 1/2/3 直接读取存储层；页面 4/5 直接读取 `review_queue`/反馈记录表，不依赖 Trace。所有页面基于 Trace 中 `method`/`provider` 字段动态渲染，更换可插拔组件后自动适配，无需改 Dashboard 代码。

**配置示例**：

```yaml
observability:
  enabled: true

  # 日志配置
  logging:
    log_file: logs/traces.jsonl  # JSON Lines 格式日志文件
    log_level: INFO  # DEBUG | INFO | WARNING

  # 追踪粒度控制
  detail_level: standard  # minimal | standard | verbose

# Dashboard 管理平台配置（见 §4.4 配置示例的 dashboard 字段）
dashboard:
  enabled: true
  port: TODO  # 待解锁任务，见 §8 末尾
  traces_dir: ./logs
  auto_refresh: true       # 是否自动刷新（轮询新 trace）
  refresh_interval: 5      # 自动刷新间隔（秒）
```

**Dashboard 技术架构**（目录结构与通用文档 RAG 一致，页面文件按本项目七页面调整）：

```
src/observability/dashboard/
├── app.py                       # Streamlit 入口，页面导航注册
├── pages/
│   ├── overview.py               # 页面 1：系统总览
│   ├── question_browser.py       # 页面 2：题库浏览器
│   ├── ingestion_manager.py      # 页面 3：数据摄取管理
│   ├── review_queue.py           # 页面 4：待审核队列（Tab A 待审提议 / Tab B taxonomy 浏览）
│   ├── evaluation_panel.py       # 页面 5：评估面板（Tab A 反馈汇总 / Tab B 指标运行）
│   ├── ingestion_traces.py       # 页面 6：Ingestion 追踪
│   └── query_traces.py           # 页面 7：Query 追踪
└── services/
    ├── trace_service.py          # Trace 数据读取服务（解析 traces.jsonl）
    ├── data_service.py           # 数据浏览服务（封装 VectorStore 读取）
    ├── review_service.py         # review_queue 读写服务（本项目新增，通用文档 RAG 无对应）
    └── config_service.py         # 配置读取服务（封装 Settings 读取与展示）
```

---

### 4.6 多模态图片处理

**目标：** 设计一套完整的图片处理方案，使检索系统能够理解、结构化并索引图片中的题目内容，实现"拍照录题"能力，同时保持架构的简洁性与可扩展性。

**设计理念与策略选型**：

多模态处理的核心挑战在于：**如何让纯文本的检索系统"看懂"图片**。业界主要有两种技术路线：

| 策略 | 核心思路 | 优势 | 劣势 |
|-----|---------|------|------|
| **Image-to-Text（图转文）** | 利用 Vision LLM 将图片转化为结构化文本，复用纯文本检索链路 | 架构统一、实现简单、成本可控 | 识别质量依赖 LLM 能力，可能丢失视觉细节 |
| **Multi-Embedding（多模态向量）** | 使用 CLIP 等模型将图文统一映射到同一向量空间 | 保留原始视觉特征，支持"图搜图" | 需引入额外向量库，架构复杂度高 |

**本项目选型：Image-to-Text（图转文）策略**，即 §4.1 `ImageLoader` 已经采用的路径。选型理由：
- **业务场景不需要"图搜图"**：本项目的检索目标是"错因相同的题"，不是"长得像的图"——学生不会有"找一道和这张图相似的题"这类需求，Multi-Embedding 解决的核心问题在本项目里不存在真实需求，不构成技术上的取舍空间。
- **架构统一**：无需引入 CLIP 等多模态 Embedding 模型，无需维护独立的图像向量库，完全复用现有的文本检索链路（Ingestion → Hybrid Search → Rerank）。
- **语义对齐**：Vision LLM 把图片中的题目转化为结构化文本后，天然可以复用 §4.2 的标准步骤匹配、错因标签匹配等结构化检索能力。
- **成本可控**：仅在数据摄取阶段一次性调用 Vision LLM，检索阶段无额外成本。

**图片处理全流程设计**（比通用文档 RAG 少一环——没有 Splitter，见 §4.1"不做 Chunking"）：

```
原始输入（学生拍照 / 老师批量扫描）
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  Loader 阶段：图片识别与 image_id 生成                      │
│  - ImageLoader 调用 Vision LLM 识别图片中的题目边界与内容    │
│  - 为每张图片生成唯一标识 image_id                          │
│  - 输出：List[Question]（见 §4.1，每道题携带 image_ref）    │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  Transform 阶段：结构化识别与质量判定                        │
│  - Vision LLM 分类型识别（题面文字 / 几何图形结构）           │
│  - 识别置信度判定，低于阈值标记 recognition_confidence       │
│  - 输出：结构清洗后的 Question（见 §4.1 Transform 阶段）      │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  Embedding 阶段：与普通题目一致                              │
│  - 结构化后的题面文本走 §4.1 标准双路编码流程                 │
│  - 图片本身不参与向量化，只有转换后的文本参与                  │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  Storage 阶段：双轨存储                                     │
│  - 向量库：存储结构化后的题目文本，用于检索                    │
│  - 图片索引（image_index.db，见 §4.1）：存储原始图片文件路径，  │
│    用于检索命中后返回图形给辅导讲师 Agent                      │
└─────────────────────────────────────────────────────────┘
```

**各阶段技术要点**：

**1. Loader 阶段：image_id 生成与识别边界**

- `image_id` 生成规则：`{content_hash}_{sequence}`（`content_hash` 为图片内容哈希，`sequence` 为同一输入中的题目序号）——与 §4.1 幂等性设计一致，同一图片重复上传产生相同 `image_id`。
- 识别边界与 §4.1"多题目边界识别机制"共用同一套 `source_position` 定位逻辑，不重复设计。

**2. Transform 阶段：Vision LLM 选型与分类型识别**

- **Vision LLM 选型策略**：不采用参考项目"国内+国外双模型、按部署环境切换"的地域二分方案——本项目单机构本地部署，不存在合规/地域驱动的切换需求。改为按**任务类型**拆分可插拔维度：文字 OCR（题目文字、手写体）与数学符号/几何图形结构理解分别对应不同的模型能力要求，具体启用哪些模型、路由规则如何设计，为待解锁任务（见 §8 末尾），需要真实题目样本测试后才能拍板。完整选型分析见 `docs/DECISIONS.md`「Vision LLM 选型策略」。
- **分类型识别（对应通用文档 RAG 的"分类型 Prompt 处理"，本项目按题目结构拆分）**：
  - **题面文字**：识别重点是文字准确率，尤其手写体与扫描件模糊字迹。
  - **几何图形结构**：识别重点是图形元素间的关系（全等对应边、角度标注、辅助线），这是本项目区别于通用文档 RAG 图表理解的核心难点——错误的图形结构理解可能导致题目被关联到错误的标准步骤。
- **幂等与增量处理**：为每张图片的识别结果计算内容哈希，若图片内容未变且 Prompt/模型版本一致，直接复用已有识别结果，避免重复调用 Vision LLM（与 §4.1 Embedding 差量计算的设计理念一致）。
- **大尺寸图片压缩**：学生拍照场景常见超大尺寸图片，传入 Vision LLM 前按比例压缩（保持宽高比，限制最大边长），控制 API 调用成本与延迟。

**3. Storage 阶段：双轨存储**

- 与 §4.1 Upsert 阶段一致：向量库存储结构化后的题目文本（用于检索），`image_index.db`（见 §4.1 SQLite 统一说明）存储原始图片文件路径（用于命中后返回图形）。

**检索与返回流程**：

当检索命中包含图形的题目（几何题）时，系统需要将图形与文本一并返回：

```
辅导讲师 Agent 调用 search_questions
    │
    ▼
Hybrid Search 命中题目（question.image_ref 非空）
    │
    ▼
查询 image_index.db，获取图片文件路径
    │
    ▼
读取图片文件，编码为 Base64
    │
    ▼
构造 MCP 响应，包含 TextContent + ImageContent（见 §4.3 返回内容设计）
```

**质量保障与边界处理**：

- **识别质量检测**：对识别结果进行基础质量检查（题面是否完整、是否包含关键数值/图形描述）。若识别置信度过低或 Vision LLM 返回"无法识别"，标记 `recognition_confidence` 低于阈值，进入 §4.1 描述的 `review_status="pending"` 待审核流程，不静默丢弃（见 §4.1 多题目边界识别机制）。
- **批量处理优化**：图片识别支持批量异步调用，提高老师批量扫描场景的吞吐量。单张图片识别失败不阻塞同批次其他图片的处理（与 §4.1 Transform 阶段的原子化设计一致）。
- **降级策略（Vision LLM 不可用时）**：当 Vision LLM 服务不可用或调用失败时，系统不阻塞整个摄取流程——该图片对应的 `Question` 记录以 `metadata.transform_failed=true`、`review_status="pending"` 的形式入库（复用 §4.1 Transform 阶段已有的失败处理机制），`stem` 字段留空或标记"待人工补充"，由老师在待审核队列页手动补全题面后再审核通过。这确保 Vision LLM 的可用性问题不会导致学生拍照录题这一核心入口完全失效。

---

## 5. 测试方案

### 5.1 设计理念：测试驱动开发（TDD）

本项目采用**测试驱动开发（Test-Driven Development）**作为核心开发范式，确保每个组件在实现前就已明确其预期行为，通过自动化测试持续验证系统质量。

**核心原则**：
- **早测试、常测试**：每个功能模块实现的同时就编写对应的单元测试，而非事后补测。
- **测试即文档**：测试用例本身就是最准确的行为规范，新加入的开发者可通过阅读测试快速理解各模块功能——对本项目而言尤其重要，因为 Review-Gate、多路径匹配这类业务规则单靠代码不容易一眼看出边界条件。
- **快速反馈循环**：单元测试应在秒级完成，支持开发者高频执行，立即发现引入的问题。
- **分层测试金字塔**：大量快速的单元测试作为基座，少量关键路径的集成测试作为保障，极少数端到端测试验证完整流程。

```
        /\
       /E2E\         <- 少量，验证关键业务流程
      /------\
     /Integration\   <- 中量，验证模块协作
    /------------\
   /  Unit Tests  \  <- 大量，验证单个函数/类
  /________________\
```

### 5.2 测试分层策略

#### 5.2.1 单元测试（Unit Tests）

**目标**：验证每个独立组件的内部逻辑正确性，隔离外部依赖。

**覆盖范围**：

| 模块 | 测试重点 | 典型测试用例 |
|-----|---------|------------|
| **Loader（PdfLoader/ImageLoader/ManualEntryLoader）** | 多题目边界识别、`Question` 对象产出、置信度标记 | - 测试单页/多页 PDF 中多道题目的边界切分<br>- 验证 `ImageLoader` 对低置信度识别结果标记 `recognition_confidence` 而非丢弃<br>- 检查 `source_position` 字段（`page`/`sequence`）正确性 |
| **Transform（结构清洗 + Vision LLM 感知转换）** | 题面去噪、语义元数据注入、幂等性 | - 验证规则去噪不破坏合法题面内容<br>- Mock Vision LLM，验证 `topic_tags` 注入逻辑<br>- 验证 Vision LLM 不可用时的降级路径（`transform_failed=true`，见 §4.6）<br>- 验证幂等性（重复处理相同内容哈希不重复调用） |
| **Embedding（双路向量化）** | 差量计算、批处理、Dense/Sparse 双路生成 | - 验证相同题面生成相同向量<br>- 测试批量请求的拆分与合并（`batch_size` 驱动）<br>- 检查内容哈希缓存命中逻辑，避免重复计费 |
| **BM25（稀疏编码）** | 关键词提取、权重计算 | - 验证中文分词与停用词过滤<br>- 测试 IDF 计算准确性<br>- 检查稀疏向量格式 |
| **Retrieval（检索器）** | 结构化过滤、召回精度、融合算法、多路径加分 | - 测试 `reference_stem` 存在/缺失两种场景下的 Dense Route 执行/跳过<br>- 验证 RRF 融合分数计算<br>- 验证单路降级场景（Dense 缺失时直接使用 Sparse 排名，不套 RRF 公式，见 §4.2）<br>- 验证多路径命中加分幅度受控（不压过标准步骤/错因匹配等主信号权重，见 `docs/DECISIONS.md`「检索排序：多路径命中加分机制」） |
| **Reranker（重排器）** | 分数归一化、Fallback 回退 | - Mock Cross-Encoder，验证分数重排<br>- 测试超时后的 Fallback 逻辑，且返回结果显式标记 `used_fallback`/`fallback_reason`（见 §4.2，不允许静默降级）<br>- 验证空候选集处理 |
| **Review-Gate 过滤** | 待审内容不可见性 | - 验证 `review_status="pending"` 的题目/标准步骤/错因标签不出现在任何检索结果中<br>- 验证审核通过后立即可被检索命中，不需要重建索引 |

**技术选型**：
- **测试框架**：`pytest`（Python 标准选择，支持参数化测试、Fixture 机制）
- **Mock 工具**：`unittest.mock` / `pytest-mock`（隔离外部依赖，如 Vision LLM、Embedding API）
- **断言增强**：`pytest-check`（支持多断言不中断执行）

#### 5.2.2 集成测试（Integration Tests）

**目标**：验证多个组件协作时的数据流转与接口兼容性。

**覆盖范围**：

| 测试场景 | 验证要点 | 测试策略 |
|---------|---------|---------|
| **Ingestion Pipeline** | Loader → Transform → Embedding → Upsert 的完整流程（无 Splitter 环节） | - 使用真实的测试 PDF 题库文件与题目图片<br>- 验证最终存入向量库的 `QuestionRecord` 数据完整性<br>- 检查 `ingestion_records`/`ingestion_history` 记录是否正确写入 |
| **Hybrid Search** | Dense + Sparse 召回的融合结果，含 `reference_stem` 有无两种路径 | - 准备已知标准步骤的题目样本<br>- 验证 `reference_stem` 提供时融合后的 Top-1 是否命中语义相近题目<br>- 验证 `reference_stem` 缺失时降级为纯结构化过滤排序仍能返回合理结果 |
| **Rerank Pipeline** | 结构化过滤 → 融合 → 精排的组合 | - 验证 `misconception_tag_id` 缺失时的"宽松包含"后置过滤逻辑<br>- 检查 Reranker 是否改变了 Top-1 结果<br>- 测试 Reranker 失败时的 Fallback 触发与标记 |
| **MCP Server** | 7 个工具（查询/检索/图片感知转换/提议写入/反馈回流五类）的端到端流程 | - 模拟 MCP Client 发送 JSON-RPC 请求<br>- 验证 `search_questions` 返回的 `content`/`structuredContent` 格式符合 §4.3 设计<br>- 验证 `propose_*` 类工具写入后进入 `review_queue` 且 `review_status=pending`<br>- 测试错误处理（如传入不存在的 `canonical_step_id`，见 §4.2"不做模糊匹配或猜测"） |

**技术选型**：
- **数据隔离**：每个测试使用独立的临时 SQLite 库/向量库（`pytest-tempdir`），覆盖 `ingestion_history.db`/`image_index.db` 等（见 §4.1 SQLite 持久化存储架构统一说明）
- **异步测试**：`pytest-asyncio`（若 MCP Server 采用异步实现）
- **契约测试**：定义 `ProcessedQuery`/`RetrievalResult`/`Question` 等核心数据类型的 Schema，确保接口不漂移

#### 5.2.3 端到端测试（End-to-End Tests）

**目标**：模拟真实用户操作，验证完整业务流程的可用性。

**核心场景**：

**场景 1：数据摄取（三种来源）**
- **测试目标**：验证图片/PDF/手动表单三种录入路径的完整性与正确性
- **测试步骤**：
  - 准备测试素材：几何证明题图片（含手写体）、PDF 题库文件、手动表单数据
  - 分别执行三种 Loader 的摄取流程
  - 验证摄取结果：检查识别出的题目数量、`metadata` 完整性（`chapter_code`/`difficulty`/`review_status`）、图片关联（`image_ref`）
  - 验证存储状态：确认向量库、BM25 索引、`ingestion_records` 正确创建
  - 验证幂等性：重复摄取同一图片/PDF，确保不产生重复题目（`id` 基于内容哈希）
- **验证要点**：
  - 多题目边界识别质量（一份输入中多道题目是否被正确拆分，见 §4.1）
  - `metadata` 字段完整性
  - Vision LLM 识别结果（含低置信度标记与降级路径，见 §4.6）
  - 向量与稀疏索引的正确性

**场景 2：召回测试**
- **测试目标**：验证检索系统在结构化输入下的召回精度与排序质量
- **测试步骤**：
  - 基于已摄取的题库，准备一组测试 `ProcessedQuery`（覆盖不同 `canonical_step_id`/`misconception_tag_id`/`reference_stem` 组合）
  - 执行混合检索（Dense + Sparse + Rerank）
  - 验证召回结果：检查 Top-K 题目是否命中预期标准步骤
  - 对比 `reference_stem` 提供 vs 缺失两种场景下的召回质量差异
  - 验证多路径命中题目是否获得合理加分，且未压过主信号排序
- **验证要点**：
  - Hit Rate@K：Top-K 结果命中预期标准步骤的比率是否达标
  - 排序质量：`score_breakdown` 各分项是否合理（MRR）
  - 边界情况处理：不存在的 `canonical_step_id`、空结果、`exclude_question_ids` 全量排除后的场景
  - Review-Gate 边界：待审核题目在任何 `ProcessedQuery` 下都不应出现在结果中

**场景 3：MCP Client 功能测试（模拟辅导讲师 Agent）**
- **测试目标**：验证 MCP Server 与辅导讲师 Agent 的协议兼容性与功能完整性
- **测试步骤**：
  - 启动 MCP Server（Stdio Transport 模式）
  - 模拟 MCP Client 依次调用五类工具：`lookup_canonical_steps`、`search_questions`、`ingest_question_from_image`、`propose_question_mapping`、`submit_recommendation_feedback`
  - 验证返回格式：符合 §4.3 协议规范（`content` 数组、`structuredContent` 中的 `score_breakdown`/`why_recommended`）
  - 测试提议写入类工具只返回简洁确认（成功与否 + 记录 ID），不返回审核状态
  - 测试多模态返回：含图形的题目响应正确编码为 Base64 ImageContent
- **验证要点**：
  - 协议合规性：JSON-RPC 2.0 格式、错误码映射
  - 工具注册：`tools/list` 返回全部 7 个工具及其 Schema
  - 响应格式：TextContent 与 ImageContent 的正确组合
  - 错误处理：无效 `step_ids`、`image_data` 格式错误、Vision LLM 超时等异常场景
  - Review-Gate 端到端验证：`propose_question_mapping` 写入后，同一题目立即通过 `search_questions` 检索确认仍不可见，审核通过后才可见

**测试工具**：
- **BDD 框架**：`behave` 或 `pytest-bdd`（以 Gherkin 语法描述场景）
- **环境准备**：
  - 临时测试向量库（独立于生产数据）
  - 预置的标准测试题库集（覆盖几何证明类核心标准步骤与错因标签）
  - 本地 MCP Server 进程（Stdio Transport）

### 5.3 检索质量评估测试

**目标**：验证已设计的评估体系（见 §4.4 评估框架抽象）是否正确实现，并能有效评估检索排序质量。

**测试要点**：

1. **黄金测试集准备**
   - 构建标准的"结构化错因输入 → 预期命中题目"测试集（JSON 格式），覆盖不同章节、难度、单路径/多路径题目
   - 初期人工标注核心标准步骤场景，后期持续积累老师审核中发现的坏 Case

2. **评估框架实现验证**
   - 验证自定义检索指标（Hit Rate、MRR）的正确实现（不含 Ragas，理由见 `docs/DECISIONS.md`「评估框架选型」）
   - 确认 `BaseEvaluator.evaluate(query, retrieved_questions, ground_truth)` 接口能输出标准化的指标字典
   - 测试组合模式：未来追加其他 Evaluator 时能与现有指标并行执行、汇总结果（见 §4.4）

3. **关键指标达标验证**
   - 检索指标：Hit Rate@K、MRR 具体达标线为待解锁任务（见 §8 末尾）——现阶段没有真实使用数据支撑具体数值，写死属于虚假精确
   - 定期运行评估，监控指标是否随排序权重调整（多路径加分系数等，见 §4.2）而回归

**说明**：本节重点是验证评估体系的工程实现，而非重新设计评估方法（评估方法的设计见 §4.4 技术选型）。不含生成质量指标（Faithfulness/Answer Relevancy）——本项目不做生成式回答，没有可评测对象。

### 5.4 性能与压力测试（可选）

> **说明**：本项目定位为本地 MCP Server，单机构部署，采用 Stdio Transport 通信方式。性能与压力测试在当前阶段**不是必需的**，此处列出主要用于：
> 1. **架构完整性**：展示完整的工程化测试体系，体现系统设计的专业性
> 2. **未来扩展性**：若后续需要支持多机构部署或云端托管，可直接参考此方案
> 3. **性能基准建立**：通过基础性能测试了解系统瓶颈，为优化提供数据支撑

**可选测试场景**：

| 测试类型 | 验证点 | 工具 | 优先级 |
|---------|-------|------|-------|
| **延迟测试** | 单次检索的 P50/P95/P99 延迟 | `pytest-benchmark` | 中（辅导讲师 Agent 实时对话场景对延迟敏感，见 §4.1 Upsert All-in-One 存储策略的设计动机） |
| **吞吐量测试** | 批量导入场景的并发写入上限 | `locust` | 低（单机构、老师批量导入非高频场景） |
| **内存泄漏检测** | 长时间运行后的内存占用 | `memory_profiler` | 低（短期运行无影响） |
| **向量库性能** | 不同题目规模下的查询速度 | 自定义 Benchmark | 中（验证题库规模扩展后的可用性） |

### 5.5 测试工具链与 CI/CD 集成

**本地开发工作流**：
- **快速验证**：仅运行单元测试，秒级反馈
- **完整验证**：单元测试 + 集成测试，生成覆盖率报告
- **质量评估**：定期执行检索质量测试，监控指标变化

**CI/CD Pipeline 设计**（可选）：
> **说明**：本地项目不强制要求 CI/CD，但配置自动化测试流程有助于代码质量保障与持续集成实践。

- **单元测试阶段**：每次提交自动触发，验证基础功能，生成覆盖率报告
- **集成测试阶段**：单元测试通过后执行，验证模块协作
- **质量评估阶段**：PR 触发，运行完整的检索质量测试，发布评估报告

**测试覆盖率目标**：
- **单元测试**：核心逻辑覆盖率 ≥ 80%
- **集成测试**：关键路径覆盖率 100%（如 Ingestion、Hybrid Search、Review-Gate）
- **E2E 测试**：核心用户场景覆盖率 100%（三种摄取来源 + 召回 + MCP Client，至少 3 个关键流程）

---

## 6. 系统架构与模块设计

### 6.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                  MCP Clients (外部调用层)                                     │
│                          ┌─────────────────────────────────┐                                │
│                          │       辅导讲师 Agent              │                                │
│                          │  (唯一确定 Client，见 §4.3)        │                                │
│                          └────────────────┬────────────────┘                                │
│                                           │  JSON-RPC 2.0 (Stdio Transport)                  │
└────────────────────────────────────────────┼──────────────────────────────────────────────────┘
                                             ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   MCP Server 层 (接口层)                                      │
│    ┌─────────────────────────────────────────────────────────────────────────────────┐      │
│    │                              MCP Protocol Handler (tools/list, tools/call)       │      │
│    └─────────────────────────────────────────────────────────────────────────────────┘      │
│  ┌──────────┬──────────────┬──────────────────┬──────────────┬──────────────┐               │
│┌────────┐┌──────────┐┌──────────────────┐┌──────────────┐┌──────────────┐                   │
││ 查询    ││ 检索      ││ 图片感知转换       ││ 提议写入 ×3   ││ 反馈回流      │                   │
││lookup_ ││search_   ││ingest_question_  ││propose_*     ││submit_       │                   │
││steps   ││questions ││from_image        ││              ││feedback      │                   │
│└────────┘└──────────┘└──────────────────┘└──────────────┘└──────────────┘                   │
│  五个分类走向不同：查询/检索 → 下方 Core 层 Retrieval Engine；                                 │
│  图片感知转换 → 触发下方 Ingestion Pipeline 的 ImageLoader 路径（实时摄取，不经过 Retrieval）；  │
│  提议写入×3/反馈回流 → 直接旁路写入 Storage 层 review_queue / 反馈记录表，不经过 Retrieval Engine │
└────────────────────────────────────────┬────────────────────────────────────────────────────┘
                                         │ （查询/检索两类走此路径）
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   Core 层 (核心业务逻辑)                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐    │
│  │                          Retrieval Engine (检索引擎，见 §4.2)                         │    │
│  │  ┌─────────────────────────────────────────────────────────────────────────────┐    │    │
│  │  │              ProcessedQuery（canonical_step_id 必填 + reference_stem 可选）    │    │    │
│  │  └─────────────────────────────────────────────────────────────────────────────┘    │    │
│  │  ┌─────────────────────────────────────────────────────────────────────────────┐    │    │
│  │  │   结构化前置过滤（review_status=approved 硬过滤 / chapter_code / canonical_    │    │    │
│  │  │   step_id，索引层面可精确支持，缩小候选集）                                     │    │    │
│  │  └─────────────────────────────────────────────────────────────────────────────┘    │    │
│  │  ┌────────────────────────────────────┼────────────────────────────────────┐        │    │
│  │  │                       Hybrid Search Engine                               │        │    │
│  │  │    ┌───────────────────┐    ┌──────┴──────┐    ┌───────────────────┐    │        │    │
│  │  │    │   Dense Route     │    │   Fusion    │    │   Sparse Route    │    │        │    │
│  │  │    │ 仅 reference_stem │◄───┤    (RRF)    ├───►│ reference_stem存在:│    │        │    │
│  │  │    │ 存在时执行         │    │             │    │  BM25关键词检索    │    │        │    │
│  │  │    │                   │    │  单路降级:   │    │ reference_stem缺: │    │        │    │
│  │  │    │                   │    │Dense缺失→直接│    │  退化为纯结构化    │    │        │    │
│  │  │    │                   │    │用Sparse排名, │    │  过滤，不打分      │    │        │    │
│  │  │    │                   │    │不套RRF公式   │    │                   │    │        │    │
│  │  │    └───────────────────┘    └─────────────┘    └───────────────────┘    │        │    │
│  │  └─────────────────────────────────────────────────────────────────────────┘        │    │
│  │  ┌─────────────────────────────────────────────────────────────────────────────┐    │    │
│  │  │   结构化后置过滤（misconception_tag_id，safety net，missing→宽松包含，         │    │    │
│  │  │   避免因标注缺失误杀候选）                                                     │    │    │
│  │  └─────────────────────────────────────────────────────────────────────────────┘    │    │
│  │  ┌─────────────────────────────────────────────────────────────────────────────┐    │    │
│  │  │  Rerank：标准步骤/错因匹配 + 题面相似 + 多路径命中加分（受控幅度）+ difficulty │    │    │
│  │  │  （软偏好信号，非硬过滤）。后端 None/Cross-Encoder/LLM；Fallback 触发时结果    │    │    │
│  │  │  必须显式标记 used_fallback/fallback_reason（契约字段，非内部细节）           │    │    │
│  │  └─────────────────────────────────────────────────────────────────────────────┘    │    │
│  │  ┌─────────────────────────────────────────────────────────────────────────────┐    │    │
│  │  │           Response Builder（score_breakdown / why_recommended 组装，见§4.3） │    │    │
│  │  └─────────────────────────────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────────────────────┘    │
│                                       ▼ 每个阶段各自调用 record_stage()（非流程末尾统一记录）   │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐    │
│  │              TraceContext（创建 → 各阶段 record_stage() → finish()，见 §4.5）         │    │
│  │      Query Trace：query_processing/dense/sparse/fusion/rerank 各阶段独立记录          │    │
│  └─────────────────────────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────┬────────────────────────────────────────────────────┘
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   Storage 层 (存储层)                                        │
│    ┌─────────────────────────────────────────────────────────────────────────────────┐      │
│    │                             Vector Store (向量存储)                              │      │
│    │     ┌─────────────────────────────────────────────────────────────────────┐     │      │
│    │     │                         Chroma DB                                   │     │      │
│    │     │  Dense Vector | Question 原文 | Metadata（含 review_status，前置      │     │      │
│    │     │  过滤依据）                                                          │     │      │
│    │     └─────────────────────────────────────────────────────────────────────┘     │      │
│    └─────────────────────────────────────────────────────────────────────────────────┘      │
│    ┌──────────────────────────────────────────────────────────────────────────────────┐     │
│    │  BM25 Index（独立倒排索引 + IDF 统计，驱动 Sparse Route 检索计算；Sparse Vector    │     │
│    │  数值本身也持久化在 Chroma 作为 payload，两处不冲突，分工不同——见 §4.1）            │     │
│    └──────────────────────────────────────────────────────────────────────────────────┘     │
│    ┌──────────────────────────────────┐    ┌──────────────────────────────────┐             │
│    │       Image Store (图片存储)      │    │  review_queue（旁路写入，待审提议，│             │
│    │    本地文件系统 | Base64 编码     │    │   独立表，见 §7 数据模型）         │             │
│    └──────────────────────────────────┘    └──────────────────────────────────┘             │
│    ┌──────────────────────────────────────────────────────────────────────────────────┐     │
│    │                     Trace Logs (追踪日志，JSON Lines 格式文件)                       │     │
│    │       Query Trace（来自上方 Core 层 TraceContext）+                                 │     │
│    │       Ingestion Trace（来自下方 Ingestion Pipeline，双链路追踪的另一条链路）           │     │
│    └──────────────────────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│           Ingestion Pipeline (离线批量导入 / 实时拍照录题共用同一流程，见 §4.1)                  │
│           触发入口：MCP Server「图片感知转换」工具（实时单条）+ 后台管理员批量导入（多条）           │
│    ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐                    │
│    │   Loader   │───►│ Transform  │───►│  Embedding │───►│   Upsert   │                    │
│    │(三种来源)  │    │ (结构清洗) │    │  (双路向量) │    │  (存储)    │                    │
│    └────────────┘    └────────────┘    └────────────┘    └────────────┘                    │
│         │                  │                  │                │                            │
│         ▼                  ▼                  ▼                ▼                            │
│    ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐                    │
│    │PdfLoader   │    │Vision LLM  │    │Dense:      │    │Chroma      │                    │
│    │ImageLoader │    │感知转换    │    │OpenAI/BGE  │    │All-in-One  │                    │
│    │ManualEntry │    │语义元数据  │    │Sparse:BM25 │    │幂等写入    │                    │
│    └────────────┘    └────────────┘    └────────────┘    └────────────┘                    │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
（无 Splitter 阶段——本项目不做 Chunking，见 §4.1；Upsert 阶段同时写入上方 Storage 层的 Trace Logs）

┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                Libs 层 (可插拔抽象层，见 §4.4)                                 │
│    ┌────────────────────────────────────────────────────────────────────────────────┐       │
│    │                            Factory Pattern (抽象工厂模式)                        │       │
│    └────────────────────────────────────────────────────────────────────────────────┘       │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐  │
│  │ LLM Client │ │Vision LLM  │ │ Embedding  │ │VectorStore │ │  Reranker  │ │ Evaluator  │  │
│  │  Factory   │ │  Factory   │ │  Factory   │ │  Factory   │ │  Factory   │ │  Factory   │  │
│  ├────────────┤ ├────────────┤ ├────────────┤ ├────────────┤ ├────────────┤ ├────────────┤  │
│  │ · Azure    │ │·独立抽象接口│·OpenAI      │ │ · Chroma   │ │ · None     │ │ · Custom   │  │
│  │ · OpenAI   │ │ BaseVisionLLM│·BGE        │ │ · Qdrant   │ │ · CrossEnc │ │  Metrics   │  │
│  │ · Ollama   │ │（非LLM子选项，│·Ollama     │ │ · ...      │ │ · LLM      │ │  (不含     │  │
│  │ · DeepSeek │ │ 与LLM并列）  │·...        │ │            │ │            │ │   Ragas)   │  │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘ └────────────┘ └────────────┘  │
│  （无 Splitter Factory——本项目不做 Chunking）                                                │
└─────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                             Observability 层 (可观测性，见 §4.5)                              │
│    ┌──────────────────────────────────────┐    ┌──────────────────────────────────────┐     │
│    │  与 Core 层 TraceContext 是同一套机制  │    │         Web Dashboard                │     │
│    │  ——此处为管理平台读取视角：解析        │    │        (Streamlit，七页面)            │     │
│    │  traces.jsonl 供 Dashboard 展示        │    │  含待审核队列 + 反馈汇总（本项目新增） │     │
│    └──────────────────────────────────────┘    └──────────────────────────────────────┘     │
│    ┌──────────────────────────────────────┐    ┌──────────────────────────────────────┐     │
│    │          Evaluation Module           │    │         Structured Logger            │     │
│    │   Hit Rate | MRR（不含 Faithfulness） │    │    JSON Formatter | File Handler     │     │
│    └──────────────────────────────────────┘    └──────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

**与通用文档 RAG 架构图的关键差异**：

- MCP Clients 层从多 Client 并列简化为单一确定 Client（辅导讲师 Agent），见 §4.3。
- MCP Server 层工具固定为 7 个、分五类（查询/检索/图片感知转换/提议写入×3/反馈回流），见 §4.3。
- Core 层无 Query Processor（输入已结构化，见 §4.2），`ProcessedQuery` 直接进入 Multi-stage Filtering；Dense Route 仅在 `reference_stem` 存在时执行，是本项目独有的降级机制。
- Storage 层新增 `review_queue`，替代通用文档 RAG 的 Processing Cache（本项目无 Chunk 级处理缓存）。
- Ingestion Pipeline 无 Splitter 环节，Loader 从单一 PDF 扩展为三种来源。
- Libs 层无 Splitter Factory；Evaluator Factory 只含自定义指标，不含 Ragas/DeepEval。

---

### 6.2 目录结构

```
rag-math-qb/
│
├── config/                              # 配置文件目录
│   ├── settings.yaml                    # 主配置文件 (LLM/Vision LLM/Embedding/VectorStore 配置，见 §4.4)
│   └── prompts/                         # Prompt 模板目录
│       ├── image_recognition.txt        # 图片题目识别 Prompt（分类型：文字OCR/几何图形结构，见 §4.6）
│       └── rerank.txt                   # LLM Rerank Prompt
│
├── src/                                 # 源代码主目录
│   │
│   ├── mcp_server/                      # MCP Server 层 (接口层，见 §4.3)
│   │   ├── __init__.py
│   │   ├── server.py                    # MCP Server 入口 (Stdio Transport)
│   │   ├── protocol_handler.py          # JSON-RPC 协议处理
│   │   └── tools/                       # MCP Tools 定义，五类共七个工具
│   │       ├── __init__.py
│   │       ├── lookup_canonical_steps.py       # 查询：候选标准步骤列表
│   │       ├── search_questions.py             # 检索：核心检索入口
│   │       ├── ingest_question_from_image.py   # 图片感知转换：触发 Ingestion Pipeline
│   │       ├── propose_canonical_step.py       # 提议写入：新标准步骤
│   │       ├── propose_misconception_tag.py    # 提议写入：新错因标签
│   │       ├── propose_question_mapping.py     # 提议写入：题目-步骤映射（单步骤/整条路径）
│   │       └── submit_recommendation_feedback.py  # 反馈回流：推荐结果反馈
│   │
│   ├── core/                            # Core 层 (核心业务逻辑)
│   │   ├── __init__.py
│   │   ├── settings.py                   # 配置加载与校验 (Settings：load_settings/validate_settings)
│   │   ├── types.py                      # 核心数据类型/契约（Question/QuestionRecord/ProcessedQuery/RetrievalResult，见 §4.1/§4.2），供 ingestion/retrieval/mcp 复用
│   │   │
│   │   ├── retrieval_engine/            # 检索引擎模块（对应通用文档 RAG 的 query_engine，见 §4.2）
│   │   │   ├── __init__.py
│   │   │   ├── pre_filter.py            # 结构化前置过滤（review_status/chapter_code/canonical_step_id，Hybrid Search 之前执行，硬边界排除）
│   │   │   ├── hybrid_search.py         # 混合检索引擎 (Dense + Sparse + RRF，reference_stem 有/无双模式)
│   │   │   ├── dense_retriever.py       # 稠密向量检索（仅 reference_stem 存在时执行）
│   │   │   ├── sparse_retriever.py      # 稀疏检索 (BM25，reference_stem 缺失时退化为纯结构化过滤)
│   │   │   ├── fusion.py                # 结果融合 (RRF 算法，含单路降级逻辑)
│   │   │   ├── post_filter.py           # 结构化后置过滤（misconception_tag_id，Hybrid Search 之后、Rerank 之前执行，safety net，missing→宽松放行）
│   │   │   └── reranker.py              # 重排序模块 (None/CrossEncoder/LLM，含多路径命中加分与 Fallback 显式标记)
│   │   │
│   │   ├── response/                    # 响应构建模块
│   │   │   ├── __init__.py
│   │   │   ├── response_builder.py      # 响应构建器（score_breakdown/why_recommended 组装，见 §4.3）
│   │   │   └── multimodal_assembler.py  # 多模态内容组装 (Text + Image)
│   │   │
│   │   └── trace/                       # 追踪模块
│   │       ├── __init__.py
│   │       ├── trace_context.py         # 追踪上下文 (trace_id/stages，创建→record_stage()→finish()，见 §4.5)
│   │       └── trace_collector.py       # 追踪收集器
│   │
│   ├── ingestion/                       # Ingestion Pipeline (离线批量导入 / 实时拍照录题共用，见 §4.1)
│   │   ├── __init__.py
│   │   ├── pipeline.py                  # Pipeline 主流程编排 (Loader→Transform→Embedding→Upsert，无 Splitter，支持 on_progress 回调)
│   │   ├── question_manager.py          # 题目生命周期管理 (list/delete/stats，对应通用文档 RAG 的 document_manager)
│   │   │
│   │   ├── transform/                   # Transform 模块 (结构清洗 + 图片感知转换)
│   │   │   ├── __init__.py
│   │   │   ├── base_transform.py        # Transform 抽象基类
│   │   │   ├── stem_cleaner.py          # 题面规则去噪（仅 PdfLoader/ImageLoader 场景，ManualEntryLoader 跳过；对应通用文档 RAG 的 chunk_refiner，不含跨块合并）
│   │   │   ├── topic_tagger.py          # 语义元数据注入（topic_tags 知识点标签，仅 PdfLoader 批量场景触发，见 §4.1）
│   │   │   └── image_recognizer.py      # 图片题目识别 (Vision LLM，分类型 Prompt，见 §4.6)
│   │   │
│   │   ├── embedding/                   # Embedding 模块 (双路向量化，见 §4.1)
│   │   │   ├── __init__.py
│   │   │   ├── dense_encoder.py         # 稠密向量编码
│   │   │   ├── sparse_encoder.py        # 稀疏向量编码 (BM25)
│   │   │   └── batch_processor.py       # 批处理优化
│   │   │
│   │   └── storage/                     # Storage 模块 (Upsert All-in-One，见 §4.1)
│   │       ├── __init__.py
│   │       ├── vector_upserter.py       # 向量库 Upsert（幂等，原子性保证）
│   │       ├── bm25_indexer.py          # BM25 索引构建
│   │       └── image_storage.py         # 图片文件存储
│   │
│   ├── libs/                            # Libs 层 (可插拔抽象层，见 §4.4)
│   │   ├── __init__.py
│   │   │
│   │   ├── loader/                      # Loader 抽象 (三种来源，见 §4.1)
│   │   │   ├── __init__.py
│   │   │   ├── base_loader.py           # Loader 抽象基类
│   │   │   ├── pdf_loader.py            # PdfLoader（题库文档多题目边界识别）
│   │   │   ├── image_loader.py          # ImageLoader（学生拍照/老师批量扫描共用）
│   │   │   ├── manual_entry_loader.py   # ManualEntryLoader（Dashboard 手动录入）
│   │   │   └── file_integrity.py        # 文件完整性检查 (SHA256 哈希)
│   │   │
│   │   ├── llm/                         # LLM 抽象
│   │   │   ├── __init__.py
│   │   │   ├── base_llm.py              # LLM 抽象基类
│   │   │   ├── llm_factory.py           # LLM 工厂
│   │   │   ├── azure_llm.py             # Azure OpenAI 实现
│   │   │   ├── openai_llm.py            # OpenAI 实现
│   │   │   ├── ollama_llm.py            # Ollama 本地模型实现
│   │   │   └── deepseek_llm.py          # DeepSeek 实现
│   │   │
│   │   ├── vision_llm/                  # Vision LLM 抽象（独立接口，非 LLM Factory 子选项，见 §4.4/§4.6）
│   │   │   ├── __init__.py
│   │   │   ├── base_vision_llm.py       # BaseVisionLLM 抽象基类
│   │   │   └── vision_llm_factory.py    # Vision LLM 工厂（按任务类型路由，具体选型为待解锁任务，见 §8）
│   │   │
│   │   ├── embedding/                   # Embedding 抽象
│   │   │   ├── __init__.py
│   │   │   ├── base_embedding.py        # Embedding 抽象基类
│   │   │   ├── embedding_factory.py     # Embedding 工厂
│   │   │   ├── openai_embedding.py      # OpenAI Embedding 实现
│   │   │   ├── azure_embedding.py       # Azure Embedding 实现
│   │   │   └── ollama_embedding.py      # Ollama 本地模型实现
│   │   │
│   │   ├── vector_store/                # VectorStore 抽象
│   │   │   ├── __init__.py
│   │   │   ├── base_vector_store.py     # VectorStore 抽象基类
│   │   │   ├── vector_store_factory.py  # VectorStore 工厂
│   │   │   └── chroma_store.py          # Chroma 实现
│   │   │
│   │   ├── reranker/                    # Reranker 抽象
│   │   │   ├── __init__.py
│   │   │   ├── base_reranker.py         # Reranker 抽象基类
│   │   │   ├── reranker_factory.py      # Reranker 工厂
│   │   │   ├── cross_encoder_reranker.py# CrossEncoder 实现
│   │   │   └── llm_reranker.py          # LLM Rerank 实现
│   │   │
│   │   └── evaluator/                   # Evaluator 抽象（不含 Ragas，见 §4.4）
│   │       ├── __init__.py
│   │       ├── base_evaluator.py        # Evaluator 抽象基类
│   │       ├── evaluator_factory.py     # Evaluator 工厂
│   │       └── custom_evaluator.py      # 自定义检索指标实现 (Hit Rate/MRR)
│   │
│   └── observability/                   # Observability 层 (可观测性，见 §4.5)
│       ├── __init__.py
│       ├── logger.py                    # 结构化日志 (JSON Formatter)
│       ├── review_service.py            # review_queue 读写服务（本项目新增，通用文档 RAG 无对应）
│       ├── dashboard/                   # Web Dashboard (可视化管理平台)
│       │   ├── __init__.py
│       │   ├── app.py                   # Streamlit 入口 (页面导航注册)
│       │   ├── pages/                   # 七大功能页面
│       │   │   ├── overview.py           # 系统总览
│       │   │   ├── question_browser.py   # 题库浏览器（对应通用文档 RAG 的 data_browser）
│       │   │   ├── ingestion_manager.py  # 数据摄取管理
│       │   │   ├── review_queue.py       # 待审核队列（本项目新增，Tab A 待审提议/Tab B taxonomy 浏览）
│       │   │   ├── evaluation_panel.py   # 评估面板（Tab A 反馈汇总/Tab B 指标运行）
│       │   │   ├── ingestion_traces.py   # Ingestion 追踪
│       │   │   └── query_traces.py       # Query 追踪
│       │   └── services/                # Dashboard 数据服务层
│       │       ├── trace_service.py     # Trace 读取服务 (解析 traces.jsonl)
│       │       ├── data_service.py      # 数据浏览服务 (VectorStore 读取)
│       │       └── config_service.py    # 配置读取服务 (Settings 展示)
│       └── evaluation/                  # 评估模块
│           ├── __init__.py
│           ├── eval_runner.py           # 评估执行器
│           └── composite_evaluator.py   # 组合评估器（挂载自定义指标，预留未来扩展）
│
├── data/                                # 数据目录
│   ├── documents/                       # 原始题库文档存放（PDF 来源）
│   │   └── {chapter_code}/              # 按章节分类
│   ├── images/                          # 题目关联图形文件存放
│   │   └── {chapter_code}/              # 按章节分类（实际存储在 {question_id}/ 子目录下）
│   └── db/                              # 数据库与索引文件目录（SQLite 持久化存储架构统一说明，见 §4.1）
│       ├── ingestion_history.db         # 文件完整性历史记录 (SQLite)
│       │                                # 表结构：input_hash, source_type, status, processed_at, error_msg, question_count
│       │                                # 用途：增量摄取，避免重复处理未变更输入
│       ├── image_index.db               # 图片索引映射 (SQLite)
│       │                                # 表结构：image_id, file_path, question_id
│       │                                # 用途：快速查询 image_id → 本地文件路径，支持检索命中后返回图形
│       ├── chroma/                      # Chroma 向量库目录
│       │                                # 存储 Dense Vector、Question 原文与 Metadata（All-in-One 存储策略，见 §4.1）
│       └── bm25/                        # BM25 索引目录
│                                        # 存储倒排索引与 IDF 统计信息（当前使用 pickle）
│
├── cache/                               # 缓存目录
│   ├── embeddings/                      # Embedding 缓存 (按内容哈希，差量计算)
│   └── recognition/                     # 图片识别结果缓存 (按图片内容哈希，见 §4.6 幂等与增量处理)
│
├── logs/                                # 日志目录
│   ├── traces.jsonl                     # 追踪日志 (JSON Lines，Query Trace + Ingestion Trace 双链路)
│   └── app.log                          # 应用日志
│
├── tests/                               # 测试目录（覆盖范围见 §5）
│   ├── unit/                            # 单元测试
│   │   ├── test_loaders.py              # Loader 多题目边界识别/置信度标记测试
│   │   ├── test_transform.py            # Transform 去噪/语义元数据/Vision LLM降级测试
│   │   ├── test_embedding.py            # Embedding 差量计算/批处理测试
│   │   ├── test_hybrid_search.py        # Retrieval reference_stem两态/多路径加分测试
│   │   ├── test_reranker_fallback.py    # Reranker Fallback回退与显式标记测试
│   │   └── test_review_gate.py          # Review-Gate 待审内容不可见性测试
│   ├── integration/                     # 集成测试
│   │   ├── test_ingestion_pipeline.py   # Loader→Transform→Embedding→Upsert 完整流程
│   │   ├── test_hybrid_search.py        # Dense+Sparse 融合集成测试（含 reference_stem 有/无）
│   │   └── test_mcp_server.py           # MCP 七工具端到端测试
│   ├── e2e/                             # 端到端测试
│   │   ├── test_question_ingestion.py   # 三种录入来源场景
│   │   ├── test_recall.py               # 结构化召回回归测试
│   │   └── test_mcp_client.py           # 模拟辅导讲师 Agent 测试
│   └── fixtures/                        # 测试数据
│       ├── sample_questions/            # 测试题目样本（覆盖不同章节/难度）
│       └── golden_test_set.json         # 黄金测试集
│
├── scripts/                             # 脚本目录
│   ├── ingest.py                        # 数据摄取脚本（离线批量导入入口）
│   ├── query.py                         # 检索测试脚本（在线查询入口）
│   ├── evaluate.py                      # 评估运行脚本
│   └── start_dashboard.py               # Dashboard 启动脚本
│
├── main.py                              # MCP Server 启动入口
├── pyproject.toml                       # Python 项目配置
├── requirements.txt                     # 依赖列表
└── README.md                            # 项目说明
```

**与通用文档 RAG 目录结构的关键差异**：

- **不做 Chunking**：去掉了 `chunking/`、`splitter/` 整个模块目录及对应工厂（`libs/splitter/`），Ingestion Pipeline 只有 Loader→Transform→Embedding→Upsert 四步。
- **`query_engine/` → `retrieval_engine/`**：去掉 `query_processor.py`（结构化输入不需要关键词提取/查询扩展），Multi-stage Filtering 的前置/后置过滤拆成两个独立文件 `pre_filter.py`/`post_filter.py`（对应 §4.2 中两个物理上不相邻的过滤阶段，中间夹着整个 Hybrid Search，不合并成一个文件）。
- **Vision LLM 独立目录**：按 §4.4 决策单独建 `libs/vision_llm/`，不塞进 `libs/llm/`，因为 `BaseVisionLLM` 是与 `LLMClient` 并列的独立抽象接口，不是 LLM Factory 的子选项。
- **新增 `review_service.py`、`review_queue.py` 页面**：对应本项目独有的 Review-Gate 机制，通用文档 RAG 无对应设计。
- **`ragas_evaluator.py` 全部移除**：`libs/evaluator/` 和 `observability/evaluation/` 目录都只保留自定义指标实现，不引入 Ragas（理由见 `docs/DECISIONS.md`「评估框架选型」）。
- **Prompt 模板去掉 `chunk_refinement.txt`**（无 Chunk 概念），新增 `image_recognition.txt`（对应 §4.6 分类型识别 Prompt）。

---

### 6.3 模块说明

#### 6.3.1 MCP Server 层

| 模块 | 职责 | 关键技术点 |
|-----|-----|----------|
| `server.py` | MCP Server 主入口，处理 Stdio Transport 通信 | Python MCP SDK，JSON-RPC 2.0 |
| `protocol_handler.py` | 协议解析与能力协商 | `initialize`、`tools/list`、`tools/call` |
| `tools/*` | 对外暴露的七个工具函数实现（五类，见 §4.3） | 装饰器定义，参数校验，响应格式化 |

#### 6.3.2 Core 层

| 模块 | 职责 | 关键技术点 |
|-----|-----|----------|
| `settings.py` | 配置加载与校验 | 读取 `config/settings.yaml`，解析为 `Settings`，必填字段校验（fail-fast） |
| `types.py` | 核心数据类型/契约（全链路复用） | 定义 `Question/QuestionRecord/ProcessedQuery/RetrievalResult`；序列化稳定；作为 ingestion/retrieval/mcp 的数据契约中心 |
| `pre_filter.py` | 结构化前置过滤 | `review_status=approved`（Review-Gate 硬规则，不可配置/不可跳过）+ `chapter_code`/`canonical_step_id`（普通结构化过滤条件），Hybrid Search 之前执行，缩小候选集 |
| `hybrid_search.py` | 混合检索编排 | 并行调度 Dense/Sparse 检索，结果转交 Fusion 融合；reference_stem 有/无双模式 |
| `dense_retriever.py` | 语义向量检索 | 仅 `reference_stem` 存在时执行；Query Embedding + VectorStore 检索，Cosine Similarity |
| `sparse_retriever.py` | BM25 关键词检索 | `reference_stem` 缺失时退化为纯结构化过滤；倒排索引查询，TF-IDF 打分 |
| `fusion.py` | 结果融合 | RRF 算法，排名倒数加权；单路降级时直接采用 Sparse 排名 |
| `post_filter.py` | 结构化后置过滤 | `misconception_tag_id`，missing→宽松包含，Hybrid Search 之后、Rerank 之前执行 |
| `reranker.py` | 精排重排 | 多路径命中加分（受控幅度）+ `difficulty` 软偏好；CrossEncoder / LLM Rerank / Fallback 回退（必须显式标记 `used_fallback`） |
| `response_builder.py` | 响应构建 | MCP 响应格式化，`score_breakdown`/`why_recommended` 组装 |
| `multimodal_assembler.py` | 多模态组装 | Text + Image Base64 编码，MCP 多内容类型 |
| `trace_context.py` | 追踪上下文 | trace_id 生成，阶段记录，finish 汇总 |
| `trace_collector.py` | 追踪收集器 | 收集 trace 并触发持久化到 JSON Lines |

#### 6.3.3 Scripts 层（命令行入口）

| 脚本 | 职责 | 关键技术点 |
|-----|-----|----------|
| `ingest.py` | 离线批量导入入口 | CLI 参数解析，调用 Ingestion Pipeline，支持 `--chapter`/`--path`/`--force` |
| `query.py` | 检索测试入口 | CLI 参数解析，调用 HybridSearch + Reranker，支持 `--canonical-step-id`/`--reference-stem`（可选）/`--top-k`/`--verbose` |
| `evaluate.py` | 评估运行入口 | 加载 golden_test_set，运行评估，输出 metrics |
| `start_dashboard.py` | Dashboard 启动入口 | Streamlit 应用启动 |

#### 6.3.4 Ingestion Pipeline 层

| 模块 | 职责 | 关键技术点 |
|-----|-----|----------|
| `pipeline.py` | Pipeline 流程编排 | Loader→Transform→Embedding→Upsert 四步（无 Splitter）；异常处理，增量更新；支持 `on_progress` 回调；统一使用 `core/types.py` 的数据契约 |
| `question_manager.py` | 题目生命周期管理 | list/delete/stats 操作；跨 4 个存储（Chroma/BM25/ImageStorage/FileIntegrity）的协调删除；供 Dashboard 与 CLI 调用 |
| `transform/base_transform.py` | Transform 抽象 | 原子化、幂等；可独立重试；失败降级不阻塞 |
| `transform/stem_cleaner.py` | 题面规则去噪 | 仅 `PdfLoader`/`ImageLoader` 场景，`ManualEntryLoader` 跳过；不含跨块合并 |
| `transform/topic_tagger.py` | 语义元数据注入 | 仅 `PdfLoader` 批量场景触发；`topic_tags` 知识点标签规则/LLM 生成 |
| `transform/image_recognizer.py` | 图片题目识别 | Vision LLM；分类型 Prompt（文字OCR/几何图形结构）；识别失败/低置信度降级 |
| `embedding/dense_encoder.py` | 稠密向量编码 | 通过 `libs.embedding` 调用具体 provider；批处理 |
| `embedding/sparse_encoder.py` | 稀疏向量编码 | BM25 编码/统计；批处理 |
| `embedding/batch_processor.py` | 批处理优化 | `batch_size` 驱动的批量调用，减少网络 RTT |
| `storage/vector_upserter.py` | 向量存储写入 | 通过 `libs.vector_store` Upsert；幂等；All-in-One 存储，metadata 完整 |
| `storage/bm25_indexer.py` | BM25 索引构建 | 倒排索引写入，驱动 `sparse_retriever.py` 检索计算；提供 `remove_document()` 供 `question_manager.py` 删除时调用 |
| `storage/image_storage.py` | 图片文件存储 | 本地文件系统写入，`image_index.db` 索引映射维护 |

#### 6.3.5 Libs 层（可插拔抽象）

| 抽象接口 | 当前默认实现 | 可替换选项 |
|---------|------------|----------|
| `LLMClient` | 占位跑通链路用 Ollama/OpenAI 任一均可，正式选型为待解锁任务，见 §8 | Azure OpenAI / OpenAI / Ollama / DeepSeek |
| `VisionLLMClient`（独立接口，非 `LLMClient` 子选项） | 占位跑通链路用任一可用 Vision LLM，正式选型（按任务类型路由）为待解锁任务，见 §8 | 具体模型对比见 `docs/DECISIONS.md`「Vision LLM 选型策略」 |
| `EmbeddingClient` | 占位跑通链路用 OpenAI text-embedding-3 或 Ollama 本地模型，正式选型为待解锁任务，见 §8 | OpenAI / BGE / Ollama 本地模型 |
| `Loader` | PdfLoader / ImageLoader / ManualEntryLoader（三种来源并存，非互相替代） | 未来可扩展新的录入来源 |
| `FileIntegrity` | SQLite (`data/db/ingestion_history.db`) | Redis（分布式）/ PostgreSQL（企业级） |
| `VectorStore` | Chroma | Qdrant / Pinecone |
| `Reranker` | None（Top-K 默认） | Cross-Encoder / LLM Rerank |
| `Evaluator` | 自定义检索指标（Hit Rate/MRR） | 不含 Ragas，见 `docs/DECISIONS.md`「评估框架选型」 |

#### 6.3.6 Observability 层

| 模块 | 职责 | 关键技术点 |
|-----|-----|----------|
| `logger.py` | 结构化日志 | JSON Formatter，JSON Lines 输出 |
| `review_service.py` | `review_queue` 读写服务 | 本项目新增，通用文档 RAG 无对应；供待审核队列页读写 |
| （`trace_context.py`/`trace_collector.py`，见 §6.3.2） | 请求级追踪机制 | 物理文件仅存在于 `src/core/trace/`，此处为可观测性层对同一机制的读取/复用视角，非独立文件 |
| `dashboard/app.py` | Dashboard 入口 | Streamlit 多页面应用，`st.navigation` 页面注册 |
| `dashboard/pages/overview.py` | 系统总览 | 组件配置卡片，数据资产统计 |
| `dashboard/pages/question_browser.py` | 题库浏览器 | 题目列表，关联标准步骤/难度，按章节筛选 |
| `dashboard/pages/ingestion_manager.py` | 数据摄取管理 | 三种来源触发摄取（进度条），题目删除 |
| `dashboard/pages/review_queue.py` | 待审核队列 | 本项目新增；Tab A 待审提议 approve/reject/edit，Tab B taxonomy 只读浏览 |
| `dashboard/pages/evaluation_panel.py` | 评估面板 | Tab A 反馈汇总，Tab B 自定义指标运行、历史趋势 |
| `dashboard/pages/ingestion_traces.py` | Ingestion 追踪 | 摄取历史，阶段耗时瀑布图（无 split 阶段） |
| `dashboard/pages/query_traces.py` | Query 追踪 | 查询历史，Dense/Sparse 对比，Rerank 变化，Fallback 触发记录 |
| `dashboard/services/trace_service.py` | Trace 数据服务 | 解析 traces.jsonl，按 trace_type 分类 |
| `dashboard/services/data_service.py` | 数据浏览服务 | 封装 VectorStore 读取 |
| `dashboard/services/config_service.py` | 配置读取服务 | 封装 Settings 展示 |
| `evaluation/eval_runner.py` | 评估执行 | 黄金测试集，指标计算，报告生成 |
| `evaluation/composite_evaluator.py` | 组合评估器 | 挂载自定义指标，预留未来追加其他 Evaluator 的组合能力 |

---

### 6.4 数据流说明

#### 6.4.1 离线/实时数据摄取流（Ingestion Flow）

```
原始输入（PDF 题库 / 图片拍照或扫描 / 手动表单）
      │
      ▼
┌─────────────────┐     未变更则跳过
│ File Integrity  │───────────────────────────► 结束
│   (SHA256)      │
└────────┬────────┘
         │ 新输入/已变更
         ▼
┌─────────────────┐
│     Loader      │  PdfLoader/ImageLoader/ManualEntryLoader
│  (三种来源)      │  多题目边界识别 + source_position 定位
└────────┬────────┘
         │ List[Question]（含 metadata.recognition_confidence）
         ▼
┌─────────────────┐
│   Transform     │  stem_cleaner（题面去噪）+ topic_tagger（语义标签，
│                 │  仅 PdfLoader 批量场景）+ image_recognizer（Vision LLM，
│                 │  仅图片来源；不可用时降级 transform_failed=true）
└────────┬────────┘
         │ 清洗后的 Question[]
         ▼
┌─────────────────┐
│   Embedding     │  Dense + Sparse 双路编码（差量计算，batch_processor 批处理）
│  (Dual Path)    │
└────────┬────────┘
         │ QuestionRecord[]（Dense Vector + Sparse Vector + Metadata）
         ▼
┌─────────────────┐
│    Upsert       │  Chroma All-in-One Upsert (幂等) + BM25 Index + 图片存储
│   (Storage)     │  Batch 事务性写入（原子性保证）
└─────────────────┘
```

（无 Splitter 阶段——本项目不做 Chunking，见 §4.1）

#### 6.4.2 在线检索流（Retrieval Flow）

```
辅导讲师 Agent 调用（via MCP Client）
      │
      ▼
┌─────────────────┐
│  MCP Server     │  JSON-RPC 解析，工具路由（search_questions）
│ (Stdio Transport)│
└────────┬────────┘
         │ ProcessedQuery（canonical_step_id 必填 + reference_stem 可选）
         ▼
┌─────────────────┐
│   Pre-Filter    │  review_status=approved（硬规则）+ chapter_code/canonical_step_id
│  (结构化前置)    │
└────────┬────────┘
         │ 已过滤候选集
         ▼
┌─────────────────────────────────────────────┐
│              Hybrid Search                  │
│  ┌─────────────┐          ┌─────────────┐   │
│  │Dense Retrieval│  并行   │Sparse Retrieval│   │
│  │仅 reference_ │◄───────►│reference_stem│   │
│  │stem存在时执行 │          │有:BM25/无:结构化│   │
│  └──────┬──────┘          └──────┬──────┘   │
│         │                        │          │
│         └────────┬───────────────┘          │
│                  ▼                          │
│         ┌─────────────┐                     │
│         │   Fusion    │  RRF 融合；单路降级时  │
│         │   (RRF)     │  直接用 Sparse 排名   │
│         └──────┬──────┘                     │
└────────────────┼────────────────────────────┘
                 │ Top-N 候选
                 ▼
┌─────────────────┐
│  Post-Filter    │  misconception_tag_id（missing→宽松包含）
│  (结构化后置)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Reranker     │  多路径命中加分 + difficulty 软偏好；CrossEncoder / LLM / None；
│   (Optional)    │  Fallback `used_fallback`/`fallback_reason` 显式标记
└────────┬────────┘
         │ Top-K 精排结果（score_breakdown）
         ▼
┌─────────────────┐
│ Response Builder│  why_recommended 组装 + 图片 Base64 编码 + MCP 格式化
│                 │
└────────┬────────┘
         │ MCP Response (TextContent + ImageContent)
         ▼
返回给辅导讲师 Agent
```

（无 Query Processor 阶段——本项目输入已结构化，见 §4.2；每个阶段各自调用 `TraceContext.record_stage()`，见 §4.5）

#### 6.4.3 管理操作流（Management Flow）

```
Dashboard (Streamlit UI)
      │
      ├─── 题库浏览 ──────────────────────────────────────────┐
      │                                                       │
      │    data_service.py（封装 VectorStore 抽象读取）         │
      │    └── 按 chapter_code/question_id 返回题目列表 /      │
      │        详情 / 图片预览                                 │
      │                                                       │
      ├─── 数据摄取管理 ──────────────────────────────────────┤
      │                                                       │
      │    触发摄取：                                          │
      │    ├── IngestionPipeline.run(source, source_type,     │
      │    │                         on_progress=callback)    │
      │    └── st.progress() 实时更新进度                      │
      │                                                       │
      │    删除题目：                                          │
      │    ├── QuestionManager.delete_question(question_id)   │
      │    │   ├── VectorStore.delete_by_id(question_id)      │
      │    │   │   （sparse 侧经 BM25Indexer.remove_document） │
      │    │   ├── ingestion_records 删除对应摄取记录           │
      │    │   ├── ingestion_history（若为唯一产出）→          │
      │    │   │   FileIntegrity.remove_record(input_hash)    │
      │    │   └── ImageStorage 删除关联图片文件（若有）        │
      │    └── 刷新题目列表                                    │
      │                                                       │
      ├─── 待审核队列 ────────────────────────────────────────┤
      │                                                       │
      │    ReviewService                                       │
      │    ├── Tab A：读取 review_queue，approve/reject/edit   │
      │    │   → approve 后 review_status 更新，检索立即可见    │
      │    └── Tab B：只读浏览已批准 taxonomy（标准步骤/错因库） │
      │                                                       │
      └─── Trace 查看（复用 trace_service.py，非新设计）───────┘
           │
           TraceService
           ├── 读取 logs/traces.jsonl
           ├── 按 trace_type 分类 (query / ingestion)
           └── 返回 Trace 列表与详情
```

**与通用文档 RAG 数据流的关键差异**：

- **6.4.1**：Loader 从单一 PDF 换成三种来源并列，去掉 Splitter 阶段，Transform 阶段的三个动作对应 §4.1/§4.6 的实际触发条件。
- **6.4.2**：去掉 Query Processor（输入已结构化），新增 Pre-Filter/Post-Filter 两个显式阶段（对应 §4.2 Multi-stage Filtering），Dense/Sparse 双路标注了 `reference_stem` 条件分支，Fusion 标注单路降级逻辑，Rerank 标注多路径加分与 Fallback 强制显式标记（`used_fallback`/`fallback_reason`）；每个阶段各自调用 `TraceContext.record_stage()`，非流程末尾统一记录。
- **6.4.3**：题库浏览管道改经 `data_service.py` 封装的 VectorStore 抽象读取，不直接点名具体实现类；`delete_question()` 按 §4.1 原文改为四步（VectorStore 删除含 dense/sparse、`ingestion_records` 删除、`ingestion_history` 处理记录移除、图片文件删除），`BM25Indexer.remove_document()` 是 VectorStore 删除步骤里 sparse 一侧的具体接口，不是独立并列步骤；新增"待审核队列"管道（本项目独有，对应 Review-Gate 机制）；"Trace 查看"管道复用已有 `trace_service.py` 机制，非新设计。

---
