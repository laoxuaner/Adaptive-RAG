# 生成、忠实度检查、记忆与流式输出

[返回阅读路线](README.md) · [上一篇：工具与确认](03-tools-and-confirmation.md)

前面的节点已经把政策资料放入 `state["retrieved_chunks"]`，或者把业务查询结果放入 `state["tool_results"]`。本文继续追踪：这些数据怎样变成模型输入，模型文本怎样变成可靠程度更高的接口回答，以及历史和 SSE 在当前实现中实际做了什么。

源码块从当前工作区提取。教学 JSON、变量值和改写内容均为示意，没有调用模型或执行订单变更。函数片段依赖原模块的导入与其他定义。

## 1. 模型输出先约定一个结构

来源：[AnswerPayload](../../app/agents/nodes/generate.py)。

```python
class AnswerPayload(BaseModel):
    answer: str
    citation_chunk_ids: list[str] = Field(default_factory=list)
    refused: bool = False
```

`answer` 是正文，`citation_chunk_ids` 是引用的片段编号，`refused` 表示模型是否明确拒答。`Field(default_factory=list)` 表示每次创建对象时得到一个新的空列表。

教学示意，模型应返回类似文本：

```json
{
  "answer": "定制商品一般不适用七天无理由退货；质量问题另按相关政策处理。",
  "citation_chunk_ids": ["示意片段_A"],
  "refused": false
}
```

JSON 文本还不是 Python 对象。需要先解析，再验证字段。调用失败和结构错误是两个不同的故障点。

## 2. 先把完整生成函数放出来

来源：[generate_node](../../app/agents/nodes/generate.py)。

```python
async def generate_node(state: AgentState, llm: LLMProvider) -> dict:
    started = time.perf_counter()
    chunks = state.get("retrieved_chunks") or []
    tools = state.get("tool_results") or []
    if tools and all(not tool.success for tool in tools):
        return {
            "final_answer": "暂时无法查到相关业务数据，请核对订单号或稍后重试。",
            "citations": [],
        }
    pending = state.get("pending_action")
    if (state.get("extra") or {}).get("missing_order_id"):
        return {"final_answer": "请提供要处理的订单号，确认后我才能继续。", "citations": []}
    if pending:
        answer = pending.summary
        latency = state.get("latency") or LatencyBreakdown()
        latency.llm_ms = (time.perf_counter() - started) * 1000
        return {
            "final_answer": answer,
            "citations": [],
            "refused": False,
            "latency": latency,
        }
    evidence = (
        "\n\n".join(f"[{c.chunk_id}] {c.title}\n{c.content}" for c in chunks[:6]) or "(无检索结果)"
    )
    tool_text = json.dumps([t.model_dump() for t in tools], ensure_ascii=False) if tools else "(无)"
    intent = state.get("intent")
    strategy = state.get("strategy") or RetrievalStrategy.DIRECT
    if isinstance(llm, OfflineLLMProvider):
        answer, citation_ids = _offline_answer(state)
        citations = [
            Citation(
                document_id=chunk.document_id,
                title=chunk.title,
                chunk_id=chunk.chunk_id,
                section=chunk.section,
            )
            for chunk in chunks
            if chunk.chunk_id in citation_ids
        ]
        return {"final_answer": answer, "citations": citations, "refused": False}
    messages = [
        {"role": "system", "content": ANSWER_SYSTEM},
        {
            "role": "user",
            "content": ANSWER_USER.format(
                query=state["user_query"],
                intent=intent.label.value if intent else "unknown",
                strategy=strategy.value,
                evidence=evidence,
                tool_results=tool_text,
            ),
        },
    ]
    try:
        response = await llm.generate(
            messages,
            temperature=0.2,
            max_tokens=700,
            response_format={"type": "json_object"},
        )
        payload = parse_model(response.content, AnswerPayload)
        usage = response.usage
    except (LLMProviderError, LLMOutputError):
        payload = AnswerPayload(
            answer=_extractive_fallback(state["user_query"], evidence, tool_text),
            citation_chunk_ids=[chunks[0].chunk_id] if chunks else [],
        )
        usage = TokenUsage()
    citations = [
        Citation(
            document_id=chunk.document_id,
            title=chunk.title,
            chunk_id=chunk.chunk_id,
            section=chunk.section,
        )
        for chunk in chunks
        if chunk.chunk_id in set(payload.citation_chunk_ids)
    ]
    latency = state.get("latency") or LatencyBreakdown()
    latency.llm_ms = (time.perf_counter() - started) * 1000
    prev_usage = state.get("token_usage") or TokenUsage()
    merged = TokenUsage(
        prompt_tokens=prev_usage.prompt_tokens + usage.prompt_tokens,
        completion_tokens=prev_usage.completion_tokens + usage.completion_tokens,
        total_tokens=prev_usage.total_tokens + usage.total_tokens,
    )
    return {
        "final_answer": payload.answer,
        "citations": citations,
        "refused": payload.refused,
        "latency": latency,
        "token_usage": merged,
    }
```

