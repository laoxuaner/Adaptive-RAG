# 01 · 从启动到一次聊天：先读懂代码怎样连接起来

本文先回答一个具体问题：用户发来 **“退货政策是什么？”**，这句话怎样从 HTTP 请求变成图的输入，再怎样变成返回结果和数据库记录？

本文基于当前仓库源码静态解读，没有启动服务、调用模型或执行数据库操作。标为“源码”的代码摘自仓库；标为“教学示意”的内容只是帮助理解，里面的会话 ID、分类、答案等都不是实测结果。函数的调用关系是确定的，模型会给出什么分类、采用哪条策略和生成什么答案，需要实际运行才能确定。

源码入口：[app/asgi.py](../../app/asgi.py)、[app/main.py](../../app/main.py)、[容器](../../app/services/container.py)、[聊天服务](../../app/services/chat_service.py)。

## 1. 先看两条时间线

项目里有两种容易混在一起的工作。

| 时间 | 执行什么 | 执行频率 |
|---|---|---|
| 应用创建 | 创建 FastAPI 对象，注册路由、中间件、异常处理 | 通常每个服务进程一次 |
| 应用启动 | 读取配置，创建模型客户端、数据库、检索器、ChatService 和图 | 通常每个服务进程一次 |
| 每个聊天请求 | 校验输入、限流、读历史、执行图、保存消息、返回响应 | 每次请求一次 |
| 应用关闭 | 关闭模型客户端、Redis 连接，释放数据库引擎 | 进程正常退出时 |

“教学示意”：本文关注的 JSON 请求路径如下。

```text
应用启动
  app/asgi.py: app = create_app()
  → FastAPI 进入 lifespan
  → build_container(settings)
  → ChatService(...) → build_graph(...)
  → 把容器放在 app.state.container

一次请求
  POST /api/v1/chat
  → FastAPI 把 JSON 校验为 ChatRequest
  → 路由取得 AppContainer
  → ChatService.chat(...)
      → 限流
      → 建立请求范围的数据库 session
      → _initial_state：整理图的输入
      → _invoke：执行 LangGraph
      → _to_response：整理对外响应
      → _persist：把消息加入数据库 session
      → commit：提交事务
      → 记录待确认动作及指标
  → FastAPI 返回 JSON
```

这里的 **session 有两个含义**：`session_id` 标识“一个用户会话”；`AsyncSession` 是“一次数据库交互使用的 SQLAlchemy 对象”。它们不是同一件东西。

## 2. Python 入口只有几行，真正的工作在生命周期里

### 2.1 ASGI 入口

源码：[app/asgi.py](../../app/asgi.py)。

```python
from app.main import create_app

app = create_app()
```

按执行顺序看：

1. `from ... import ...` 从另一个模块取得 `create_app` 函数。
2. `create_app()` 调用函数，获得一个 FastAPI 应用对象。
3. `app = ...` 把这个对象绑定到模块变量 `app`。Uvicorn 的 `app.asgi:app` 就是“导入 `app.asgi` 模块，再取它的 `app` 变量”。

此时不要把 `app = create_app()` 理解为“已经处理了一次聊天”。它只是组装 HTTP 应用，后面还要进入启动生命周期。

### 2.2 应用工厂注册生命周期和路由

下面是 [create_app](../../app/main.py) 的**源码节选**，省略了长注释、中间件、异常处理和静态文件挂载部分；完整函数见链接。

```python
def create_app(settings: Settings | None = None) -> FastAPI:
    if settings is not None:
        from app.config.settings import clear_settings_cache

        clear_settings_cache()
    resolved = settings or get_settings()
    app = FastAPI(
        title="Adaptive-RAG Customer Service",
        version="0.1.0",
        lifespan=lifespan,
    )
    prefix = resolved.api_v1_prefix
    app.include_router(health.router, prefix=prefix)
    app.include_router(chat.router, prefix=prefix)
    app.include_router(admin.router, prefix=prefix)
```

`Settings | None` 表示“配置对象或 `None`”；`= None` 是默认值；`-> FastAPI` 是返回类型提示。类型提示帮助 IDE 和 mypy 理解代码，本身不是执行步骤。

`settings or get_settings()` 表示：有传入的配置就使用它，否则读取配置。这里 `resolved` 主要用于应用注册配置；下文的 `lifespan` 本身仍调用 `get_settings()` 构建运行时容器，不是直接接收 `resolved`。

