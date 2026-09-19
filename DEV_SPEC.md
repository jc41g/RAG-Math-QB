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

**目标：** 给定辅导讲师 Agent 传入的结构化错因信息，检索并排序出对症的练习题。与通用文档 RAG 不同，本模块的输入不是"待消歧的自由文本查询"，而是**已经结构化的错因描述**——标准步骤 ID、可选的错因标签 ID、章节、难度——上游的语义理解、路径判断、自由文本归一化工作已由辅导讲师 Agent 完成，本模块只负责"拿结构化条件找题、排序"。

设计要点：

- **Query 输入契约**：`ProcessedQuery` 不做关键词提取/同义词扩展（那是通用文档 RAG 应对自由文本查询的手段，本场景输入已结构化，不适用）。完整字段：
  - `canonical_step_id: str`（必填）
  - `misconception_tag_id: Optional[str]`
  - `chapter_code: Optional[str]`
  - `difficulty: Optional[int]`
  - `top_k: int`（默认值为待解锁任务，当前占位 10）
  - `exclude_question_ids: List[str]`（可选，用于排除学生已经做过的题，默认空列表）
  - 无 `keywords`/`expanded_terms` 字段——这两个字段服务于自由文本查询的关键词提取/同义词扩展，本场景输入本身已结构化，不需要这一步预处理。
- **结构化过滤（硬过滤，前置）**：`review_status != approved` 的题目、以及不满足 `chapter_code`/`canonical_step_id` 硬约束的候选，在检索前直接排除，不进入候选集。对应 `docs/DECISIONS.md`「技术选型验证」中确立的"结构化过滤 + 语义排序"分工原则——能用结构化字段精确排除的，不留给语义层判断。
- **Hybrid Search（粗排召回）**：与通用文档 RAG 设计一致，不做改动——Dense Route（题面 Embedding 相似度，捕捉"标签相同、具体条件不同"的语义差异，几何题的图形构造差异即典型场景）+ Sparse Route（BM25 关键词检索）+ RRF 融合（`Score = 1 / (k + Rank_Dense) + 1 / (k + Rank_Sparse)`，`k` 可配置）。
- **多路径匹配加分（本项目特有设计）**：候选题目可能关联多条标准步骤路径（`step_sequence`，见 §7 数据模型的多对多关系）。若同一题目的多条路径都命中传入的 `canonical_step_id`，视为该题对这个错误步骤更有代表性，给予小幅加分；命中单条路径的题目不因"存在其他不相关路径"而受影响。**加分幅度必须受控（远小于标准步骤匹配/错因匹配等主信号权重）**，避免"路径数量多"本身压过"是否真正对症"这个核心排序目标——具体加分系数为待解锁任务（见 §8 末尾），当前用小值占位。**路径本身是审核录入阶段的静态数据，本模块只做路径命中判断与加分，不做路径推理**（硬边界，见 `docs/DECISIONS.md`）。
- **Rerank（精排）**：
  - 候选集按标准步骤匹配度、错因标签匹配度、题面语义相似度、章节匹配、难度匹配、多路径命中加分共同加权排序，具体权重公式为待解锁任务（见 §8 末尾），当前用等权重占位跑通链路。
  - 可插拔后端：None（直接用 Fusion 排名）/ Cross-Encoder / LLM Rerank，与通用可插拔架构一致（见 §4.4）。
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
  - 排序理由透明：对应通用文档 RAG 的"引用透明"理念，本项目的等价物是 `score_breakdown` + `why_recommended`（见 §4.2），而非文档的 `source_file`/`page`/`chunk_id`。
  - 单一确定 Client：不需要像通用文档 RAG 那样为不同 Client（Copilot vs Claude Desktop）设计差异化的降级/适配策略，因为当前唯一的 Client 就是辅导讲师 Agent，其能力边界由本项目团队自行定义。
