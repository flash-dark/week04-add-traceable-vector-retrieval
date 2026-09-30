## Purpose

让已登录用户在本班材料中检索相关切片，并让每条命中回溯到材料与字符偏移。没有依据时返回未找到，不编造出处。

## ADDED Requirements

### Requirement: Retrieval is limited to the session class

The system MUST search only chunks whose class is the class stored in the current login session. A class identifier supplied by the client MUST NOT change the search scope.

#### Scenario: Search returns only the current class

- **WHEN** 已登录用户发起检索，且本班与其他班都存在可匹配的切片
- **THEN** 返回的每条切片 MUST 属于会话中的班级
- **AND** 其他班的切片 MUST NOT 出现在结果中

#### Scenario: Client-supplied class is ignored

- **WHEN** 已登录用户在请求参数或 JSON 中提交另一个 `class_id`
- **THEN** 检索范围 MUST 仍是会话中的班级
- **AND** 结果 MUST 与不提交该字段时一致

#### Scenario: Unauthenticated search is rejected

- **WHEN** 请求没有有效登录会话
- **THEN** 系统 MUST 拒绝检索
- **AND** MUST NOT 返回任何切片正文

### Requirement: Keyword vector and hybrid modes

The system MUST support keyword search, vector search, and hybrid search over the current class. Hybrid search MUST be the default. Vector search MUST discard candidates whose cosine similarity is below 0.35.

#### Scenario: Keyword search does not embed the query

- **WHEN** 用户以关键字模式检索一个在原文中出现过的词
- **THEN** 系统 MUST 只在本班切片的正文索引中查找
- **AND** MUST NOT 调用嵌入服务

#### Scenario: Vector search keeps results at or above the threshold

- **WHEN** 用户以向量模式检索
- **THEN** 问句 MUST 先被嵌入并与本班切片向量计算余弦相似度
- **AND** 余弦相似度低于 0.35 的切片 MUST NOT 出现在结果中

#### Scenario: Hybrid search fuses ranks

- **WHEN** 用户以混合模式检索，且关键字路径与向量路径各自产生了排序
- **THEN** 系统 MUST 以 `1 / (60 + 关键字名次) + 1 / (60 + 向量名次)` 融合名次
- **AND** 两条路径都命中的切片 MUST 排在只命中一条路径的切片之前

#### Scenario: No surviving candidate

- **WHEN** 所选模式过滤之后没有任何候选切片
- **THEN** 系统 MUST 返回未找到
- **AND** MUST NOT 用低于阈值或属于其他班的切片凑一条结果

### Requirement: Hits are traceable to source text

Each retrieval hit MUST identify the source material, the chunk index, and the character offsets of that chunk in the material text. The displayed snippet MUST be the chunk text stored for that same chunk identifier.

#### Scenario: Hit carries source coordinates

- **WHEN** 检索返回一条切片
- **THEN** 该条结果 MUST 包含材料标识、材料标题、切片序号、起始偏移、结束偏移和切片正文
- **AND** 用该材料标识与偏移 MUST 能定位到入库正文中的同一段文字

#### Scenario: Vector store is not the text source

- **WHEN** 向量检索命中一条切片
- **THEN** 系统 MUST 用向量主键回到 `knowledge_chunks` 读取正文
- **AND** 返回给调用方的正文 MUST NOT 取自向量库中的浮点向量

### Requirement: Chunk text and vectors share one identifier

The system MUST store chunk text in MySQL and the corresponding vector in a vector database. The vector point identifier MUST equal `knowledge_chunks.id`.

#### Scenario: Identifiers match after indexing

- **WHEN** 一份材料完成切分并写入向量
- **THEN** 每个切片在 MySQL 中的主键 MUST 等于向量库中的主键
- **AND** 该主键 MUST 等于向量 payload 中的 `chunk_id`

### Requirement: Answers cite retrieved chunks only

Question answering MUST run hybrid retrieval for the latest user sentence, limited to the session class, and MUST keep at most the top 4 chunks. The system MUST call the generation model only when at least one chunk was retrieved.

#### Scenario: Answer uses bracket citations

- **WHEN** 混合检索至少命中一条本班切片
- **THEN** 系统 MUST 把最多前 4 条切片交给生成过程
- **AND** 回答中的 `[1]`、`[2]` MUST 按返回的出处列表顺序指回这些切片

#### Scenario: No hit skips the model

- **WHEN** 本班混合检索没有候选切片
- **THEN** 系统 MUST 返回「资料中未找到相关内容」
- **AND** `citations` MUST 为空
- **AND** 系统 MUST NOT 调用生成模型