`lifespan=lifespan` 把启动/关闭函数交给 FastAPI。`include_router` 把路由模块注册进应用；默认 `api_v1_prefix` 是 `/api/v1`，所以模块内的 `/chat` 最后成为 `/api/v1/chat`。

### 2.3 lifespan：在 yield 前启动，在 finally 中关闭

源码：[app/main.py](../../app/main.py)，完整函数。

```python
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    """Build the container on startup and release it on shutdown."""
    settings = get_settings()
    configure_logging(settings)
    container = await build_container(settings)
    app.state.container = container
    try:
        yield
    finally:
        await container.aclose()
```

按执行顺序理解：

1. `get_settings()` 从环境与 `.env` 等位置读取并缓存配置；它不会调用大模型。
2. `configure_logging(settings)` 设置日志。
3. `await build_container(settings)` 创建运行时组件，等它完成后才能进入下一行。
4. `app.state.container` 是应用级别的存放位置，后续请求从这里取得同一进程中的容器。
5. `yield` 把控制权交回 FastAPI。服务在此期间接受请求。
6. 应用关闭时继续执行，进入 `finally` 释放资源。`finally` 的含义是退出这段上下文时执行清理，即使中途遇到异常也会尝试执行。

`@asynccontextmanager` 是装饰器：把“`yield` 前准备、`yield` 后清理”的异步生成器包装成异步上下文管理器。后面的 `async with request_session(...)` 也使用同一种结构。

**async / await 先记住三点：**

- `async def` 定义异步函数。调用它会得到一个协程对象，通常由 `await` 或事件循环驱动执行。
- `await` 用于等待可等待对象完成，拿到其结果；等待期间，事件循环可以去处理其他就绪任务。
- `async` 不等于“自动并行”或“自动更快”。例如这里先等容器建立，再开放服务；一次 `chat()` 里的主要步骤也是按书写顺序等待。

## 3. 容器是什么：把零件组装成可用对象

“容器”在这里是一个普通 Python 对象，不是 Docker 容器。它保存各个服务和资源的引用，让上层代码不用每次请求都重新组装模型、数据库、检索器。

### 3.1 先创建基础设施

源码：[build_infrastructure](../../app/services/container.py)，完整函数。

```python
async def build_infrastructure(settings: Settings):
    """Create the database session factory and optional Redis connection."""
    engine = create_engine(settings)
    await init_schema(engine)
    session_factory = create_session_factory(engine)
    redis = await connect_redis(settings)
    return engine, session_factory, redis
```

1. `engine` 是数据库引擎，负责连接与连接池等底层工作。
2. `init_schema` 使用 SQLAlchemy 元数据创建缺少的表，当前实现是 `Base.metadata.create_all`，不是逐版本迁移命令。
3. `session_factory` 是“生产数据库 session 的工厂”，不是一条已经执行的查询。
4. Redis 是可选组件，连接结果可能是 `None`。
5. `return a, b, c` 返回一个包含三个元素的元组；调用方可以用 `engine, session_factory, redis = ...` 分别接收。

### 3.2 再创建检索器

源码：[build_retrieval](../../app/services/container.py)，完整函数。

```python
async def build_retrieval(settings: Settings, redis):
    """Build the knowledge ingestion and hybrid retrieval components."""
    embedder = build_embedding_provider(settings)
    cache = (
        RedisRetrievalCache(redis, ttl_seconds=settings.retrieval_cache_ttl_seconds)
        if redis is not None
        else MemoryRetrievalCache(ttl_seconds=settings.retrieval_cache_ttl_seconds)
    )
    bm25, store, _ = await ingest_knowledge(
        settings.knowledge_dir,
        embedder,
        settings=settings,
        qdrant_url=settings.qdrant_url,
        qdrant_collection=settings.qdrant_collection,
        embedding_dim=getattr(embedder, "dim", settings.embedding_dim),
    )
    retriever = HybridRetriever(
        bm25, VectorRetriever(store, embedder), rrf_k=settings.rrf_k, cache=cache
    )
    reranker = DocumentReranker(CrossEncoderRerankProvider(settings))
    return retriever, reranker, store
```

这一段还没有收到“退货政策是什么？”，它准备的是所有请求共享的检索组件：

1. 构建 embedding provider，把文本映射为向量的组件。
2. 有 Redis 就用 Redis 检索缓存，否则用进程内缓存。`A if condition else B` 是条件表达式。
3. `ingest_knowledge` 读取知识库并建立检索所需的数据。它返回多个值，`_` 表示调用方有意不使用其中一个返回值。
4. `HybridRetriever` 组合 BM25 与 `VectorRetriever`；后者依赖向量存储 `store` 和 embedding provider。
5. 构建 reranker。`CrossEncoderRerankProvider(settings)` 是否实际可加载模型，要看配置和运行环境，单凭这一行不能断言当前一定使用真实 CE 权重。