这段比较长，可以按下面六段读，而不是一次记下所有变量。

### 2.1 提前返回：有些情况不用调用模型

`chunks` 是检索证据，`tools` 是工具结果。`state.get(...) or []` 让缺失值和 None 退到空列表。

`tools and all(not tool.success for tool in tools)` 表示“有工具结果，并且所有工具都失败”。前面的 `tools and` 很重要：Python 的 `all([])` 会返回 True，单独使用 `all` 会把没有工具的普通问答也判断成所有工具失败。

`missing_order_id` 表示业务需要订单号但没提取到，直接要求补充。存在 `pending_action` 时直接返回确认摘要，不让生成模型声称操作已经完成。

`return` 会立刻结束本次函数。后面的 Prompt 构造不会再执行。

### 2.2 将证据对象变成字符串

`chunks[:6]` 取前六个片段。每个 `c` 是 `RetrievedChunk` 对象，因此用 `c.chunk_id`、`c.title`、`c.content` 读取。

`f"[{c.chunk_id}] {c.title}\n{c.content}"` 将一个片段整理成带编号的文本。`"\n\n".join(...)` 用空行拼接它们。末尾 `or "(无检索结果)"` 处理没有内容的情况。

教学示意，拆开写相当于：

```python
parts = []
for chunk in chunks[:6]:
    part = f"[{chunk.chunk_id}] {chunk.title}\n{chunk.content}"
    parts.append(part)
evidence = "\n\n".join(parts) or "(无检索结果)"
```

工具结果的 `model_dump()` 先把 Pydantic 模型转为字典，`json.dumps(..., ensure_ascii=False)` 再转成便于阅读的 JSON 文本，保留中文而不强制转为 Unicode 转义。

### 2.3 离线 provider 使用确定的资料摘录

当 `llm` 是 `OfflineLLMProvider` 时，进入 `_offline_answer`。订单状态使用白名单字段，知识问答提取召回原文。该分支不会请求 DeepSeek，也不会假装得到完整的生成回答。

离线分支是 provider 类型判断。配置了在线模型但调用暂时失败时，走的是后面的异常兜底，不能把所有失败都理解为切换成 OfflineLLMProvider。

### 2.4 构造 messages：RAG 的关键连接点

`messages` 这里只有两条：系统规则和本轮填好模板的用户消息。`ANSWER_USER.format(...)` 把问题、意图、策略、证据、工具结果填进字符串模板。

`intent.label.value if intent else "unknown"` 是条件表达式：有意图对象就取枚举值，否则写 unknown。`strategy.value` 则把策略枚举转成如 `single` 的字符串。

这是 RAG 的实际数据连接：检索的 `chunks` 被转成 `evidence`，填进 Prompt，随问题一起发给模型。检索器不是在后台自动改变模型知识，也没有在此训练模型。

### 2.5 调模型、解析、兜底

`llm.generate` 返回一个响应对象，`.content` 是文本，`.usage` 是这次调用的 token 用量。`max_tokens=700` 限制生成长度，`temperature=0.2` 调低采样随机性。

