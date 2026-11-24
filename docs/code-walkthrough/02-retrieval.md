# 02 · 从一篇政策到可引用证据：检索代码逐段讲解

这篇按“文档怎样入库 → 用户问题怎样改写 → 怎样找资料 → 怎样排序 → 资料不足怎样再找”的顺序读代码。默认读者懂基本编程概念，但不熟悉 Python、向量检索和 RAG。

本文代码块标为“真实源码”的部分来自当前仓库。类方法保留类内缩进，省略了所属类和 import，因此单独复制一个代码块不一定能运行。“教学数据”是便于手算的假设输入、分数或模型回复，**不是本项目实测输出**。本文没有运行或下载模型。

## 1. 先认清传递的数据

检索返回的不是一整篇文件，而是 `RetrievedChunk` 对象。可以把一个对象理解为“一段正文，加上它来自哪里、当前排第几等信息”。定义在 [app/schemas/retrieval.py](../../app/schemas/retrieval.py)。

| 字段 | 示例（教学数据） | 用途 |
| --- | --- | --- |
| `chunk_id` | `return_policy_ab12cd34_00` | 片段的标识；去重、引用都依赖它 |
| `document_id` | `return_policy_ab12cd34` | 片段所属文档 |
| `content` | `质量问题退货的运费由商家承担。` | 真正用于检索和生成的正文 |
| `title` / `section` | `退货政策` / `运费承担` | 标题、章节等来源信息 |
| `score` | `0.0325` | 当前阶段的分数；不同阶段含义不同 |
| `rank` | `1` | 当前阶段的排名，从 1 开始 |
| `source` | `bm25`、`vector`、`rrf`、`rerank` | 当前结果由哪个阶段产出 |

`score` 并不是始终不变的“可信度”：BM25 写入关键词相关性分数，向量库写入向量相似度，RRF 写入融合分数，精排又会覆盖成精排模型分数。不能把这些数直接当成同一量纲比较。

默认配置来自 [app/config/settings.py](../../app/config/settings.py)：

| 配置 | 默认值 | 在哪里生效 |
| --- | --- | --- |
| `retrieval_top_k_simple` | `3` | SINGLE 路径每个查询的 Hybrid 返回上限 |
| `retrieval_top_k_complex` | `8` | MULTI 路径每跳、每个查询的 Hybrid 返回上限 |
| `rrf_k` | `60` | RRF 排名融合公式中的常数 |
| `rerank_top_n` | `5` | 每次精排最多留下的片段数 |
| `max_retrieval_hops` | `3` | Self-RAG 最多检索几跳 |
| `local_embedding_model` | `BAAI/bge-small-zh-v1.5` | 默认本地向量模型 |
| `embedding_dim` | `512` | 默认向量维度 |
| `cross_encoder_model` | `BAAI/bge-reranker-base` | 默认本地精排模型 |

`top_k` 是“取前 k 个”，`top_n` 是“留下前 n 个”，这里两个名字对应不同阶段。SINGLE 可能执行两次 top 3 查询，合并后最多有 6 个不同片段，再选前 5；不能把它理解成整个 SINGLE 永远只有 3 个候选。

## 2. 入库：把 Markdown 拆成片段

启动构建检索组件时，[KnowledgeIngestor.load](../../app/retrieval/ingest.py) 读取知识目录，解析文件、合并并去重片段，然后调用 embedding provider 生成向量，最后建立 BM25 和向量索引。下文先看最容易读懂的 Markdown 分块逻辑；其他格式还经过 [unstructured_parser.py](../../app/retrieval/unstructured_parser.py)，不能把下面这个函数当成所有格式唯一的解析入口。

### 2.1 完整的 Markdown 分块函数

**真实源码：** [app/retrieval/chunking.py](../../app/retrieval/chunking.py) 中的 `chunk_markdown`。

```python
def chunk_markdown(
    text: str,
    *,
    document_id: str,
    title: str,
    category: DocumentCategory,
    source: str,
    policy_version: str = "v1",
) -> list[RetrievedChunk]:
    sections = _split_sections(text)
    chunks: list[RetrievedChunk] = []
    chunk_index = 0
    for section_title, body in sections:
        for piece in _window(body, MAX_CHUNK_CHARS, OVERLAP_CHARS):
            content = normalize_text(piece, join_hard_breaks=True)
            if not content:
                continue
            chunks.append(
                RetrievedChunk(
                    chunk_id=stable_chunk_id(document_id, chunk_index),
                    document_id=document_id,
                    title=title,
                    content=content,
                    score=0.0,
                    category=category,
                    section=normalize_text(section_title, join_hard_breaks=False) or section_title,
                    chunk_index=chunk_index,
                    policy_version=policy_version,
                    metadata={"source": source, "parser": "markdown"},
                    source="ingest",
                )
            )
            chunk_index += 1
    return chunks
```

输入 `text` 是整个文档的文本；输出是一个列表，其中每项是一个 `RetrievedChunk`。

按执行顺序读：

