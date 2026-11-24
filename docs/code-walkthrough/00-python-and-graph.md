# Python、共享状态与 LangGraph：先看懂代码怎样连接

[返回阅读路线](README.md) · [下一篇：启动与对话](01-startup-and-chat.md)

本文先解释反复出现的语法，再展示真正的工作流与路由源码。除明确标注“教学示意”的例子外，源码块直接摘自对应文件。代码需要放回原模块、连同导入和依赖使用，不能把单个代码块当作独立程序运行。

## 1. 函数签名先读哪几部分

下面摘自 `ChatService` 的实际方法签名：

```python
async def chat(self, request: ChatRequest, *, trace_id: str) -> ChatResponse:
```

从左到右读：

- `async def`：定义异步函数；等待网络、数据库等异步操作时，可以把执行机会让给其他任务。
- `chat`：方法名。
- `self`：当前服务对象；Python 调用对象方法时会自动传入。
- `request`：本次传入的参数，期望是 `ChatRequest` 对象。
- `*`：后面的参数必须写名字，例如 `trace_id="trace_demo"`，不能只传一个匿名位置参数。
- `-> ChatResponse`：期望返回的类型。

类型提示本身不是运行时强制检查。项目的 `BaseModel` 校验、显式 `if` 判断才是具体约束。

教学示意：

```python
# service 已经是一个 ChatService 对象
response = await service.chat(request, trace_id="trace_demo")
```

`await` 得到异步函数最终返回的值。它不会自动把多个调用变成并行。`await A()` 后再 `await B()`，依然是先等 A，再执行 B。

## 2. class、实例和 self

源码中 `ChatService.__init__` 保存了 `self._llm = llm`。这个赋值把传进来的模型客户端存到当前对象上，后面的方法可以继续访问它。

教学示意：

```python
class Greeter:
    def __init__(self, name):
        self.name = name

    def greet(self):
        return "你好，" + self.name

greeter = Greeter("小王")
text = greeter.greet()  # "你好，小王"
```

`Greeter` 是类型，`greeter` 是具体实例。`__init__` 在创建实例时初始化属性。`self.name` 是对象的属性，函数里的普通变量只在该次调用中使用。

`_llm` 前的单下划线表达“内部使用”的约定，Python 不会因此阻止外部访问。

## 3. 字典、对象、列表如何取值

本项目混合使用三种结构：

| 结构 | 访问方式 | 示例 |
|---|---|---|
| 普通字典 | `["字段名"]` 或 `.get("字段名")` | `state["user_query"]` |
| Pydantic 模型对象 | `.属性名` | `request.message`、`chunk.content` |
| 列表 | `[下标]` 或切片 | `chunks[0]`、`chunks[:6]` |

教学示意：

```python
state = {"user_query": "退货政策是什么？"}
query = state["user_query"]
missing = state.get("final_answer")  # 没这个字段，返回 None
# state["final_answer"]             # 没这个字段，会抛出 KeyError
```

`state.get("retrieved_chunks") or []` 表示：如果取出的值是假值，例如 `None` 或空列表，则使用空列表。`or` 会返回操作数本身，不只是返回 True/False。

`list[RetrievedChunk]` 表示“元素应是资料片段对象的列表”。`IntentResult | None` 表示“意图对象或空值”。

## 4. state 的完整字段

来源：[AgentState](../../app/agents/state.py)。

```python
class AgentState(TypedDict, total=False):
    """LangGraph state. Partial updates replace listed keys; no infinite append reducer.

    Messages are compressed explicitly in the memory node instead of Annotated[add]
    so the graph cannot silently blow the 128k budget.
    """

    trace_id: str
    session_id: str
    user_id: str
    user_query: str
    messages: list[ChatMessage]
    summary: str
    intent: IntentResult | None
    complexity: ComplexityResult | None
    strategy: RetrievalStrategy | None
    query_complexity: QueryComplexity | None
    rewritten_queries: list[str]
    retrieved_chunks: list[RetrievedChunk]
    plan: AgentPlan | None
    tool_results: list[ToolResult]
    pending_action: ActionProposal | None
    confirmed_action: ActionProposal | None
    confirm_action_id: str | None
    idempotency_key: str | None
    final_answer: str | None
    citations: list[Citation]
    refused: bool
    error: str | None
    step_count: int
    token_usage: TokenUsage
    latency: LatencyBreakdown
    extra: dict[str, Any]
```