`response_format` 是请求模型输出 JSON 的约束意图，之后仍需解析与校验，不能假定返回一定符合类型。

指定的模型调用异常或输出异常发生时，构造 `AnswerPayload`，改为资料原文摘录。`TokenUsage()` 表示该分支没有拿到本次成功响应的用量，不表示上游所有尝试都没有成本。

### 2.6 生成引用与节点更新

`for chunk in chunks if chunk.chunk_id in set(payload.citation_chunk_ids)` 只从实际召回列表构造引用。因此模型编一个不在列表里的 ID，不会直接变成有效的 Citation。

但这只约束“引用编号确实来自证据”，不证明正文的每句话都被该片段支持。后续忠实度检查专门处理这个问题。

最后 `return` 的字典更新 `final_answer`、`citations`、`refused`、`latency` 和 `token_usage`，供后面的图节点与 `ChatService._to_response` 使用。

这里的 token 累加主要覆盖本生成节点拿到的 usage。意图、路由、改写、裁判等调用没有在所有路径统一累加到该字段，不能把 API 返回的这个数当作整条链路精确账单。

## 3. 看看模型实际收到的模板

来源：[回答 Prompt 模块](../../app/prompts/answer.py)。

```python
"""Answer generation prompt."""

from app.core.constants import UNTRUSTED_CONTEXT_BANNER

ANSWER_SYSTEM = f"""你是电商平台客服助手。用简体中文回答，语气克制、可执行。
{UNTRUSTED_CONTEXT_BANNER}

规则：
- 只根据提供的证据、工具结果和明确政策回答
- 证据不足就说不确定，并给出下一步（补订单号/转人工）
- 不要编造运单号、金额、时效
- 引用时使用 chunk_id
- 对退款/取消等不可逆操作，只说明将发起确认，不要声称已经完成
"""

ANSWER_USER = """用户问题：{query}
意图：{intent}
策略：{strategy}

检索证据：
{evidence}

工具结果：
{tool_results}

输出 JSON：
{{"answer":"...","citation_chunk_ids":["..."],"refused":false}}
"""
```

`ANSWER_SYSTEM` 给出回答约束，`ANSWER_USER` 是待填的模板。`{query}` 是替换位置；JSON 例子中的 `{{` 与 `}}` 是为 `.format` 转义，最终会变回普通的大括号。

`UNTRUSTED_CONTEXT_BANNER` 提醒模型把检索内容当数据，不要执行其中的指令。它是一条模型层指令，还需要配合入站检查、工具权限与业务校验，不是单独充分的安全保证。

## 4. llm.generate 最后怎样变成网络请求

来源：[DeepSeekProvider.generate](../../app/providers/deepseek.py)。

```python
async def generate(
    self,
    messages: list[dict[str, str]],
    *,
    temperature: float = 0.0,
    max_tokens: int = 1024,
    response_format: dict[str, str] | None = None,
) -> LLMResponse:
    api_key = self._settings.deepseek_api_key.get_secret_value()
    if not api_key:
        raise LLMProviderError("DeepSeek API key is not configured")

    payload: dict[str, object] = {
        "model": self._settings.deepseek_model,
        "messages": messages,
        "temperature": temperature,
        "max_tokens": max_tokens,
    }
    if response_format is not None:
        payload["response_format"] = response_format

    async def _call() -> LLMResponse:
        try:
            response = await self._client.post(
                "/chat/completions",
                headers={"Authorization": f"Bearer {api_key}"},
                json=payload,
            )
        except httpx.TimeoutException as exc:
            raise LLMProviderError("DeepSeek request timed out", timeout=True) from exc
        except httpx.HTTPError as exc:
            raise LLMProviderError("DeepSeek transport error") from exc

        if response.status_code in {400, 401, 403, 404, 422}:
            raise LLMProviderError(
                f"DeepSeek HTTP {response.status_code}",
                retryable=False,
            )
        if response.status_code == 429:
            raise LLMProviderError("DeepSeek rate limited", rate_limited=True)
        if is_retryable_status(response.status_code):
            raise LLMProviderError(f"DeepSeek HTTP {response.status_code}")
        if response.status_code >= 400:
            raise LLMProviderError(f"DeepSeek HTTP {response.status_code}")

        body = response.json()
        try:
            content = body["choices"][0]["message"]["content"]
        except (KeyError, IndexError, TypeError) as exc:
            raise LLMProviderError("DeepSeek response missing choices") from exc
        usage_raw = body.get("usage") or {}
        usage = TokenUsage(
            prompt_tokens=int(usage_raw.get("prompt_tokens") or 0),
            completion_tokens=int(usage_raw.get("completion_tokens") or 0),
            total_tokens=int(usage_raw.get("total_tokens") or 0),
        )
        return LLMResponse(
            content=content or "", usage=usage, model=self._settings.deepseek_model
        )

    return await with_exponential_backoff(
        _call,
        max_attempts=self._settings.llm_max_retries,
        operation_name="deepseek.generate",
    )
```