`getattr(embedder, "dim", settings.embedding_dim)` 的意思是“优先读取 `embedder.dim`，没有该属性时使用配置中的维度”。

### 3.3 build_container 把组件注入 ChatService

源码：[build_container](../../app/services/container.py)，下面保留主要组装逻辑，**省略了末尾 `logger.info(...)` 日志块**。

```python
async def build_container(settings: Settings, seed: bool = False) -> AppContainer:
    """Compose runtime services from independently testable infrastructure builders."""
    llm: LLMProvider = (
        DeepSeekProvider(settings) if settings.llm_configured else OfflineLLMProvider()
    )
    engine, session_factory, redis = await build_infrastructure(settings)
    rate_limiter = (
        RedisRateLimiter(redis, limit=settings.rate_limit_per_minute)
        if redis is not None
        else InMemoryRateLimiter(limit=settings.rate_limit_per_minute)
    )
    retriever, reranker, store = await build_retrieval(settings, redis)
    registry = build_tool_registry(retriever)
    if settings.mcp_enabled:
        specs = list(LOCAL_CATALOG)
        if settings.mcp_remote_url:
            specs.append(
                {
                    "name": "mcp_remote_echo",
                    "description": "Remote MCP tool via HTTP (same Function Calling schema)",
                    "transport": "http",
                    "endpoint": settings.mcp_remote_url.rstrip("/") + "/mcp/call",
                }
            )
        merge_mcp_tools(registry, specs)
    chat_service = ChatService(
        settings=settings,
        llm=llm,
        retriever=retriever,
        reranker=reranker,
        session_factory=session_factory,
        rate_limiter=rate_limiter,
        registry=registry,
    )
    if seed:
        async with session_factory() as session:
            await seed_demo_data(session)
            await session.commit()
    return AppContainer(
        settings=settings,
        llm=llm,
        chat_service=chat_service,
        engine=engine,
        session_factory=session_factory,
        retriever=retriever,
        redis=redis,
        vector_store=store,
    )
```

按执行顺序理解：

1. 有配置好的 LLM key 时构建 DeepSeek provider，否则构建离线 provider。离线 provider 是项目自己的实现，不代表离线运行了 DeepSeek。
2. 创建数据库等基础设施，选择限流实现。
3. 构建检索与重排组件，注册可用工具；开启 MCP 时再合并工具规格。
4. `ChatService(...)` 把这些已经构造好的组件传入聊天服务。这就是本项目一种很直接的**依赖注入**：聊天服务需要什么，由组装层提供什么。
5. `seed=False` 是默认值。上面的 `lifespan` 没传 `seed=True`，因此正常启动这个路径不会因为执行到这里就自动插入演示订单。
6. 最后把所有引用包装到 `AppContainer` 中返回。

依赖注入的好处可以用测试理解：测试可以把 `llm` 换成一个不会联网的假对象，ChatService 的主流程仍然可以复用。它不是某种必须安装的魔法框架。

容器关闭时运行的源码如下，同在 [container.py](../../app/services/container.py)：

```python
    async def aclose(self) -> None:
        closer = getattr(self.llm, "aclose", None)
        if closer:
            await closer()
        if self.redis is not None:
            closer = getattr(self.redis, "aclose", None)
            if closer:
                await closer()
        await self.engine.dispose()
```

它依次尝试关闭模型客户端、Redis，再释放数据库引擎。`getattr(..., None)` 允许不同 provider 不必都提供同名关闭方法。

## 4. HTTP 输入先经过 Pydantic，才进入业务函数

### 4.1 ChatRequest 是入口的数据契约

源码：[app/schemas/chat.py](../../app/schemas/chat.py)，完整类。

```python
class ChatRequest(BaseModel):
    message: str = Field(min_length=1, max_length=4000)
    session_id: str | None = Field(default=None, max_length=64)
    user_id: str = Field(default="anonymous", min_length=1, max_length=64)
    stream: bool = False
    confirm_action_id: str | None = None
    idempotency_key: str | None = Field(default=None, max_length=128)
```

它继承 Pydantic 的 `BaseModel`，因此能在运行时校验输入；`Field(...)` 声明字段约束与默认值。

“教学示意”：客户端可以发送以下 JSON。

```json
{
  "message": "退货政策是什么？",
  "user_id": "u_demo"
}
```