这是 `TypedDict`，实际运行时仍是字典。`total=False` 允许字段尚未存在，不表示每个字段已经被赋值为 None。与之相比，`ChatRequest(BaseModel)` 会在构建对象时执行数据校验。

可以把字段分成四组来读：

| 数据 | 字段 |
|---|---|
| 请求身份与输入 | trace_id、session_id、user_id、user_query |
| 中间决策 | intent、complexity、strategy、rewritten_queries、plan |
| 证据与结果 | retrieved_chunks、tool_results、final_answer、citations |
| 控制和统计 | pending_action、confirmed_action、refused、token_usage、latency |

`messages` 和 `summary` 是记忆相关数据。它们是否进入生成模型，需要继续追踪 04 中的 Prompt 构造。

## 5. 一个节点完整做了什么

来源：[intent_node](../../app/agents/nodes/router.py)。

```python
async def intent_node(state: AgentState, classifier: IntentClassifier) -> dict:
    started = time.perf_counter()
    intent = await classifier.classify(state["user_query"])
    latency = _latency(state)
    latency.intent_ms = (time.perf_counter() - started) * 1000
    return {"intent": intent, "latency": latency}
```

按执行顺序解释：

1. `time.perf_counter()` 记录计时起点。
2. 从 `state["user_query"]` 读取当前用户问题。
3. 调用分类器，等待结果。结果是 `IntentResult` 对象。
4. 取出已有延迟对象或创建一个，将本段耗时换算为毫秒。
5. 返回字典，仅列出要更新的字段。

项目图中没有为这些字段配置累加 reducer，节点一般更新同名字段、保留未返回字段。因此返回 `{"intent": intent}` 不会清空 `user_query`。

教学示意，帮助理解合并结果；不是 LangGraph 内部实现：

```python
state = {"user_query": "退货政策是什么？", "retrieved_chunks": []}
update = {"intent": "这里代表一个 IntentResult 对象"}
state.update(update)
# 原来的 user_query 和 retrieved_chunks 仍然存在
```

## 6. 意图分类为什么既有规则也有模型

来源：[IntentClassifier.classify](../../app/agents/nodes/intent.py)。

```python
async def classify(self, query: str) -> IntentResult:
    fallback = rule_based_intent(query)
    if isinstance(self._llm, OfflineLLMProvider):
        return self._maybe_overlay(query, fallback)
    try:
        response = await self._llm.generate(
            [
                {"role": "system", "content": self._system_prompt},
                {
                    "role": "user",
                    "content": INTENT_USER.format(
                        query=query, prompt_version=self._prompt_version
                    ),
                },
            ],
            temperature=0.0,
            max_tokens=256,
            response_format={"type": "json_object"},
        )
        parsed = parse_model(response.content, IntentResult)
        if not parsed.prompt_version:
            parsed.prompt_version = self._prompt_version
        return self._maybe_overlay(query, parsed)
    except (LLMProviderError, LLMOutputError):
        return self._maybe_overlay(query, fallback)
```

`fallback` 先按关键词等规则计算一个可用结果。离线 provider 不调用远程模型，直接使用规则与覆盖层。在线时，模型收到 system/user 两条消息，返回结构化 JSON，再由 `parse_model` 校验为 `IntentResult`。

`isinstance(x, SomeClass)` 判断对象是不是某种类型。`try/except` 捕获指定类型的异常；模型请求或输出解析失败时走兜底。成功解析后仍会调用 `_maybe_overlay`，按项目的高混淆短语规则纠偏。因此意图结果可能来自“模型 + 规则覆盖”，不能一概称为纯模型分类。

`temperature=0.0` 表示尽量降低采样随机性，不保证远程模型多次调用完全相同。

意图回答的是“用户在谈退货还是物流”。接下来复杂度路由回答“应当查政策，还是查个人订单”。

## 7. 复杂度节点怎样写入策略

来源：[router_node](../../app/agents/nodes/router.py)。