`payload` 是将要发送的 JSON。`self._client.post(...)` 才是真正向模型服务发送请求的地方。源代码的 `api_key` 是变量，这里没有读取或展示实际密钥。

里面定义了一个 `_call` 异步函数。定义它不会立刻请求；函数末尾的 `with_exponential_backoff` 才按重试策略调用它。

请求超时与 HTTP 传输异常被包装为 `LLMProviderError`。400/401/403 等参数或认证类失败被标记为不重试；429、部分服务端错误可以走重试。具体是否重试还由异常属性和重试器决定。

成功时，`body["choices"][0]["message"]["content"]` 从服务响应取出文本。缺字段则抛出 provider 异常，而不是让上层误把空数据当正常答案。

`LLMProvider` 是项目定义的接口约定，不是一个会自动选模型的魔法对象。启动容器选择具体的 DeepSeek 或 Offline 实现，生成节点再调用共同的方法。

## 5. 文本怎样被安全地解析为模型对象

来源：[结构化输出解析](../../app/providers/json_parser.py)。

```python
def extract_json_object(text: str) -> dict[str, Any]:
    stripped = text.strip()
    fenced = _FENCE.search(stripped)
    if fenced:
        stripped = fenced.group(1).strip()
    try:
        payload = json.loads(stripped)
        if isinstance(payload, dict):
            return payload
    except json.JSONDecodeError:
        pass
    start = stripped.find("{")
    end = stripped.rfind("}")
    if start == -1 or end == -1 or end <= start:
        raise LLMOutputError("LLM output did not contain a JSON object")
    try:
        payload = json.loads(stripped[start : end + 1])
    except json.JSONDecodeError as exc:
        raise LLMOutputError("LLM output JSON is malformed") from exc
    if not isinstance(payload, dict):
        raise LLMOutputError("LLM JSON root is not an object")
    return payload
```

```python
def parse_model(text: str, model_type: type[T]) -> T:
    payload = extract_json_object(text)
    try:
        return model_type.model_validate(payload)
    except ValidationError as exc:
        raise LLMOutputError(f"LLM JSON failed schema validation: {exc}") from exc
```

第一层先尝试去掉 Markdown 围栏、直接解析 JSON；失败后尝试从第一个 `{` 到最后一个 `}` 提取对象。若格式仍不正确或顶层不是对象，就抛 `LLMOutputError`。

第二层调用 `model_validate`，检查字段、类型、枚举和必填项。`parse_model(response.content, AnswerPayload)` 的含义是“将这段文本按 AnswerPayload 的要求解析”。

`TypeVar` 等泛型注解是为了让类型检查器知道：传入什么模型类型，就返回那个模型的对象。初读时可以先把它理解为带校验的解析函数。

## 6. 忠实度裁判检查什么

来源：[FaithfulnessResult](../../app/guardrails/hallucination.py) 和检查方法。

```python
class FaithfulnessResult(BaseModel):
    supported: bool
    hallucinated_spans: list[str] = Field(default_factory=list)
    rationale: str = ""
```