FastAPI 解析后，业务函数看到的是 `ChatRequest` 对象，可以读取 `request.message`；没有传的 `session_id` 是 `None`，`stream` 是 `False`。本例不涉及取消订单，所以没有待确认动作 ID。

空字符串不符合 `min_length=1`，超过 4000 字符不符合上限，会在进入路由业务逻辑前失败。需要注意，`min_length=1` 不等于“已经去除全部空白并判定非空”。

`user_id` 在这里是客户端传入的普通字段，模型校验“它是合法字段”不等于认证“它确实属于当前登录者”。

### 4.2 路由层只转交任务

源码：[app/api/routes/chat.py](../../app/api/routes/chat.py)，JSON 路由完整函数。

```python
router = APIRouter(tags=["chat"])


@router.post("/chat", response_model=ChatResponse)
async def chat(
    payload: ChatRequest,
    request: Request,
    container: AppContainer = Depends(get_container),
) -> ChatResponse:
    """Answer one turn and return the full JSON response."""
    return await container.chat_service.chat(payload, trace_id=request.state.trace_id)
```

这一段里的三个参数来自不同地方：

| 参数 | 来源 | 用途 |
|---|---|---|
| `payload` | HTTP JSON 请求体，经 Pydantic 校验 | 业务输入，如用户的问题 |
| `request` | FastAPI / Starlette 的请求对象 | 取得请求范围的 trace ID 等 |
| `container` | `Depends(get_container)` | 取得应用启动时建立的服务 |

`@router.post(...)` 是装饰器，把这个函数注册为 POST 处理器；`response_model=ChatResponse` 指定输出的数据契约。

`Depends(get_container)` 是 FastAPI 的依赖注入机制：请求进来后，框架先调用依赖函数，再把结果提供给路由。它不要求路由每次自己创建 ChatService。

依赖函数源码：[app/api/dependencies.py](../../app/api/dependencies.py)。

```python
def get_container(request: Request) -> AppContainer:
    return request.app.state.container
```

回头对应 `lifespan` 的 `app.state.container = container`，可以看到：启动时保存，处理请求时取出，前后是同一个存放位置。

`request.state.trace_id` 则由 [main.py 中的请求中间件](../../app/main.py) 在进入路由之前设置；它来自请求头或新生成的 ID，用于关联同一次请求的日志与返回结果。

### 4.3 这里有两条 API，不是一个 stream 开关

同文件的流式路由源码如下。

```python
@router.post("/chat/stream")
async def chat_stream(
    payload: ChatRequest,
    request: Request,
    container: AppContainer = Depends(get_container),
) -> EventSourceResponse:
    """Answer one turn as an SSE stream of node, token and answer events."""

    async def events():
        async for item in container.chat_service.stream(payload, trace_id=request.state.trace_id):
            yield JSONServerSentEvent(item)

    return EventSourceResponse(events())
```

`async for` 逐个读取异步迭代器产生的事件，`yield` 逐个交出事件；普通 `return` 则一次返回结果。前端使用 `/chat/stream`，而本文先通过 `/chat` 理解共同的业务步骤。

虽然 `ChatRequest` 里定义了 `stream` 字段，这两个路由没有根据它切换行为：选择哪个 HTTP 路径才决定调用 `chat()` 还是 `stream()`。

流式服务先运行图、验证答案并持久化，然后才把最终答案按最多 48 字符分段输出。因此这里的 `token` 事件不是模型原始 token 到达时立即透传；具体实现可继续查看 [ChatService.stream](../../app/services/chat_service.py)。

## 5. ChatService.__init__：self 是当前服务对象

源码：[app/services/chat_service.py](../../app/services/chat_service.py)，完整构造函数。

```python
class ChatService:
    def __init__(
        self,
        *,
        settings: Settings,
        llm: LLMProvider,
        retriever: HybridRetriever,
        reranker: DocumentReranker,
        session_factory: async_sessionmaker[AsyncSession],
        rate_limiter: RateLimiter,
        registry: ToolRegistry | None = None,
    ) -> None:
        self._settings = settings
        self._llm = llm
        self._retriever = retriever
        self._reranker = reranker
        self._session_factory = session_factory
        self._rate_limiter = rate_limiter
        self.registry = registry or build_tool_registry(retriever)
        self._pending_actions: dict[str, PendingConfirmation] = {}
        self._graph = build_graph(
            llm=llm,
            settings=settings,
            retriever=retriever,
            reranker=reranker,
            registry=self.registry,
        )
```