```python
async def router_node(state: AgentState, router: AdaptiveRouter) -> dict:
    started = time.perf_counter()
    intent = state["intent"]
    assert intent is not None
    complexity = (
        ComplexityResult(
            complexity=QueryComplexity.TOOL_REQUIRED,
            strategy=RetrievalStrategy.AGENT,
            confidence=1.0,
            rationale="confirmed_action",
        )
        if state.get("confirmed_action")
        else await router.route(state["user_query"], intent)
    )
    latency = _latency(state)
    latency.routing_ms = (time.perf_counter() - started) * 1000
    return {
        "complexity": complexity,
        "strategy": complexity.strategy,
        "query_complexity": complexity.complexity,
        "latency": latency,
    }
```

这里有一个跨行的条件表达式：`A if 条件 else B`。已经有经过服务端验证的确认动作时，直接构造 `AGENT` 结果，否则让 AdaptiveRouter 判断。

`assert intent is not None` 表示作者预期前面的节点已经填好意图。它不适合代替外部输入校验，因为优化运行模式可以关闭 assert。

返回值里的 `strategy` 是枚举对象，`query_complexity` 是复杂度枚举，`complexity` 保存完整判断结果和理由。

来源：[AdaptiveRouter.route](../../app/retrieval/adaptive.py)。

```python
async def route(self, query: str, intent: IntentResult) -> ComplexityResult:
    fallback = rule_based_complexity(query, intent)
    if isinstance(self._llm, OfflineLLMProvider):
        return fallback
    try:
        response = await self._llm.generate(
            [
                {"role": "system", "content": COMPLEXITY_SYSTEM},
                {
                    "role": "user",
                    "content": COMPLEXITY_USER.format(
                        query=query,
                        intent=intent.label.value,
                        prompt_version=PROMPT_VERSION_COMPLEXITY,
                    ),
                },
            ],
            temperature=0.0,
            max_tokens=256,
            response_format={"type": "json_object"},
        )
        parsed = parse_model(response.content, ComplexityResult)
        if (
            fallback.rationale == "general_policy_without_order"
            and parsed.strategy == RetrievalStrategy.AGENT
        ):
            return fallback
        if _ORDER_ID.search(query) and fallback.strategy == RetrievalStrategy.AGENT:
            return fallback
        return parsed
    except (LLMProviderError, LLMOutputError):
        return fallback
```

仍然是“先算规则兜底 → 尝试模型 → 解析 → 必要时规则保护”的模式。两个保护条件特别值得看：一般政策问题不应被误派去执行业务工具；明确出现订单号时保留工具路径。

规则函数的完整逻辑见 [rule_based_complexity](../../app/retrieval/adaptive.py)。它按顺序检查越界、社交问候、一般售后政策、账户或订单数据、多条件提示，最后默认单轮检索。`if` 的先后顺序会影响结果，不能认为所有满足条件的策略都会执行。

## 8. 策略如何变成下一站

来源：[策略到节点的映射](../../app/agents/graph.py)。

```python
def _route_after_complexity(state: AgentState) -> str:
    strategy = state.get("strategy") or RetrievalStrategy.SINGLE
    mapping = {
        RetrievalStrategy.DIRECT: "generate_answer",
        RetrievalStrategy.REFUSE: "refuse_answer",
        RetrievalStrategy.SINGLE: "retrieve_docs",
        RetrievalStrategy.MULTI: "retrieve_docs",
        RetrievalStrategy.AGENT: "plan_tasks",
    }
    return mapping[strategy]
```

`mapping[strategy]` 是字典查找，结果是一个节点名称字符串。

| 策略 | 返回的节点名 | 下一步 |
|---|---|---|
| DIRECT | generate_answer | 直接生成 |
| SINGLE | retrieve_docs | 一轮检索流程 |
| MULTI | retrieve_docs | 同一个检索节点内部进入多轮路径 |
| AGENT | plan_tasks | 生成业务计划 |
| REFUSE | refuse_answer | 拒答 |

`SINGLE` 不代表只取一个片段，它表示不走多跳 Self-RAG 循环。该路径仍可查询原问题与一个不同改写。

## 9. 完整的工作流构建函数

来源：[build_graph](../../app/agents/graph.py)。