1. `_split_sections(text)` 先得到若干 `(章节标题, 正文)` 对。
2. 外层 `for section_title, body in sections` 取一个章节；这里一次把二元组的两个值赋给两个变量，叫“解包”。
3. 内层 `_window(...)` 再把这个章节的正文切成若干窗口。默认窗口是 **500 字符**，相邻窗口重叠 **60 字符**。这里是字符数，不是 token 数。
4. `normalize_text` 清洗正文；清洗后为空就 `continue`，跳过这一小段。
5. `chunks.append(...)` 把带来源信息的片段放入结果列表；入库时尚无检索分数，所以 `score=0.0`。
6. `chunk_index += 1` 等价于 `chunk_index = chunk_index + 1`，给下一个有效片段递增编号。

函数参数中的 `*` 表示它后面的参数必须按名字传，例如 `document_id="demo"`，避免把标题、来源等一串字符串位置传错。`-> list[RetrievedChunk]` 是返回类型提示，不是额外执行的一步。

**教学数据：** 假设输入正文为：

```markdown
# 退货政策
## 普通退货
符合条件的商品支持七天无理由退货。
## 质量问题
质量问题退货的运费由商家承担。
```

假设传入 `document_id="demo"`，两个非空章节都不足 500 字符，则主要输出是：

```text
chunks[0]: chunk_id="demo_00", section="普通退货", content="符合条件的商品支持七天无理由退货。"
chunks[1]: chunk_id="demo_01", section="质量问题", content="质量问题退货的运费由商家承担。"
```

主标题后没有正文时不会额外生成一个空片段。章节标题保存在 `section` 字段里；此函数没有把章节标题自动拼入 `content`。

### 2.2 分章节和滑动窗口究竟怎样执行

**真实源码：** 同文件的两个完整辅助函数。

```python
def _split_sections(text: str) -> list[tuple[str, str]]:
    lines = text.splitlines()
    sections: list[tuple[str, str]] = []
    current_title = "body"
    buffer: list[str] = []
    for line in lines:
        heading = _HEADING.match(line)
        if heading:
            if buffer:
                sections.append((current_title, "\n".join(buffer)))
                buffer = []
            current_title = heading.group(1).strip()
        else:
            buffer.append(line)
    if buffer:
        sections.append((current_title, "\n".join(buffer)))
    return sections or [("body", text)]


def _window(text: str, size: int, overlap: int) -> list[str]:
    cleaned = re.sub(r"\n{3,}", "\n\n", text).strip()
    if len(cleaned) <= size:
        return [cleaned] if cleaned else []
    pieces: list[str] = []
    start = 0
    while start < len(cleaned):
        end = min(len(cleaned), start + size)
        pieces.append(cleaned[start:end])
        if end == len(cleaned):
            break
        start = max(end - overlap, start + 1)
    return pieces
```

`_HEADING` 的定义是 `re.compile(r"^#{1,3}\s+(.*)$")`，识别行首的一至三个 `#` 加空白。因此当前逻辑只按 Markdown 1–3 级标题分章节。

`buffer` 暂存当前章节的正文行。遇到下一条标题，就先把旧正文存进 `sections`，清空 `buffer`，再切换 `current_title`。`"\n".join(buffer)` 把多行重新拼成带换行符的一段字符串。文件读完后还要存一次 `buffer`，否则最后一个章节会遗漏。

`_window` 的关键是 `cleaned[start:end]`：Python 切片含 `start`，不含 `end`。`start = end - overlap` 让下一段从上一段结束前 60 字符开始，以保留跨片段的上下文；`max(..., start + 1)` 保证起点始终向前，避免异常参数导致死循环。

**教学手算：** 假设正文有 1,000 字符，`size=500, overlap=60`：

| 次数 | `start` | `end` | 本次切片 | 下一次起点 |
| --- | --- | --- | --- | --- |
| 1 | 0 | 500 | `[0:500]` | `440` |
| 2 | 440 | 940 | `[440:940]` | `880` |
| 3 | 880 | 1000 | `[880:1000]` | 到末尾，停止 |

最后一片只有 120 字符，这是正常行为。窗口按字符截取，可能切断一句话；重叠可以缓解，但不是按句法或语义寻找边界。

### 2.3 ID 的稳定范围

**真实源码：**

```python
def stable_chunk_id(document_id: str, chunk_index: int) -> str:
    """IDs are deterministic across ingest runs so citations stay stable."""
    return f"{document_id}_{chunk_index:02d}"
```

`f"..."` 是格式化字符串；`{chunk_index:02d}` 把整数按至少两位显示，0 变成 `00`，9 变成 `09`，100 仍是 `100`。文档 ID 另外由相对文件路径计算得出。

“稳定”指同一份文件、同样分块结果再次入库时 ID 一致；它不是正文内容的指纹。往文档前面插入新章节后，后面片段编号可能变化。当前 Qdrant 路径执行 upsert，未在这里实现旧片段的全量清除或增量删除。

## 3. 查询改写：让口语问题变得适合检索

**真实源码：** [app/retrieval/query_rewrite.py](../../app/retrieval/query_rewrite.py) 的完整 `QueryRewriter.rewrite` 方法。