```python
async def check(
    self,
    *,
    query: str,
    answer: str,
    chunks: list[RetrievedChunk],
    tool_results_text: str,
) -> FaithfulnessResult:
    evidence = "\n".join(f"[{c.chunk_id}] {c.content}" for c in chunks[:8]) or "(none)"
    try:
        response = await self._llm.generate(
            [
                {"role": "system", "content": FAITHFULNESS_SYSTEM},
                {
                    "role": "user",
                    "content": FAITHFULNESS_USER.format(
                        query=query,
                        answer=answer,
                        evidence=evidence,
                        tool_results=tool_results_text or "(none)",
                    ),
                },
            ],
            temperature=0.0,
            max_tokens=256,
            response_format={"type": "json_object"},
        )
        return parse_model(response.content, FaithfulnessResult)
    except (LLMProviderError, LLMOutputError):
        return FaithfulnessResult(supported=False, rationale="checker_unavailable")
```

裁判收到原问题、答案、证据和工具文本，再给出 `supported`。`hallucinated_spans` 记录不受支持的内容，`rationale` 是理由。

若请求或解析失败，返回 `supported=False, rationale="checker_unavailable"`。这使后续代码能区分“模型认为证据不支持”和“裁判根本没成功工作”。

这里没有外部事实数据库或独立人工核验，裁判仍是 LLM。supported 表示该裁判对提供证据的判断，不能直接等价为现实世界中必然正确。

## 7. 检查不通过怎样处理

来源：[faithfulness_node](../../app/agents/nodes/faithfulness.py)。

```python
async def faithfulness_node(
    state: AgentState, checker: FaithfulnessChecker, llm: LLMProvider
) -> dict:
    answer = state.get("final_answer") or ""
    if (
        state.get("refused")
        or state.get("pending_action")
        or (state.get("extra") or {}).get("missing_order_id")
    ):
        return {}
    chunks = state.get("retrieved_chunks") or []
    tools = state.get("tool_results") or []
    if isinstance(llm, OfflineLLMProvider) or state.get("strategy") == RetrievalStrategy.DIRECT:
        return {}
    if state.get("strategy") in {RetrievalStrategy.SINGLE, RetrievalStrategy.MULTI} and not chunks:
        return {
            "final_answer": "现有资料不足，暂时无法可靠回答。请补充具体商品或政策信息。",
            "citations": [],
            "refused": True,
        }
    tool_text = str([t.model_dump() for t in tools])
    result = await checker.check(
        query=state["user_query"],
        answer=answer,
        chunks=chunks,
        tool_results_text=tool_text,
    )
    extra = {"faithfulness": result.model_dump(), "self_corrected": False}
    if result.supported:
        return {"extra": extra}
    HALLUCINATION_FLAGS.inc()
    if result.rationale == "checker_unavailable":
        fallback_answer, citation_ids = _offline_answer(state)
        citations = [
            Citation(
                document_id=chunk.document_id,
                title=chunk.title,
                chunk_id=chunk.chunk_id,
                section=chunk.section,
            )
            for chunk in chunks
            if chunk.chunk_id in citation_ids
        ]
        return {
            "final_answer": fallback_answer,
            "citations": citations,
            "extra": extra,
        }
    patched, judged, corrected = await self_correct_answer(
        llm=llm,
        checker=checker,
        query=state["user_query"],
        answer=answer,
        chunks=chunks,
        tool_results_text=tool_text,
        first_result=result,
    )
    extra = {"faithfulness": judged.model_dump(), "self_corrected": corrected}
    if not judged.supported:
        return {
            "final_answer": "现有证据不足，暂时无法给出可靠答复。请补充信息或转人工。",
            "citations": [],
            "refused": True,
            "extra": extra,
        }
    return {"final_answer": patched, "extra": extra}
```

先看提前返回条件：已经拒答、等待确认、缺订单号时不再检查。离线 provider 和 DIRECT 也跳过。SINGLE/MULTI 没有证据时，直接返回资料不足。