先理解常见 Python 语法：

- `class ChatService` 定义一种对象类型；调用 `ChatService(...)` 会创建实例，并调用 `__init__` 做初始化。
- `self` 是“当前这个实例”。调用 `service.chat(...)` 时，Python 会把 `service` 自动作为方法的 `self` 传进去。
- `self._llm = llm` 把传入组件保存在实例属性中，后续方法可以通过 `self._llm` 使用它。
- `_llm` 前面的单下划线是“内部实现”的命名约定，不是 Python 强制的私有权限。
- 参数中的裸 `*` 表示后面的参数必须写名字，所以调用时是 `ChatService(llm=..., settings=...)`。
- `dict[str, PendingConfirmation]` 表示键为字符串、值为待确认记录的字典；`{}` 创建空字典。

按初始化顺序看：先保存依赖，再得到工具 registry，再创建待确认动作字典，最后构建 LangGraph。

**关键点：图在初始化时构建，请求到来时调用这个图。** 创建图不等于已经对某个问题完成检索和生成；每次请求仍会传入新的 `AgentState`。共享的是组件与图结构，不应该把一个用户的请求数据当成服务实例的公共字段随意保存。

## 6. chat：把一轮对话的顺序钉住

源码：[ChatService.chat](../../app/services/chat_service.py)，完整函数。

```python
    async def chat(self, request: ChatRequest, *, trace_id: str) -> ChatResponse:
        await self._enforce_rate_limit(request.user_id)
        started = time.perf_counter()
        session_id = request.session_id or new_id("sess_")
        async with request_session(self._session_factory) as db:
            state = await self._initial_state(db, request, session_id, trace_id)
            result = await self._invoke(state)
            response = self._to_response(result, session_id=session_id, trace_id=trace_id)
            await self._persist(db, request, response, summary=result.get("summary") or "")
            await db.commit()
        self._remember_pending(request, response)
        self._observe(response, started)
        return response
```

把本例带进去，按实际执行顺序读：

1. 用 `request.user_id` 做限流，限流不通过就抛出异常；后续主流程不会执行。
2. `time.perf_counter()` 获取适合测量耗时的计时值。
3. 如果请求包含 `session_id`，复用它；否则创建 `sess_` 前缀的新 ID。本例未提供 ID，因此会尝试创建新 ID，具体内容由运行时生成。
4. `async with ... as db` 进入一个请求范围的数据库上下文，得到 `AsyncSession`。
5. `_initial_state` 从请求和历史消息构造图的输入。
6. `_invoke` 执行图，等待图给出最终状态。
7. `_to_response` 把内部状态转换成 HTTP 响应模型。
8. `_persist` 把本轮用户消息与助手答案加入数据库 session。
9. `db.commit()` 提交事务。正常返回路径先提交，再返回响应。
10. 离开数据库上下文后保存可能存在的待确认动作，记录观测指标与总耗时，再返回。

`_initial_state` 和 `_invoke` 都需要等待异步操作，所以前面有 `await`；`_to_response` 只是本地组装数据，是普通函数，不加 `await`。

限流方法源码很短：

```python
    async def _enforce_rate_limit(self, user_id: str) -> None:
        allowed = await self._rate_limiter.allow(f"user:{user_id}")
        if not allowed:
            raise RateLimitError()
```

`f"user:{user_id}"` 是 f-string，会把变量值填进字符串。限流 key 按用户区分；Redis 或内存实现由容器注入。`raise` 把当前正常流程打断，交给上层异常处理。

## 7. request_session：为什么不在共享图里存 db

源码：[app/core/request_scope.py](../../app/core/request_scope.py)，相关定义和完整上下文函数。

```python
_db_session: ContextVar[AsyncSession | None] = ContextVar("db_session", default=None)


def get_db_session() -> AsyncSession:
    session = _db_session.get()
    if session is None:
        raise RuntimeError("no request-scoped database session")
    return session


@asynccontextmanager
async def request_session(
    factory: async_sessionmaker[AsyncSession],
) -> AsyncIterator[AsyncSession]:
    async with factory() as session:
        token: Token = _db_session.set(session)
        try:
            yield session
        finally:
            _db_session.reset(token)
```

1. `factory()` 创建这次请求使用的数据库 session；内层 `async with` 在离开时负责关闭它。
2. `_db_session.set(session)` 把 session 绑定到当前异步上下文，返回一个用于恢复的 token。这里的 token 不是 LLM token。
3. `yield session` 把 session 交给 `chat()` 中的变量 `db`。
4. 图里的工具可以通过 `get_db_session()` 取得当前上下文的数据库 session，而不需要在初始化共享图时塞入某一次请求的 session。
5. `finally` 用 token 恢复原先的绑定，避免后续请求错误复用。