```python
    async def rewrite(self, query: str, *, missing_aspect: str = "") -> list[str]:
        if isinstance(self._llm, OfflineLLMProvider):
            return [query]
        try:
            response = await self._llm.generate(
                [
                    {"role": "system", "content": REWRITE_SYSTEM},
                    {
                        "role": "user",
                        "content": REWRITE_USER.format(
                            query=query, missing_aspect=missing_aspect or "none"
                        ),
                    },
                ],
                temperature=0.0,
                max_tokens=256,
                response_format={"type": "json_object"},
            )
            payload = extract_json_object(response.content)
            queries = payload.get("queries") or []
            cleaned = [str(item).strip() for item in queries if str(item).strip()]
            return cleaned[:3] or [query]
        except (LLMProviderError, LLMOutputError):
            return [query]
```

输入是用户问题 `query`，以及可选的 `missing_aspect`，表示“之前检索还缺什么”。输出是最多三个查询字符串。

`async def` 定义异步函数；`await self._llm.generate(...)` 表示等待模型回复，期间事件循环可以处理其他任务。它**不代表**后面多个检索查询会自动并行。

`self._llm` 是这个对象保存的模型 provider。`isinstance(...)` 检查它是不是离线 provider；如果是，就直接返回原问题，无须调模型。`try/except` 表示尝试调用并解析；遇到列出的模型调用或输出错误，退回 `[query]`。

模型调用要求返回 JSON。`payload.get("queries")` 从字典取 `queries`；没有或为空就用 `[]`。`[str(item).strip() for item in queries if str(item).strip()]` 是列表推导式：逐个转字符串、去掉首尾空白、过滤空字符串。`cleaned[:3] or [query]` 的意思是“清洗结果有内容则取前三项，否则保留原问题”。

**教学数据：**

```text
query = "拆封了还能退吗？"
假设模型 JSON：{"queries": ["已拆封商品退货条件", " 七天无理由退货适用条件 ", ""]}
输出：["已拆封商品退货条件", "七天无理由退货适用条件"]
```

`temperature=0.0` 是降低输出随机性的设置，不能据此保证改写永远正确。这里主要是提取 JSON 和清洗字符串，没有专门训练查询改写模型，也没有在这一层严格验证 `queries` 必须是字符串数组。

## 4. 在线入口：完整阅读 retrieval_node

路由器先决定策略；只有检索分支才会调用这个节点。SINGLE 在这里直接检索，MULTI 委托给 Self-RAG。问候的 DIRECT、业务操作的 AGENT、拒答的 REFUSE 由图走其他分支，不能把下面的 `else` 理解成所有策略都会检索。

**真实源码：** [app/agents/nodes/retrieval.py](../../app/agents/nodes/retrieval.py) 的完整函数。

```python
async def retrieval_node(
    state: AgentState,
    *,
    retriever: HybridRetriever,
    reranker: DocumentReranker,
    rewriter: QueryRewriter,
    self_rag: SelfRAGLoop,
    settings: Settings,
) -> dict:
    started = time.perf_counter()
    query = state["user_query"]
    strategy = state.get("strategy")
    top_k = (
        settings.retrieval_top_k_complex
        if strategy == RetrievalStrategy.MULTI
        else settings.retrieval_top_k_simple
    )
    if strategy == RetrievalStrategy.MULTI:
        chunks = await self_rag.retrieve(query, top_k=top_k)
        rewritten = [query]
    else:
        suggestions = await rewriter.rewrite(query)
        rewritten = list(dict.fromkeys([query, *suggestions]))[:2]
        by_id: dict[str, RetrievedChunk] = {}
        for search_query in rewritten:
            for candidate in await retriever.retrieve(search_query, top_k=top_k):
                by_id.setdefault(candidate.chunk_id, candidate)
        chunks = await reranker.rerank(query, list(by_id.values()), top_n=settings.rerank_top_n)
    latency = state.get("latency") or LatencyBreakdown()
    latency.retrieval_ms = (time.perf_counter() - started) * 1000
    return {"retrieved_chunks": chunks, "rewritten_queries": rewritten, "latency": latency}
```

`state` 是图节点之间传递的状态字典。`state["user_query"]` 要求键存在，缺失会抛错；`state.get("strategy")` 在键不存在时返回 `None`。多个服务通过参数传进来，因此节点无需自己创建模型、索引或连接。

`a if 条件 else b` 是 Python 条件表达式：MULTI 取 `8`，其他走到此处的检索路径取 `3`。MULTI 的具体循环稍后再读；先跟踪 SINGLE：

**教学数据：**

```text
query       = "拆封还能退吗"
suggestions = ["拆封还能退吗", "已拆封商品退货条件", "商品退货政策"]

[query, *suggestions]
→ ["拆封还能退吗", "拆封还能退吗", "已拆封商品退货条件", "商品退货政策"]

dict.fromkeys(...)       → 用字符串作为字典键，重复键只保留一次，保留首次出现顺序
list(...)               → 把这些键转回列表
[:2]                    → ["拆封还能退吗", "已拆封商品退货条件"]
```