正常路径调用 checker。通过就只补充 `extra` 中的判断信息，保留原答案；没通过就增加一个指标。

若是 `checker_unavailable`，退到离线的白名单或摘录回答。否则执行一次自纠错，仍不受支持则替换成拒答。

来源：[self_correct_answer](../../app/agents/nodes/faithfulness.py)。

```python
async def self_correct_answer(
    *,
    llm: LLMProvider,
    checker: FaithfulnessChecker,
    query: str,
    answer: str,
    chunks: list[RetrievedChunk],
    tool_results_text: str,
    first_result: FaithfulnessResult,
) -> tuple[str, FaithfulnessResult, bool]:
    """LLM rewrite once, then extractive fallback. Returns (answer, judge, corrected)."""
    evidence = (
        "\n\n".join(f"[{c.chunk_id}] {c.title}\n{c.content}" for c in chunks[:6]) or "(无检索结果)"
    )
    try:
        response = await llm.generate(
            [
                {"role": "system", "content": SELF_CORRECT_SYSTEM},
                {
                    "role": "user",
                    "content": SELF_CORRECT_USER.format(
                        query=query,
                        answer=answer,
                        critique=first_result.rationale
                        or ",".join(first_result.hallucinated_spans),
                        evidence=evidence,
                        tool_results=tool_results_text or "(无)",
                    ),
                },
            ],
            temperature=0.0,
            max_tokens=500,
            response_format={"type": "json_object"},
        )
        rewritten = parse_model(response.content, AnswerPayload).answer
        judged = await checker.check(
            query=query,
            answer=rewritten,
            chunks=chunks,
            tool_results_text=tool_results_text,
        )
        if judged.supported:
            return rewritten, judged, True
    except (LLMProviderError, LLMOutputError):
        pass
    patched = _extractive_fallback(query, evidence, tool_results_text)
    judged = await checker.check(
        query=query,
        answer=patched,
        chunks=chunks,
        tool_results_text=tool_results_text,
    )
    return patched, judged, True
```

注意返回类型是三元组：`tuple[str, FaithfulnessResult, bool]`。因此调用方可以写 `patched, judged, corrected = await ...`，一次接收三个返回值。

`critique` 把上次不支持的理由送给模型，要求根据证据重写。重写后再判一次；仍不行或发生指定异常，就转原文摘录并再检查。这里的 `except ...: pass` 表示结束尝试，继续执行后面的兜底代码。

一个实现细节：重写时解析出了 AnswerPayload，但主要只取 `.answer`；节点最后只更新正文与 extra，没有按新正文重新生成 citation 列表。因此引用 ID 的存在约束与修订后正文的逐句对应，需要分别评估。

## 8. 记忆预算实际上怎么算

来源：[预算估算](../../app/memory/budget.py)。

```python
def estimate_tokens(text: str) -> int:
    """DeepSeek tokenizer is not public; CJK-heavy heuristic: ~1 token / 1.5 chars."""
    if not text:
        return 0
    return max(1, int(len(text) / 1.5))
```

```python
def decide_level(used: int, settings: Settings) -> CompressionLevel:
    ratio = utilization(used, settings.token_budget)
    if ratio >= settings.auto_compact_ratio:
        return CompressionLevel.AUTO_COMPACT
    if ratio >= settings.compress_hard_ratio:
        return CompressionLevel.HARD_COMPRESS
    if ratio >= settings.compress_soft_ratio:
        return CompressionLevel.SOFT_TRUNCATE
    return CompressionLevel.NONE
```

`len(text) / 1.5` 是按字符数粗估，不是调用模型的 tokenizer。默认预算 128000，50%/70%/85% 分别对应软截断、强压缩和自动摘要。

教学数值：预算 128000 时，64000/89600/108800 分别达到三个门槛。`>=` 包含恰好等于阈值的情况。代码先判断最高阈值，再判断较低阈值，避免高使用率提前落入低级别。

## 9. 压缩函数怎样处理工具和历史