`ContextVar` 表达“当前执行上下文的值”，比在共享 `ChatService` 上写 `self.db = db` 更适合隔离并发请求。它不会自动把同一个 session 变成可以被任意并发任务同时操作的安全对象。

**事务边界在 `chat()` 中的 `await db.commit()`。** `request_session` 自己没有自动 commit。SQLAlchemy session 关闭时，尚未提交的事务会回滚；已经提交过的事务不会因为后面的指标记录失败而自动撤销。

这也是工程上的一个代价：当前结构把读历史、图调用与模型等待都放在数据库 session 的上下文里，数据库事务/连接占用可能跨越较长处理过程。理解代码时先看清现状，不要把上下文管理器误认为“这里只在最后写库时才涉及数据库”。

## 8. _initial_state：把 API 输入改造成图需要的字典

源码：[ChatService._initial_state](../../app/services/chat_service.py)，完整函数。

```python
    async def _initial_state(
        self,
        db: AsyncSession,
        request: ChatRequest,
        session_id: str,
        trace_id: str,
    ) -> AgentState:
        confirmed_action = self._resolve_confirmation(request, session_id)
        sessions = SessionRepository(db)
        await sessions.get_or_create(session_id, request.user_id)
        history = await sessions.recent_messages(session_id)
        return {
            "trace_id": trace_id,
            "session_id": session_id,
            "user_id": request.user_id,
            "user_query": request.message,
            "messages": history,
            "summary": "",
            "retrieved_chunks": [],
            "tool_results": [],
            "citations": [],
            "refused": False,
            "step_count": 0,
            "token_usage": TokenUsage(),
            "latency": LatencyBreakdown(),
            "confirm_action_id": request.confirm_action_id,
            "confirmed_action": confirmed_action,
            "idempotency_key": request.idempotency_key,
        }
```

逐步读：

1. `_resolve_confirmation` 处理人工确认操作。如果没有 `confirm_action_id` 就返回 `None`，本例走这一种情况。
2. `SessionRepository(db)` 创建一个使用当前数据库 session 的仓储对象。Repository 把 SQL 操作封装在一起，ChatService 不需要在这里写 SQL。
3. `get_or_create` 获取或新建会话行；`recent_messages` 读最近的历史消息，默认最多 20 条，并整理成时间正序。
4. 返回图的初始状态字典：问题文本进入 `user_query`；历史进入 `messages`；待检索、待执行的列表先置空；用量和耗时模型先取默认值。

本例中的 `user_query` 一定等于请求中的“退货政策是什么？”。首次新建且没有历史记录时，`messages` 会是空列表。注意当前问题单独放在 `user_query`，不是这一步立即 append 到 `messages`。

`summary` 这里初始化为 `""`。虽然仓储支持保存摘要，这个函数没有把已有的数据库摘要加载回来；读代码时不能因为看见 `save_summary` 就推定整条摘要恢复链路都已实现。

### 8.1 Pydantic 与 TypedDict 的区别一定要分清

[AgentState 定义](../../app/agents/state.py) 的**源码节选**：

```python
class AgentState(TypedDict, total=False):
    trace_id: str
    session_id: str
    user_id: str
    user_query: str
    messages: list[ChatMessage]
    summary: str
    intent: IntentResult | None
    strategy: RetrievalStrategy | None
    retrieved_chunks: list[RetrievedChunk]
    final_answer: str | None
    citations: list[Citation]
    refused: bool
    token_usage: TokenUsage
    latency: LatencyBreakdown
```

| 代码结构 | 运行时是什么 | 是否像 Pydantic 一样自动校验 |
|---|---|---|
| `ChatRequest(BaseModel)` | Pydantic 模型实例，用 `request.message` 访问 | 会按模型规则校验输入 |
| `ChatResponse(BaseModel)` | Pydantic 模型实例 | 会校验构造数据，并支持序列化 |
| `AgentState(TypedDict)` | 普通字典，用 `state["user_query"]` 或 `.get(...)` 访问 | TypedDict 自身不负责运行时校验 |

`TypedDict` 是在类型层面描述“这个字典允许哪些键，各自是什么类型”。`total=False` 表示定义中的键可以不出现，因此初始状态不必一次就填好 `intent`、`strategy`、`final_answer`。它们由后续图节点逐步补上。

“教学示意”，不是运行日志：