所以原问题始终保留在第一位，随后最多带一个不同改写。若改写器只返回原问题，则只检索一次。

`by_id` 是用片段 ID 做键的字典。假设第一次结果为 A、B、C，第二次为 B、C、D：

| 步骤 | 操作 | `by_id` 里的键 |
| --- | --- | --- |
| 初始 | `{}` | 空 |
| 第一次查询 | 依次 `setdefault(A/B/C, 对象)` | A、B、C |
| 第二次查询 | B、C 已存在，保留原对象；加入 D | A、B、C、D |
| 精排 | `list(by_id.values())` | 取出这四个片段对象给精排模型 |

`setdefault` 不会把两次查询中同一片段的分数再融合一次，只是保留第一次看到的对象。后续精排用**原问题**重新打分，覆盖这些候选的旧分数。

两个查询通过普通 `for` 加 `await` 依次执行，当前代码没有 `asyncio.gather` 等并发调度。最后返回的是状态增量，里面含片段、对外记录的查询和耗时。MULTI 分支把 `rewritten_queries` 设为 `[query]`，因此这个字段并不展示 Self-RAG 内部每跳真正用过的改写。

## 5. BM25：按词匹配并排序

BM25 不会先把文本变成语义向量。它根据查询词在片段中是否出现、出现频率、词的区分度、片段长度等计算相关性。具体公式交给第三方 `BM25Okapi`；本项目负责分词、传数据和包装结果。

**真实源码：** [app/retrieval/bm25.py](../../app/retrieval/bm25.py) 的完整 `build` 和 `search` 方法。

```python
    def build(self, corpus: list[RetrievedChunk]) -> None:
        self._chunks = list(corpus)
        tokenized = [tokenize(chunk.content) for chunk in self._chunks]
        # BM25Okapi requires a non-empty corpus; keep None if ingest has not run.
        self._bm25 = BM25Okapi(tokenized) if tokenized else None

    def search(self, query: str, *, top_k: int) -> list[RetrievedChunk]:
        if self._bm25 is None or not self._chunks:
            return []
        scores = self._bm25.get_scores(tokenize(query))
        ranked = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)[:top_k]
        results: list[RetrievedChunk] = []
        for rank, index in enumerate(ranked, start=1):
            # BM25 IDF can be negative on tiny corpora; still return the ranked docs.
            chunk = self._chunks[index].model_copy(deep=True)
            chunk.score = float(scores[index])
            chunk.rank = rank
            chunk.source = "bm25"
            results.append(chunk)
        return results
```

`build` 通常在入库阶段执行一次；`search` 在每次在线检索时执行。`corpus` 就是全部片段组成的列表，`self._chunks` 保存它，保证打分数组的位置可以对应回原片段。

`tokenize` 位于 [tokenizer.py](../../app/retrieval/tokenizer.py)：优先使用 jieba 中文分词；没有 jieba 时，回退到相邻汉字二元组及英文数字词。精确分词结果要以实际运行的分词器为准，本文不假定某个中文词一定怎样被切开。

`get_scores` 为每个片段返回一个分数，所以 `scores[0]` 对应 `self._chunks[0]`。`sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)` 排的是**下标**，而不是复制排序所有片段：

**教学数据，分数为假设值：**

```text
self._chunks = [A, B, C]
scores       = [0.4, 2.1, 0.7]
range(...)   = [0, 1, 2]
降序后下标    = [1, 2, 0]
top_k=2      → ranked=[1, 2] → 取回 B、C
```

`lambda i: scores[i]` 是一个很短的匿名函数，意思是“比较下标 i 时，看它对应的分数”。`enumerate(ranked, start=1)` 同时提供从 1 开始的排名和原数组下标，因此第一轮是 `rank=1, index=1`，第二轮是 `rank=2, index=2`。排名和下标不是同一个数。

`model_copy(deep=True)` 复制结果对象，随后更新 `score/rank/source`，避免污染入库保存的原对象。这里**没有最低相关性分数门槛**，即使查询无有效匹配、分数为零或小语料下出现负分，也会按排名返回前若干片段。

## 6. 向量检索：先编码问题，再查近邻

向量是固定长度的数字数组。使用语义模型时，相近含义的文本通常映射到相近的位置。入库时文档正文已经编码；查询时只编码当前问题，再查向量库。

**真实源码：** [app/retrieval/vector.py](../../app/retrieval/vector.py) 的完整 `VectorRetriever.search` 方法。

```python
    async def search(self, query: str, *, top_k: int) -> list[RetrievedChunk]:
        vectors = await self.embedder.embed([query])
        if not vectors:
            return []
        # Qdrant client is sync; do not block the event loop.
        return await asyncio.to_thread(self.store.search, vectors[0], top_k=top_k)
```

这里传 `[query]`，因为 embedding API 支持一次编码一批文本。即使只有一个问题，也要传一个列表；对应输出是“向量列表”，所以 `vectors[0]` 才是本次问题的向量。

**教学数据，下面只用三维帮助理解，真实默认是 512 维：**