来源：[apply_compression](../../app/memory/compression.py)。

```python
def apply_compression(
    messages: list[ChatMessage],
    tool_results: list[ToolResult],
    summary: str,
    settings: Settings,
) -> tuple[list[ChatMessage], list[ToolResult], CompressionLevel, str]:
    used = messages_tokens(messages) + estimate_tokens(summary)
    level = decide_level(used, settings)
    tools = compress_tool_results(tool_results, level)
    if level in {CompressionLevel.HARD_COMPRESS, CompressionLevel.AUTO_COMPACT}:
        window = sliding_window(messages)
        if summary:
            window = [ChatMessage(role="system", content=f"对话摘要：{summary}")] + window
        return window, tools, level, summary
    if level == CompressionLevel.SOFT_TRUNCATE:
        return messages, tools, level, summary
    return messages, tools, level, summary
```

`used` 只累计历史消息与摘要，不包含完整系统 Prompt、检索证据与工具输出。虽然模块里另有 `chunks_tokens`，此处没有调用它。

工具结果按级别截断 preview；高压缩级别将历史保留最近四轮，已有摘要则在窗口前增加一条 system 消息。函数返回四个值：消息列表、工具结果、级别、摘要。类型注解帮助你知道每个返回位置是什么。

`compress_tool_results` 即使在 NONE 级别也有 10000 字符的单个结果上限；SOFT 是 400，HARD/AUTO 是 160。它复制结果对象后改 output，不是直接修改工具原始返回对象。

来源：[memory_node](../../app/agents/nodes/memory.py)。

```python
async def memory_node(
    state: AgentState,
    *,
    settings: Settings,
    summarizer: ConversationSummarizer,
) -> dict:
    messages = list(state.get("messages") or [])
    messages.append(ChatMessage(role="user", content=state["user_query"]))
    final_answer = state.get("final_answer")
    if final_answer:
        messages.append(ChatMessage(role="assistant", content=final_answer))
    tool_results = list(state.get("tool_results") or [])
    summary = state.get("summary") or ""
    messages, tool_results, level, summary = apply_compression(
        messages, tool_results, summary, settings
    )
    if level == CompressionLevel.AUTO_COMPACT:
        summary = await summarizer.summarize(messages)
        messages, tool_results, _, summary = apply_compression(
            messages, tool_results, summary, settings
        )
    return {"messages": messages, "tool_results": tool_results, "summary": summary}
```

这段先拷贝历史列表，加入本轮用户问题和最终回答，再调用压缩。AUTO_COMPACT 时再生成摘要，之后再次压缩。

### 当前链路有三个重要接线问题

1. `ChatService._initial_state` 读取最近 20 条消息，但 summary 固定为 `""`，没有取回数据库已保存的摘要。
2. 生成节点构造的 `messages` 是新建的两条 Prompt，没有拼接 `state["messages"]` 或 `state["summary"]`。
3. 图中的 compact_memory 在生成和检查后面，本轮的模型请求已经发出，因此不能在该位置限制本轮生成 Prompt 的体积。

还有一个顺序问题：AUTO_COMPACT 先执行滑窗，再把截短后的消息交给 summarizer，早期消息可能已经丢掉。持久化保存了当前用户/助手原始消息，压缩后的 messages 也没有作为新历史窗口写回数据库。

因此，目前可以准确说“有历史持久化、预算判断、截断与摘要组件”；不能仅看模块名就认为完整长期记忆和上下文预算已经生效。

## 10. SSE：进度流与最终答案流分开

来源：[ChatService.stream](../../app/services/chat_service.py)。