```python
def build_graph(
    *,
    llm: LLMProvider,
    settings: Settings,
    retriever: HybridRetriever,
    reranker: DocumentReranker,
    registry: ToolRegistry,
) -> CompiledStateGraph:
    classifier = IntentClassifier(llm)
    adaptive = AdaptiveRouter(llm)
    rewriter = QueryRewriter(llm)
    self_rag = SelfRAGLoop(retriever, reranker, rewriter, llm, settings)
    planner = Planner(llm, registry)
    checker = FaithfulnessChecker(llm)
    summarizer = ConversationSummarizer(llm)

    graph = StateGraph(AgentState)
    graph.add_node("apply_guardrail", guardrail_node)
    graph.add_node("classify_intent", partial(intent_node, classifier=classifier))
    graph.add_node("route_complexity", partial(router_node, router=adaptive))
    graph.add_node(
        "retrieve_docs",
        partial(
            retrieval_node,
            retriever=retriever,
            reranker=reranker,
            rewriter=rewriter,
            self_rag=self_rag,
            settings=settings,
        ),
    )
    graph.add_node("plan_tasks", partial(planner_node, planner=planner))
    graph.add_node("execute_tools", partial(executor_node, registry=registry, settings=settings))
    graph.add_node("generate_answer", partial(generate_node, llm=llm))
    graph.add_node("refuse_answer", refuse_node)
    graph.add_node("check_faithfulness", partial(faithfulness_node, checker=checker, llm=llm))
    graph.add_node("compact_memory", partial(memory_node, settings=settings, summarizer=summarizer))

    def _after_guardrail(state: AgentState) -> str:
        return "refuse_answer" if state.get("refused") else "classify_intent"

    graph.add_edge(START, "apply_guardrail")
    graph.add_conditional_edges(
        "apply_guardrail",
        _after_guardrail,
        {"refuse_answer": "refuse_answer", "classify_intent": "classify_intent"},
    )
    graph.add_edge("classify_intent", "route_complexity")
    graph.add_conditional_edges(
        "route_complexity",
        _route_after_complexity,
        {
            "generate_answer": "generate_answer",
            "refuse_answer": "refuse_answer",
            "retrieve_docs": "retrieve_docs",
            "plan_tasks": "plan_tasks",
        },
    )
    graph.add_edge("retrieve_docs", "generate_answer")
    graph.add_edge("plan_tasks", "execute_tools")
    graph.add_edge("execute_tools", "generate_answer")
    graph.add_edge("generate_answer", "check_faithfulness")
    graph.add_edge("refuse_answer", "compact_memory")
    graph.add_edge("check_faithfulness", "compact_memory")
    graph.add_edge("compact_memory", END)
    return graph.compile()
```

这段在服务构造时运行，不是每次提问都重新构建。

先创建分类器、改写器、规划器等对象；随后注册节点，最后连接边。

`partial(intent_node, classifier=classifier)` 是提前固定参数。因为图通常给节点传入 state，额外的分类器、模型、检索器由 `partial` 预先绑定。可以把它理解为一个“已经配好依赖的函数”。

`add_edge(A, B)` 表示 A 完成后进入 B。`add_conditional_edges` 表示先调用一个路由函数，根据它返回的字符串选分支。

`graph.compile()` 返回可执行的图对象，没有在这里实际处理用户问题。真正处理问题发生在 `await self._graph.ainvoke(state)`，流式路径则使用 `astream`。

这里还要区分两种循环：图本身是受控的分支流程，Self-RAG 的多轮检索循环位于检索对象内部，不能只看图连线就判断它有几跳。

## 10. 对照例子检查理解

以下是教学示意，假定路由选择了 SINGLE：

```text
初始 state：user_query="退货政策是什么？"
classify_intent：写入 IntentResult
route_complexity：写入 strategy=SINGLE
_route_after_complexity：返回 "retrieve_docs"
retrieve_docs：写入 retrieved_chunks
generate_answer：写入 final_answer、citations
check_faithfulness：保留、修正或拒绝答案
compact_memory：整理 messages / summary
返回图的最终 state
```

练习：在 `build_graph` 里找到 `execute_tools` 的下一站；再找到 `refuse_answer` 的下一站。前者进入 `generate_answer`，后者直接进入 `compact_memory`，不必把拒答交给模型重新生成。

[继续阅读：启动与对话主流程](01-startup-and-chat.md)