```text
query                        = "退款什么时候到账"
embedder.embed([query])       → [[0.2, 0.8, 0.1]]
vectors[0]                   →  [0.2, 0.8, 0.1]
store.search(..., top_k=3)    →  与该向量最接近的三个片段对象
```

`self.store` 可以是内存向量库或 Qdrant。`store.search` 是同步方法，`asyncio.to_thread(...)` 把它交给线程运行，避免在等待同步客户端时卡住异步事件循环。这里不是把所有查询自动并行化，只是把一次同步工作移到线程。

内存库把“找到最接近的向量”写得很直白，适合继续往下追。

**真实源码：** 同文件的完整 `InMemoryVectorStore.search` 方法。

```python
    def search(self, query_vector: list[float], *, top_k: int) -> list[RetrievedChunk]:
        scored: list[tuple[float, VectorRecord]] = [
            (cosine(query_vector, record.embedding), record) for record in self._records
        ]
        scored.sort(key=lambda item: item[0], reverse=True)
        results: list[RetrievedChunk] = []
        for rank, (score, record) in enumerate(scored[:top_k], start=1):
            chunk = record.chunk.model_copy(deep=True)
            chunk.score = float(score)
            chunk.rank = rank
            chunk.source = "vector"
            results.append(chunk)
        return results
```

`VectorRecord` 把一个片段 `chunk` 与其数字向量 `embedding` 放在一起。内存搜索遍历每条记录，为每条计算相似度，形成 `(分数, 记录)` 的列表，再按分数降序取前 k 个。它是全量遍历，不是近似近邻索引；使用 Qdrant 时搜索交给向量数据库。

**真实源码：** 这里调用的 `cosine`。

```python
def cosine(a: list[float], b: list[float]) -> float:
    return sum(x * y for x, y in zip(a, b, strict=False))
```

`zip(a, b)` 逐项配对，`sum(x * y ...)` 将每对乘积相加，也就是点积。对已经归一化的向量，点积等于余弦相似度；这个函数自身没有除以向量长度。默认本地模型会归一化，但接入其他 embedding 服务时不能忽略这个前提。

**教学手算，只用二维归一化向量：** 查询向量为 `[1, 0]`，A 为 `[0.8, 0.6]`，B 为 `[0, 1]`；则查询与 A 的分数为 `1×0.8+0×0.6=0.8`，与 B 为 `1×0+0×1=0`，A 排在 B 前。实际 512 维执行相同的逐项乘积求和，具体数字由模型产生。

**真实源码：** 默认本地 provider 的实际编码方法，位于 [app/providers/embeddings.py](../../app/providers/embeddings.py)。

```python
    @staticmethod
    def _encode(model: Any, texts: list[str]) -> list[list[float]]:
        raw = model.encode(
            texts,
            normalize_embeddings=True,
            show_progress_bar=False,
        )
        if hasattr(raw, "tolist"):
            raw = raw.tolist()
        if raw and isinstance(raw[0], int | float):
            return [[float(value) for value in raw]]
        return [[float(value) for value in row] for row in raw]
```

`@staticmethod` 表示此方法不需要对象的 `self`。`normalize_embeddings=True` 要求归一化向量；`tolist()` 把 NumPy 等数组变成普通 Python 列表；最后把每个数统一转为 `float`。这段同时容忍单个向量和一批向量的返回形状，最终统一成二维列表。

默认本地模型是 `BAAI/bge-small-zh-v1.5`。虽然 provider 的类名叫 `BgeM3EmbeddingProvider`，也不能据类名说默认模型是 BGE-M3。

当前 provider 的选择顺序是：配置完整的远端兼容 embedding API → 本地模型成功加载 → hash 占位向量。Hash 是把文本哈希成数字，**不能视为可靠语义检索**。测试环境默认不加载本地权重。已选远端 API 后，如果请求失败，这个 provider 不会在请求内部重新选择本地模型；在线 Hybrid 可继续依赖 BM25，启动入库则需要先成功完成向量生成。

## 7. Hybrid：两路检索汇合到哪里

下面是 `HybridRetriever.retrieve` 中真实的核心连续片段，省略了函数前面的缓存检查和局部变量初始化，以及后面的耗时日志、结果截断和缓存写入。完整方法见 [app/retrieval/hybrid.py](../../app/retrieval/hybrid.py)。**这是明确节选，不是伪造的完整函数。**

```python
        try:
            lexical = self._bm25.search(query, top_k=top_k)
        except Exception as exc:  # BM25 is in-process; unexpected errors still isolated
            lexical_error = exc
            logger.exception("bm25_failed")
        try:
            dense = await self._vector.search(query, top_k=top_k)
        except Exception as exc:
            dense_error = exc
            logger.exception("vector_search_failed")
        if not lexical and not dense:
            raise RetrievalError("both lexical and dense retrievers failed") from (
                dense_error or lexical_error
            )
        fused = reciprocal_rank_fusion([lexical, dense], k=self._rrf_k)
```

`lexical` 是 BM25 的片段列表，`dense` 是向量检索的片段列表。两个 `try` 分开，所以一路报错不会阻止另一条路尝试检索。两路都没有结果时抛出 `RetrievalError`；注意这也涵盖两路正常返回空列表，异常消息中的 “failed” 不一定表示两个后端都发生故障。