- **传输协议**：Stdio（本地子进程通信），与通用文档 RAG 设计一致——无需网络端口/鉴权，数据不经网络，`stdout` 仅输出合法 MCP 消息，日志统一走 `stderr`。
- **SDK 选型**：优先采用 Python 官方 MCP SDK（`mcp`），复用其 `@server.tool()` 声明式定义方式，不自行实现协议底层细节。
- **协议版本协商**：跟踪 MCP 最新稳定版本，在 `initialize` 阶段完成 Client/Server 能力协商，确保兼容性，与通用文档 RAG 一致，不做改动。
- **Tools 设计**：按职责分五类，共八个工具。

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

设计原则（与通用文档 RAG 相同）：

- **接口隔离**：为每类组件定义最小化抽象接口，上层业务逻辑仅依赖接口，不依赖具体实现。
- **配置驱动**：通过 `config/settings.yaml` 指定各组件的具体后端，代码无需修改即可切换实现。
- **工厂模式**：工厂函数根据配置动态实例化对应实现类，一处配置、处处生效。
- **优雅降级**：首选后端不可用时，自动回退到备选方案或安全默认值。

各组件抽象（沿用原版设计，具体默认 Provider 为待解锁任务，见 §8 末尾）：

- **LLM / Embedding 提供者**：`BaseLLM`（`chat(messages) -> response`）、`BaseEmbedding`（`embed(texts) -> vectors`），统一屏蔽 Azure OpenAI / OpenAI / DeepSeek / Ollama 等不同 Provider 的认证与请求格式差异。
- **Vision LLM 提供者**：`BaseVisionLLM`，支持文本+图片多模态输入，供 §4.1 `ImageLoader` 调用。这是本项目相对通用文档 RAG **优先级更高**的一环——通用文档 RAG 里 Vision LLM 只用于可选的图片描述增强，本项目里它是图片题目录入这一核心摄取路径的必需依赖。
- **向量数据库**：`BaseVectorStore`（`.add()`/`.query()`/`.delete()`），默认 Chroma（嵌入式、零部署成本，适合本地开发与快速原型验证）。
- **精排后端（Reranker）**：见 §4.2，None / Cross-Encoder / LLM Rerank 三种，工厂路由 + Fallback 语义。
- **不需要 Splitter 抽象**：与通用文档 RAG 不同——本项目不做 Chunking（见 §4.1），因此不需要 `BaseSplitter`/`SplitterFactory` 这一层。

**评估框架**：与通用文档 RAG 不同，本项目**主用自定义检索指标**（Hit Rate、MRR 等，衡量排序质量），不引入 Ragas——Ragas 的核心指标（Faithfulness、Answer Relevancy）面向"生成式回答质量"评测，本项目不做生成式回答，没有可用的评测对象。完整分析见 `docs/DECISIONS.md`「评估框架选型」。`BaseEvaluator` 抽象接口本身仍保留（`evaluate(...) -> metrics`），为未来接入其他检索类评估指标留出扩展空间，但默认实现只需覆盖自定义检索指标。

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
  fusion_algorithm: rrf
  rerank_backend: none  # none | cross_encoder | llm
evaluation:
  backends: [custom_metrics]  # 不含 ragas，理由见 docs/DECISIONS.md
```

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

**`TraceContext` 机制**（与通用文档 RAG 完全一致，直接复用，因为这是纯工程范式，不涉及业务差异）：

1. **创建**：Pipeline/检索入口处创建 `TraceContext` 实例，生成唯一 `trace_id`，记录基础信息。
2. **阶段记录**：`TraceContext.record_stage(stage_name: str, method: str, details: dict, latency_ms: float)`——各阶段执行完毕后调用，`stage_name` 是固定的通用大类（如 `retrieval`/`rerank`），`method` 记录具体实现（如 `bm25`/`cross_encoder`），`details` 记录该方法相关的细节数据。这样无论底层可插拔组件怎么替换，`stage_name` 结构保持稳定，Dashboard 展示逻辑无需调整。
3. **结束**：调用 `TraceContext.finish()`，序列化为 JSON，追加写入 `traces.jsonl`。
4. **调用约定**：显式调用模式——不强制、不会因未调用而报错，但依赖各可插拔组件的实现者在核心逻辑执行后主动调用 `record_stage()`。好处是代码透明，代价是需要开发者自觉遵守约定（与通用文档 RAG 的取舍一致）。

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