```python
# 输入对象：Pydantic 模型
request.message  # "退货政策是什么？"

# 初始状态：普通 dict
state["user_query"]  # "退货政策是什么？"
state.get("final_answer")  # 初始时没设置这个键，得到 None
```

`.get("key")` 在键不存在时默认返回 `None`；`state["key"]` 若键不存在会抛出 `KeyError`。这解释了后面为什么经常使用 `.get(...)`。

## 9. _invoke：真正把控制权交给 LangGraph

源码：[ChatService._invoke](../../app/services/chat_service.py)，完整函数。

```python
    async def _invoke(self, state: AgentState) -> AgentState:
        try:
            # LangGraph widens the state to dict[str, Any]; the graph contract is AgentState.
            return cast(AgentState, await self._graph.ainvoke(state))
        except GuardrailError as exc:
            return {
                **state,
                "final_answer": exc.message,
                "refused": True,
                "error": exc.error_code.value,
            }
        except AppError:
            raise
```

正常路径只有一个核心动作：`await self._graph.ainvoke(state)`。图已经在 `__init__` 构建好了，这里把本次初始状态送进去，等待图执行结束并拿回结果。图内部的护栏、分类、路由、检索与生成，交给下一篇逐节点讲解；本篇先把调用边界弄清楚。

这里尤其要理解 `cast`：

```python
cast(AgentState, some_value)
```

它告诉静态类型检查器：“此处按 AgentState 看待这个值。”**它不会校验字典字段，不会补默认值，也不会把错误数据转换成正确数据。** 运行时基本仍然是原来的对象。代码使用它是因为 LangGraph 的返回类型比项目约定更宽泛。

异常路径有两类：

1. 遇到 `GuardrailError`，把护栏错误转成可返回的拒答状态。`**state` 将原字典的键值展开复制到新字典，后面同名键覆盖前面的值，所以 `refused` 被设为 `True`。
2. 遇到其他 `AppError`，裸 `raise` 原样向上抛出，让 FastAPI 层的错误处理器生成错误响应；它不会在这里伪造普通聊天答案。

其他未在这里捕获的异常也会向上传播，不等于每种故障都会得到 `ChatResponse`。

对于“退货政策是什么？”，这里只能确定该问题进入图执行。把它写死为某个意图、策略或具体答案，都会超出这段源码能够证明的范围。

## 10. _to_response：把内部状态压成对外契约

源码：[ChatService._to_response](../../app/services/chat_service.py)，完整函数。

```python
    def _to_response(
        self,
        result: AgentState,
        *,
        session_id: str,
        trace_id: str,
    ) -> ChatResponse:
        answer = result.get("final_answer") or "暂时无法回答，请稍后重试。"
        strategy = result.get("strategy")
        intent = result.get("intent")
        return ChatResponse(
            answer=answer,
            session_id=session_id,
            trace_id=trace_id,
            intent=intent.label if intent else None,
            complexity=result.get("query_complexity"),
            strategy=strategy,
            citations=result.get("citations") or [],
            pending_action=result.get("pending_action"),
            refused=bool(result.get("refused")),
            error_code=result.get("error"),
            token_usage=result.get("token_usage") or TokenUsage(),
            latency=result.get("latency") or LatencyBreakdown(),
        )
```

按数据流理解：

1. 从图结果中取最终答案，缺失或空字符串时使用兜底文本。`x or y` 在 `x` 为假值时取 `y`，不仅判断 `None`。
2. 图内 `intent` 是包含标签等信息的 `IntentResult`，返回时只取 `intent.label`。
3. 把策略、引用、待确认操作、拒答标志、用量和耗时等填进 `ChatResponse`。
4. 图中的中间信息，如改写查询、完整检索内容、执行计划，不会因为存在于状态字典里就全部自动返回。

`bool(...)` 把值转为布尔值；未提供 `refused` 时 `.get` 为 `None`，最终得到 `False`。

输出模型源码：[ChatResponse](../../app/schemas/chat.py)，完整类。

```python
class ChatResponse(BaseModel):
    answer: str
    session_id: str
    trace_id: str
    intent: IntentLabel | None = None
    complexity: QueryComplexity | None = None
    strategy: RetrievalStrategy | None = None
    citations: list[Citation] = Field(default_factory=list)
    pending_action: ActionProposal | None = None
    refused: bool = False
    error_code: str | None = None
    token_usage: TokenUsage = Field(default_factory=TokenUsage)
    latency: LatencyBreakdown = Field(default_factory=LatencyBreakdown)
```