代码先做 BM25，再等待向量查询，属于顺序执行。之后把两份排名传给 RRF，取融合结果的前 `top_k` 个。默认还有 60 秒缓存，缓存 key 包含 query 和 top_k；明显含订单的查询会绕过缓存，避免缓存用户业务结果。

## 8. RRF：完整函数和一次手算

BM25 和向量的原始分数不是一个量纲。RRF（Reciprocal Rank Fusion，倒数排名融合）只看“某个片段在每份结果里排第几”，不直接相加原始分数。

**真实源码：** [app/retrieval/fusion.py](../../app/retrieval/fusion.py) 的完整函数。`DEFAULT_RRF_K` 在 [constants.py](../../app/core/constants.py) 中为 `60`。

```python
def reciprocal_rank_fusion(
    ranked_lists: list[list[RetrievedChunk]],
    *,
    k: int = DEFAULT_RRF_K,
) -> list[RetrievedChunk]:
    if k <= 0:
        raise ValueError("RRF k must be positive")
    scores: dict[str, float] = {}
    canonical: dict[str, RetrievedChunk] = {}
    for ranked in ranked_lists:
        for rank_position, chunk in enumerate(ranked, start=1):
            scores[chunk.chunk_id] = scores.get(chunk.chunk_id, 0.0) + 1.0 / (k + rank_position)
            if chunk.chunk_id not in canonical:
                canonical[chunk.chunk_id] = chunk.model_copy(deep=True)
    fused = []
    for chunk_id, score in sorted(scores.items(), key=lambda item: item[1], reverse=True):
        chunk = canonical[chunk_id]
        chunk.score = score
        chunk.source = "rrf"
        fused.append(chunk)
    for index, chunk in enumerate(fused, start=1):
        chunk.rank = index
    return fused
```

两个字典分别负责两件事：`scores` 保存“片段 ID → 累计融合分数”；`canonical` 保存“片段 ID → 第一次看到的片段对象副本”。分数会累加，标题、正文等元数据沿用第一次看到的对象。

公式是：

```text
RRF(某片段) = Σ 1 / (k + 该片段在一路结果中的排名)
```

片段没出现在某一路，该路就没有贡献。`enumerate(..., start=1)` 读取的是列表顺序，而不是片段旧的 `rank` 字段。

**教学数据：** 假设 BM25 排名为 `[A, B]`，向量排名为 `[B, C]`，`k=60`：

| 片段 | BM25 的贡献 | 向量路的贡献 | 累加分数 |
| --- | --- | --- | --- |
| A | `1/(60+1)` | 未出现，0 | `0.016393` |
| B | `1/(60+2)` | `1/(60+1)` | `0.032522` |
| C | 未出现，0 | `1/(60+2)` | `0.016129` |

执行中的变量变化是：

```text
初始：scores={}
处理 BM25 A：scores={A: 0.016393}
处理 BM25 B：scores={A: 0.016393, B: 0.016129}
处理向量 B：scores={A: 0.016393, B: 0.032522}
处理向量 C：scores={A: 0.016393, B: 0.032522, C: 0.016129}
按分数降序：B、A、C
重新设置 rank：B=1、A=2、C=3
```

`scores.get(chunk.chunk_id, 0.0)` 表示第一次遇到某个 ID 时，把旧分数当 0；再次遇到就用已有分数。`scores.items()` 返回 `(键, 值)` 对，`lambda item: item[1]` 按其中的值，即分数排序。

这里每一路等权，没有根据“向量模型是否退化成 hash”动态降低权重。因此“融合了两路”本身不保证更准；如果一路很弱，融合也可能引入噪声。

## 9. 精排：对较少候选做更细判断

召回阶段尽量别漏，精排阶段再从少量候选里挑最相关的。这里的 `DocumentReranker` 负责把 provider 返回的“原位置与分数”转换回片段列表。

**真实源码：** [app/retrieval/reranker.py](../../app/retrieval/reranker.py) 的完整方法。

```python
    async def rerank(
        self,
        query: str,
        chunks: list[RetrievedChunk],
        *,
        top_n: int,
    ) -> list[RetrievedChunk]:
        if not chunks:
            return []
        documents = [chunk.content for chunk in chunks]
        ranked = await self._provider.rerank(query, documents, top_n=min(top_n, len(chunks)))
        results: list[RetrievedChunk] = []
        for new_rank, (original_index, score) in enumerate(ranked, start=1):
            chunk = chunks[original_index].model_copy(deep=True)
            chunk.score = score
            chunk.rank = new_rank
            chunk.source = "rerank"
            results.append(chunk)
        return results
```

`documents` 只保留正文字符串交给模型，片段对象仍在 `chunks` 里。`min(top_n, len(chunks))` 保证只剩两个候选时，不要求模型返回五个。

**教学数据，分数为假设值：**

```text
chunks    = [A, B, C]
documents = [A.content, B.content, C.content]
provider 返回 ranked = [(2, 0.91), (0, 0.75)]
```