```python
async def stream(self, request: ChatRequest, *, trace_id: str) -> AsyncIterator[dict]:
    """SSE: report progress, then send only the validated user-facing answer."""
    await self._enforce_rate_limit(request.user_id)
    started = time.perf_counter()
    session_id = request.session_id or new_id("sess_")
    async with request_session(self._session_factory) as db:
        state = await self._initial_state(db, request, session_id, trace_id)
        yield {"event": "started", "trace_id": trace_id}
        queue: asyncio.Queue[dict | None] = asyncio.Queue()
        holder: dict = {"merged": dict(state), "guardrail": None, "error": None}

        async def _run_graph() -> None:
            try:
                async for update in self._graph.astream(state, stream_mode="updates"):
                    for node, payload in update.items():
                        if isinstance(payload, dict):
                            holder["merged"].update(payload)
                        event: dict = {"event": "node", "node": node}
                        if isinstance(payload, dict) and payload.get("intent"):
                            event["intent"] = payload["intent"].label.value
                        if isinstance(payload, dict) and payload.get("strategy"):
                            event["strategy"] = payload["strategy"].value
                        await queue.put(event)
            except GuardrailError as exc:
                holder["guardrail"] = exc
            except Exception as exc:  # graph failures surface after the queue drains
                holder["error"] = exc
            finally:
                await queue.put(None)

        task = asyncio.create_task(_run_graph())
        try:
            while True:
                item = await queue.get()
                if item is None:
                    break
                yield item
            await task
        finally:
            if not task.done():
                task.cancel()

        if holder["guardrail"] is not None:
            exc = holder["guardrail"]
            await db.commit()
            yield {
                "event": "answer",
                "data": ChatResponse(
                    answer=exc.message,
                    session_id=session_id,
                    trace_id=trace_id,
                    refused=True,
                    error_code=exc.error_code.value,
                ).model_dump(mode="json"),
            }
            yield {"event": "done", "trace_id": trace_id}
            return
        if holder["error"] is not None:
            raise holder["error"]
        merged: AgentState = holder["merged"]
        response = self._to_response(merged, session_id=session_id, trace_id=trace_id)
        await self._persist(db, request, response, summary=merged.get("summary") or "")
        await db.commit()
    self._remember_pending(request, response)
    self._observe(response, started)
    # Never stream the LLM's JSON envelope or an answer that may still be
    # replaced by the faithfulness check. Only the final answer is public.
    for index in range(0, len(response.answer), 48):
        yield {"event": "token", "text": response.answer[index : index + 48]}
    yield {"event": "answer", "data": response.model_dump(mode="json")}
    yield {"event": "done", "trace_id": trace_id}
```

这是异步生成器：普通函数通过 return 结束并交出一个结果，生成器可用 yield 逐次交出多个事件。调用方用 `async for` 接收。

数据流可以按三个阶段理解：

1. 发出 started，然后创建后台任务执行图。图每完成一个节点，就把 node 事件放进队列。
2. 外层等待 queue.get，把事件 yield 给 API；None 是本地约定的结束标记。不是向客户端发送一个空答案。
3. 图完成、答案校验通过、数据提交后，再把最终正文每 48 个字符分段，发送 token、完整 answer、done。

`asyncio.create_task` 让图执行与队列消费可以交错运行。`finally` 在消费结束或取消时尝试停止尚未完成的图任务。`holder["merged"].update(payload)` 则将本图节点的状态更新合并到最终结果。

事件名 token 不表示它是模型 tokenizer 的一个 token。当前切片单位是 Python 字符串字符；用户真正看到答案正文之前，生成与忠实度检查已完成。

这与 provider 中存在一个 `stream` 方法是不同层次：聊天服务这条路径使用图的状态更新流，不直接把模型原始增量文本透传给用户。

## 11. 练习与对照答案

1. 为什么空 tools 不能只用 all 判断失败？因为 `all([])` 为 True，需要先判断确实有结果。
2. 哪个变量把检索与生成连接起来？先是 chunks 转成 evidence，再填入 ANSWER_USER 并放入 messages。
3. valid citation 能否证明正文准确？不能；ID 存在校验与语义支持判断是不同检查。
4. 为什么设置了 128k 预算，仍不能声称本轮 Prompt 被严格限制？估算不完整且压缩节点在生成后，历史也未注入生成。
5. SSE 的 token 事件在何时发送？完整工作流校验并提交后，按 48 字符分段发送。

[回到阅读路线](README.md)