`Field(default_factory=list)` 表示“每个新模型都调用 `list()` 创建默认列表”；`default_factory=TokenUsage` 则创建新的 `TokenUsage()`。它表达默认值如何构建，避免不同响应对象误用同一份可变数据的心智模型。

“教学示意”：若某次图结果的 `final_answer` 是一段政策解释，`_to_response` 会把那段解释放入 `answer`。它本身不向模型提问，也不会校对政策；这些工作发生在前面的图节点中。

## 11. _persist：加入数据库 session 和 commit 是两步

源码：[ChatService._persist](../../app/services/chat_service.py)，完整函数。

```python
    async def _persist(
        self,
        db: AsyncSession,
        request: ChatRequest,
        response: ChatResponse,
        *,
        summary: str = "",
    ) -> None:
        sessions = SessionRepository(db)
        await sessions.append_message(response.session_id, "user", request.message)
        await sessions.append_message(response.session_id, "assistant", response.answer)
        if summary:
            await sessions.save_summary(response.session_id, summary)
```

执行顺序是：取得仓储对象，加入用户消息，加入助手消息，有非空摘要时更新摘要。对于本例，用户消息内容是“退货政策是什么？”，助手消息内容来自 `response.answer`，不是在这一层重新生成。

需要继续追一层：[SessionRepository.append_message](../../app/repositories/session_repository.py)，完整函数。

```python
    async def append_message(self, session_id: str, role: str, content: str) -> None:
        self._session.add(MessageRow(session_id=session_id, role=role, content=content))
        row = await self._session.get(SessionRow, session_id)
        if row is not None:
            row.updated_at = utcnow()
```

`MessageRow(...)` 创建数据库映射对象，`.add(...)` 把它交给 SQLAlchemy session 管理。后续查询或提交可能触发 flush，把变更发送给数据库；但 **flush 不等于 commit**。

- `flush`：让当前事务中的 SQL 变更发送给数据库，事务还没有最终提交。
- `commit`：提交当前事务，让本次事务的变更生效。
- 回滚：撤销本次尚未提交事务中的变更。

所以 `_persist` 返回，不代表这轮消息已经完成事务提交。真正的 `await db.commit()` 在它的调用方 `chat()` 中，保证正常路径上的用户消息与助手消息按同一事务提交。

这种事务边界也有实际含义：如果模型执行成功而最终提交数据库失败，`chat()` 正常返回路径会被异常打断；不能简单认为“答案已经生成，HTTP 就必然成功”。另外，数据库事务不能撤销外部系统已经发生的动作，不能把它理解成整个 Agent 世界的全局事务。

## 12. 最后把本例的数据变化对一遍

下面是**教学示意**，其中占位符表示运行时值，不是项目的一次真实执行记录。

| 位置 | 数据形态 | 本例中最值得观察的值 |
|---|---|---|
| HTTP 请求 | JSON | `message = "退货政策是什么？"` |
| 路由参数 | `ChatRequest` | `payload.message`，首次无 `session_id` |
| `chat()` | 局部变量 | 新的 `session_id`，本次 `trace_id` |
| `_initial_state()` | `AgentState` 普通字典 | `user_query` 为原问题；加载历史；空检索结果 |
| `_invoke()` | 传入图、返回图状态 | 图逐步产生分类、策略、证据、最终答案等 |
| `_to_response()` | `ChatResponse` | `answer`、引用、策略、用量、耗时等对外字段 |
| `_persist()` + `commit()` | 数据库记录 | 一条 user 消息和一条 assistant 消息 |
| 返回 HTTP | JSON | Pydantic/FastAPI 序列化响应模型 |

再用“谁负责什么”检查一次理解：

- FastAPI 路由负责接 HTTP 请求，交给服务；Pydantic 负责输入输出结构与字段校验。
- 容器负责在启动时组装组件；ChatService 负责安排一轮对话的执行顺序。
- LangGraph 负责按图执行节点；AgentState 是节点之间传递的状态字典约定。
- Repository 负责具体数据库访问；ChatService 决定正常聊天路径在哪一步 commit。
- `cast` 只影响静态类型认知，不能代替校验；`await` 只表达异步等待，不能代替事务或并发控制。

读完本篇，可以自己沿着这几个调用跳转一次：`lifespan → build_container → ChatService.__init__ → 路由 chat → ChatService.chat → _initial_state → _invoke → _to_response → _persist`。下一步再进入 [build_graph](../../app/agents/graph.py)，看“问题进图后，具体按哪些条件走向哪些节点”。