`(2, 0.91)` 表示“原列表下标 2 的 C 得分 0.91”，不是“排名第二”。进入循环后，第一轮 `new_rank=1, original_index=2, score=0.91`，因此输出第一个是 C；第二轮取 A。最终 `[C, A]` 的 `rank` 分别是 1、2。

实际打分在另一个 provider 中。

**真实源码：** [app/providers/reranker.py](../../app/providers/reranker.py) 的完整 `CrossEncoderRerankProvider.rerank` 方法。

```python
    async def rerank(
        self,
        query: str,
        documents: list[str],
        *,
        top_n: int,
    ) -> list[tuple[int, float]]:
        model = self._ensure_model()
        if model is None:
            return await self._fallback.rerank(query, documents, top_n=top_n)
        pairs = [(query, document) for document in documents]
        scores = await asyncio.to_thread(self._predict, model, pairs)
        ranked = sorted(enumerate(scores), key=lambda item: item[1], reverse=True)
        return [(index, score) for index, score in ranked[:top_n]]
```

Cross-Encoder 把“问题 + 一段文档”作为一对联合打分，因此 `pairs` 类似 `[(问题, A正文), (问题, B正文)]`。相比入库时分别编码文档、查询时分别编码问题的双塔向量召回，它更适合候选较少的精排阶段。

默认模型是 `BAAI/bge-reranker-base`，最大输入长度配置为 512。模型未加载成功或被关闭时，`_fallback` 使用字符集合重叠打分；这个降级算法不是 Cross-Encoder。测试环境默认走降级。当前 `model is None` 分支处理的是模型不可用；已加载模型在 `_predict` 中发生运行异常时，这里没有 `try/except` 再切换降级算法。

## 10. Self-RAG：资料不够，就带着“缺什么”再检索

前面的 SINGLE 只走一轮流程。复杂问题可能需要总则、例外、运费等多条规则，MULTI 因此会进入 `SelfRAGLoop`。

这里的反思是“调用普通 LLM，要求结构化 JSON 判断证据”，没有训练 Self-RAG 论文中的特殊反思 token。`_grade` 检查的是证据是否够用，最终回答生成后的忠实度检查是另外一个节点。

### 10.1 完整的 retrieve 方法

**真实源码：** [app/retrieval/self_rag.py](../../app/retrieval/self_rag.py)。

```python
    async def retrieve(
        self,
        query: str,
        *,
        top_k: int,
    ) -> list[RetrievedChunk]:
        accumulated: dict[str, RetrievedChunk] = {}
        current_query = query
        missing = ""
        for _hop in range(self._settings.max_retrieval_hops):
            rewritten = await self._rewriter.rewrite(current_query, missing_aspect=missing)
            queries = list(dict.fromkeys([query, *rewritten]))[:2]
            candidates_by_id: dict[str, RetrievedChunk] = {}
            for search_query in queries:
                for candidate in await self._retriever.retrieve(search_query, top_k=top_k):
                    candidates_by_id.setdefault(candidate.chunk_id, candidate)
            reranked = await self._reranker.rerank(
                query, list(candidates_by_id.values()), top_n=self._settings.rerank_top_n
            )
            for chunk in reranked:
                accumulated[chunk.chunk_id] = chunk
            grade = await self._grade(query, list(accumulated.values()))
            if grade.sufficient:
                break
            missing = grade.missing_aspect or "more specific policy conditions"
            current_query = query
        return list(accumulated.values())[: self._settings.rerank_top_n * 2]
```

先区分三个容器：

| 变量 | 保存什么 | 生命周期 |
| --- | --- | --- |
| `queries` | 本跳最多两个查询字符串 | 每跳重新生成 |
| `candidates_by_id` | 本跳召回的候选片段，按 ID 去重 | 每跳清空重建 |
| `accumulated` | 各跳精排后留下的片段，按 ID 累积 | 整次多跳调用一直保留 |

`range(3)` 会产生 0、1、2，因此默认最多执行三次循环。变量 `_hop` 的下划线表示作者没有使用这个计数值，它没有额外的 Python 特殊行为。

每跳先用 `missing` 指导改写，再重复“原问题 + 一个不同改写 → Hybrid → 精排”。本跳精排结果写进 `accumulated`；同 ID 会更新对象，但字典已有键的顺序不改变。`_grade` 看到的是累计证据，而不是只看最新一跳。

`if grade.sufficient: break` 在资料足够时提前结束，避免一定跑满三跳。如果不够，把 `missing_aspect` 写进下一跳的 `missing`。`current_query` 最后又赋回原问题，所以下一跳的变化主要来自 `missing_aspect`，不是把上次改写反复改写下去。

**教学过程，模型回复与召回结果均是假设：**

问题：`定制杯子有质量问题，过了七天还能退吗，运费谁出？`

| 时刻 | `missing` | 本次新增证据 | 评分结果 | `accumulated` |
| --- | --- | --- | --- | --- |
| 第 1 跳前 | 空字符串 | 尚未检索 | — | 空 |
| 第 1 跳后 | 将更新为“质量问题例外及运费” | A：定制商品无理由退货限制 | `sufficient=false` | A |
| 第 2 跳后 | 带着上一行缺失点改写 | B：质量问题售后规则；C：运费承担 | `sufficient=true` | A、B、C |
| 返回 | — | 结束，无第 3 跳 | — | `[A, B, C]` |

这个例子用于说明变量流动，不表示知识库真实条款或模型一定做出这样的判断。若三跳后仍不足，代码还是返回现有片段，由后续生成链路处理；这个方法本身不返回一个“最终充分性状态”。

### 10.2 完整的评分模型与 _grade 方法

**真实源码：** 同文件。

```python
class GradeResult(BaseModel):
    relevant: bool
    sufficient: bool
    missing_aspect: str = ""
    rationale: str = ""
```

```python
    async def _grade(self, query: str, chunks: list[RetrievedChunk]) -> GradeResult:
        evidence = "\n\n".join(f"[{c.chunk_id}] {c.content}" for c in chunks[:6]) or "(empty)"
        try:
            response = await self._llm.generate(
                [
                    {"role": "system", "content": GRADE_SYSTEM},
                    {"role": "user", "content": GRADE_USER.format(query=query, evidence=evidence)},
                ],
                temperature=0.0,
                max_tokens=256,
                response_format={"type": "json_object"},
            )
            return parse_model(response.content, GradeResult)
        except (LLMProviderError, LLMOutputError):
            return GradeResult(relevant=bool(chunks), sufficient=bool(chunks), rationale="fallback")
```

`BaseModel` 来自 Pydantic，把 JSON 输出转换并校验成有字段的对象，所以后面能写 `grade.sufficient`。模型被要求返回类似这样的内容：

```json
{"relevant": true, "sufficient": false, "missing_aspect": "质量问题例外及运费承担", "rationale": "现有证据只有无理由退货限制"}
```

这段 JSON 同样是**教学数据**。`relevant` 表示证据是否相关，`sufficient` 表示是否足够覆盖问题；但当前循环实际只根据 `sufficient` 分支，未单独用 `relevant` 过滤片段。

`chunks[:6]` 只拿前六个片段。`f"[{c.chunk_id}] {c.content}"` 给每段加上来源 ID；`"\n\n".join(...)` 用空行分隔，拼成一次模型输入。没有片段时用 `"(empty)"` 占位。

`parse_model` 若不能得到符合 `GradeResult` 的对象，会按模型输出错误处理。发生列出的调用或解析错误时，`bool(chunks)` 把“是否有结果”转换成真假：非空列表为 `True`。因此降级行为是“有资料就认为足够并停止”，这是减少空转的可用性选择，并不保证资料真的齐全。

### 10.3 当前实现的几个重要边界

- **串行执行。** 每跳最多两个查询，每个查询内 BM25 和向量也是顺序执行。`async/await` 不能直接等同于并行。
- **没有最终统一精排。** 每跳分别精排，`accumulated` 按首次插入顺序保存，最后只截取前 `rerank_top_n * 2`，默认最多 10 条。这 10 条不是跨所有跳重新排序后的全局 top 10。
- **后几条证据可能无法参与判断。** `_grade` 只读取累计结果前 6 条；[generate_node](../../app/agents/nodes/generate.py) 同样只把前 6 条检索片段放入证据文本。假设第一跳已有 A–E，第二跳新增 F–J，评分和最终生成看到的只有 A–F，G–J 即使有用也可能没有进入 prompt。第三跳补充的片段也可能排在后面。
- **停止条件不等于严谨证明。** 模型对 `sufficient` 的判断会出错，异常降级又倾向提前停止；三跳上限只是成本和时延保护。
- **Self-RAG 发生在生成前。** 不要把它和生成答案后检查忠实度混成同一个循环。

## 11. 把整条链路连起来再读一次

下面是**教学流程描述，不是仓库中的可执行代码**：

```text
启动：知识文件 → 解析/清洗/切块 → BM25 建库 + 文档向量入库

用户问题 → Adaptive 路由
  SINGLE：原问题 + 一个不同改写
          → 每个查询 BM25 + 向量 → RRF → 跨查询按 ID 去重
          → 用原问题精排 → 证据交给生成节点
  MULTI ：上述检索和精排
          → 累积证据 → JSON 评分
          → 不足则带着缺失方面继续，最多三跳
          → 证据交给生成节点
```

业务工具分支中的 `search_knowledge` 另有一条较短路径：直接 Hybrid top 4，不调用这里的改写器、精排和 Self-RAG，见 [app/tools/knowledge_tools.py](../../app/tools/knowledge_tools.py)。因此理解某次请求时，先确认它到底走了 SINGLE、MULTI 还是 AGENT，再判断哪些函数会执行。

最后可以用四个问题检查自己是否读懂：`chunk_id` 怎样用于两层去重？为什么 BM25 分数不能和向量分数直接相加？`original_index` 与 `new_rank` 为什么不同？Self-RAG 下一跳真正变化的是哪个变量？对应答案分别是：字典键、不同量纲、原候选位置与新排名、`missing_aspect` 指导的改写。
