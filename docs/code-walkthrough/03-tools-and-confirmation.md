# 工具调用与用户确认：逐步读懂“取消 ORD10003”

这一章沿着“帮我取消订单 ORD10003”阅读真实代码。你会看到：模型怎样提出操作、程序怎样要求确认、确认后怎样找到 Python 工具、工具怎样修改数据库。

本文没有启动服务，也没有发送取消请求。所有请求 JSON、响应 JSON、动作编号和执行结果都是说明用的示例。`ORD10003` 在 [演示数据定义](../../app/services/seed.py) 中属于 `u_demo`，初始状态是 `PAID`；这不代表你现在数据库中的订单仍处于这个状态，种子程序也不会覆盖已有订单。

## 1. 先分清这几个对象

| 名称 | 在这里代表什么 | 本例中的值或例子 |
| --- | --- | --- |
| Tool | 一个能被程序调用的业务能力，由 Python 类实现 | `CancelOrderTool` |
| ToolRegistry | 按工具名查找工具对象的字典 | `"cancel_order"` → 一个 `CancelOrderTool` 对象 |
| Skill | 某类业务的步骤提示和工具名单 | `order_cancellation` |
| Planner | 把用户问题转换成结构化计划的代码 | 计划使用 `cancel_order` |
| Executor | 检查权限、处理幂等并调用工具的代码 | 执行确认过的取消操作 |
| Repository | 负责查询和修改数据库的代码 | `OrderRepository` |
| Context | 本次调用是谁发起的、属于哪个会话等背景信息 | `user_id`、`session_id`、`trace_id` |

一次业务操作会经过多层，不是模型直接操作数据库：

```text
用户提出取消 → Planner 生成待确认动作 → 返回给前端
用户确认动作 → ChatService 验证确认 → Planner 恢复原动作
→ Executor 查工具、查权限、查幂等 → Tool 验参数、验归属
→ Repository 验状态并修改数据 → ChatService 提交事务
```

“是否执行”与“是否允许取消这个订单”是两个问题。前者涉及用户确认；后者涉及订单归属和当前状态，两者都要满足。

## 2. 先看外部请求：确认前后传什么

以下示意按默认的 `POST /api/v1/chat` 接口说明。实际前缀由 `api_v1_prefix` 配置决定。

### 第一次：提出取消要求

下面只是请求体示意，**没有发送**：

```json
{
  "message": "帮我取消订单 ORD10003",
  "user_id": "u_demo",
  "session_id": "sess_demo_cancel"
}
```

`session_id` 可以在首轮省略，由后端生成；这里写出来便于理解两次请求属于同一会话。真实前端应使用后端返回的会话编号。

如果规划器确实规划出取消操作，首次响应的相关字段会类似下面这样。此处省略了意图、延迟等其他字段，也没有实际生成这个动作编号：

```json
{
  "answer": "确认后将取消订单 ORD10003。请核对订单号。",
  "session_id": "sess_demo_cancel",
  "pending_action": {
    "action_id": "act_example_001",
    "tool_name": "cancel_order",
    "arguments": {
      "order_id": "ORD10003"
    },
    "risk": "destructive",
    "summary": "确认后将取消订单 ORD10003。请核对订单号。"
  }
}
```

此时只有“准备执行什么”的提案，不能据此说订单已经取消。当前实现中，只要存在尚未确认的 `pending_action`，执行器会跳过整份计划，连计划中的查询工具也不会执行。后文会看到这一判断。

### 第二次：确认第一次返回的动作

下面仍是**未发送的请求体示意**。真正调用时，`confirm_action_id` 必须来自后端首次响应，不能照抄本文示例值：

```json
{
  "message": "确认取消",
  "user_id": "u_demo",
  "session_id": "sess_demo_cancel",
  "confirm_action_id": "act_example_001"
}
```

关键是 `confirm_action_id`，不是“确认取消”这四个字。只发自然语言“确认”而不传动作编号，不会自动关联到上次保存的动作；服务端也不会把随意填写的动作编号视为有效确认。

第二次不要求客户端重新提交工具名和参数：服务端会找到此前保存的动作。这样可以确认“之前展示的那个动作”，而不是让模型根据确认消息重新猜测要做什么。

如果动作有效、订单归属正确、状态允许，且事务提交成功，返回的相关字段才可能类似以下**结果示意**：

```json
{
  "answer": "订单 ORD10003 已取消。",
  "session_id": "sess_demo_cancel",
  "pending_action": null
}
```

在线模型可能使用不同措辞；本章没有验证当前数据库状态，也没有执行这项操作。

## 3. Tool 基类：所有具体工具共用的执行入口

### 3.1 参数为什么要用 Pydantic

取消订单工具的输入类型定义如下，展示的是这个类的完整定义：

<!-- source:app/tools/aftersale_tools.py:11:17 -->
源码：[app/tools/aftersale_tools.py](../../app/tools/aftersale_tools.py)，第 11–17 行。

```python
class OrderIdInput(BaseModel):
    order_id: str = Field(
        min_length=3,
        max_length=32,
        pattern=r"^[A-Za-z0-9_-]+$",
        description="订单号，如 ORD1001；不要填 refund_id 或 user_id",
    )
```

`BaseModel` 来自 Pydantic，可以把外部字典转换成有字段的对象，并检查字段是否合法。`Field` 定义的约束包含：订单号至少 3 个字符、最多 32 个字符，只允许字母、数字、下划线和连字符。

例如，`{"order_id": "ORD10003"}` 符合这个输入结构；`{}` 缺少必填字段，`{"order_id": "这是中文订单号"}` 不符合这里的字符规则。注意，**格式正确不代表订单存在，更不代表属于当前用户**；这些要继续去数据库检查。

### 3.2 `parse()` 与 `run()` 的完整实现

下面展示 `Tool` 类中的两个完整方法：

<!-- source:app/tools/base.py:59:78 -->
源码：[app/tools/base.py](../../app/tools/base.py)，第 59–78 行。

```python
    def parse(self, arguments: dict[str, Any]) -> BaseModel:
        """Validate raw arguments against ``input_model``.

        Raises:
            InvalidRequestError: If the arguments do not match the schema.
        """
        try:
            return self.input_model.model_validate(arguments)
        except ValidationError as exc:
            raise InvalidRequestError(f"invalid arguments for {self.name}: {exc}") from exc

    async def run(self, arguments: dict[str, Any], context: ToolContext) -> ToolResult:
        """Validate, then execute, mapping unexpected errors to a tool failure."""
        parsed = self.parse(arguments)
        try:
            return await self.execute(parsed, context)
        except InvalidRequestError:
            raise
        except Exception as exc:
            raise ToolExecutionError(f"{self.name} failed") from exc
```

按执行顺序读：

1. 执行器调用 `tool.run(arguments, context)`。
2. `self.parse(arguments)` 使用具体工具的 `input_model` 验证字典。
3. 验证通过，得到 `parsed` 对象；取消订单场景中，它是 `OrderIdInput`。
4. `await self.execute(parsed, context)` 调用具体工具实现的业务方法。
5. 参数有误或业务显式抛出 `InvalidRequestError` 时，保留这一错误；其他执行异常包装为 `ToolExecutionError`。

这里有几个 Python 写法值得停下来理解：

| 写法 | 怎么读 |
| --- | --- |
| `self` | 当前这个工具对象。调用对象方法时，Python 自动把它传进来 |
| `arguments: dict[str, Any]` | 参数是一个字典，键是字符串，值可以是不同类型 |
| `-> BaseModel` / `-> ToolResult` | 返回值的类型提示；单纯类型提示不会自动执行校验，实际校验来自 `model_validate()` |
| `async def` | 定义异步函数 |
| `await` | 等待异步操作完成；等待数据库等 I/O 时，事件循环可以继续处理其他任务 |
| `try / except` | 尝试运行代码，发生指定异常时进入对应处理分支 |
| `raise ... from exc` | 抛出更适合当前层理解的异常，同时保留原异常作为原因 |

`parsed = self.parse(arguments)` 在 `run()` 的 `try` 之前。参数校验失败时，`parse()` 自己已经将 Pydantic 的 `ValidationError` 转成项目的 `InvalidRequestError`，异常会直接交给上层。

`Tool` 是父类，`CancelOrderTool` 是具体子类。父类的 `run()` 负责公共步骤，子类的 `execute()` 负责具体业务；所以执行器可以用同一种方式调用各种工具。

### 3.3 `context` 为什么不放进业务参数

`ToolContext` 的字段来自以下原样片段；类的说明文字未展示：

<!-- source:app/tools/base.py:25:29 -->
源码：[app/tools/base.py](../../app/tools/base.py)，第 25–29 行。

```python
    user_id: str
    session_id: str
    trace_id: str
    confirmed_action_id: str | None = None
    idempotency_key: str | None = None
```

同一项业务操作有两类信息：

- `arguments` / `parsed`：**要做什么**，例如取消哪个 `order_id`。
- `context`：**这次调用的背景**，例如由哪个用户发起、在哪个会话、使用哪个追踪编号。

`trace_id` 用于串联日志和审计，不是订单号。`confirmed_action_id` 用于标识已确认动作；`idempotency_key` 用于重复提交时去重。

它们分开传递，有助于避免模型把业务参数中的 `user_id` 当作当前身份。当前项目的真正身份来源仍存在边界：`context.user_id` 最终来自请求体里的 `user_id`，不是认证系统验证出的用户，详见最后一节。

## 4. Registry：一个工具名字怎样变成 Python 对象

下面是注册表的初始化、注册和查找三个完整方法：

<!-- source:app/tools/registry.py:9:20 -->
源码：[app/tools/registry.py](../../app/tools/registry.py)，第 9–20 行。

```python
class ToolRegistry:
    def __init__(self) -> None:
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool) -> None:
        self._tools[tool.name] = tool

    def get(self, name: str) -> Tool:
        try:
            return self._tools[name]
        except KeyError as exc:
            raise InvalidRequestError(f"unknown tool: {name}") from exc
```

可以把 `_tools` 想成普通 Python 字典，其结构类似下面的**概念示意**，不是文件里的原代码：

```python
{
    "cancel_order": 一个 CancelOrderTool 对象,
    "get_order": 一个 GetOrderTool 对象,
}
```

`register(tool)` 把对象放进字典；`get("cancel_order")` 根据名字取回对象。名字不存在时，Python 字典产生 `KeyError`，这里把它改成项目能统一处理的 `InvalidRequestError`。

真实初始化在 [build_tool_registry()](../../app/tools/factory.py) 中完成，其中创建了 `CancelOrderTool()` 并注册。模型产生的是工具名和参数；执行器把工具名交给注册表，才找到实际可以运行的 Python 对象。

这也是为什么“模型输出一个名字”不等于可以执行任意函数：只有进入注册表的工具，才有可能被这里的代码找到。不过，工具注册和后面的 Skill 白名单都不能替代用户身份认证。

## 5. Skill：先缩小本次规划能使用的工具集合

### 5.1 取消订单 Skill 的完整定义

<!-- source:app/skills/definitions/order_cancellation.json:1:11 -->
源码：[app/skills/definitions/order_cancellation.json](../../app/skills/definitions/order_cancellation.json)，第 1–11 行。

```json
{
  "name": "order_cancellation",
  "description": "取消尚未发货订单的确认流程",
  "triggers": ["取消订单", "不想要了"],
  "steps": [
    "核对订单号与当前状态",
    "明确用户希望取消后，先展示待确认操作",
    "收到确认后执行取消"
  ],
  "tools": ["get_order", "cancel_order"]
}
```

`triggers` 用于匹配用户原话；本例包含“取消订单”，能匹配该 Skill。匹配代码在 [match_skill()](../../app/skills/loader.py) 中，先按触发词找，找不到再根据意图映射找，最多选择一个 Skill。

`steps` 是给规划器看的业务步骤提示；它本身不会执行 Python，也不会保证“核对状态”已经发生。`tools` 才是后续收缩工具集合使用的名单。

### 5.2 白名单是怎样计算并实际过滤计划的

下面是 `_allowed_tools()` 的完整实现：

<!-- source:app/agents/nodes/planner.py:80:86 -->
源码：[app/agents/nodes/planner.py](../../app/agents/nodes/planner.py)，第 80–86 行。

```python
def _allowed_tools(skill: dict | None, registry: ToolRegistry) -> set[str] | None:
    if skill is None:
        return None
    names = set(registry.names())
    allowed = {name for name in skill.get("tools", []) if name in names}
    allowed.update(name for name in _ALWAYS_AVAILABLE if name in names)
    return allowed or None
```

`set` 表示集合。这里先从 Skill 的 `tools` 中保留已经注册的工具，再额外加入始终允许的 `search_knowledge`、`escalate_to_human`（定义在同文件的 `_ALWAYS_AVAILABLE`）。

对于当前取消订单 Skill，集合通常为：`get_order`、`cancel_order`、`search_knowledge`、`escalate_to_human`。没有匹配 Skill 时返回 `None`，在这条代码路径中表示不按某个 Skill 收缩工具，而非禁止所有工具。

仅在提示词里写“请只用这些工具”还不够，模型返回后还有实际过滤。下面是 `_sanitize_plan()` 的完整实现：

<!-- source:app/agents/nodes/planner.py:97:106 -->
源码：[app/agents/nodes/planner.py](../../app/agents/nodes/planner.py)，第 97–106 行。

```python
def _sanitize_plan(
    plan: AgentPlan,
    registry: ToolRegistry,
    allowed: set[str] | None,
) -> AgentPlan:
    names = set(registry.names())
    if allowed is not None:
        names = names & allowed
    steps = [step for step in plan.steps if not step.tool_name or step.tool_name in names]
    return AgentPlan(steps=steps[:6], rationale=plan.rationale)
```

`names & allowed` 是两个集合的交集。随后删除包含不允许工具名的步骤，并最多保留 6 步。`tool_name` 为空的步骤可以保留，但执行器会跳过它们。

`AgentPlan` 和 `PlanStep` 是 [Pydantic 数据模型](../../app/schemas/agent.py)。一个计划步骤的关键字段是 `tool_name` 和 `arguments`，本例对应 `cancel_order` 与 `{"order_id": "ORD10003"}`。

在线路径是“请求 LLM 返回 JSON → 解析为 AgentPlan → 过滤”；离线路径和模型失败时可按 Skill 工具顺序构造计划。两者最终都会经过下面的确认逻辑。当前执行器不会自动把某一步查询结果填进下一步参数，也没有执行后重新规划的循环。

## 6. Planner：明确要求取消，也要先生成待确认动作

### 6.1 第一次请求的关键分支

以下是 `planner_node()` 第 186–229 行的连续原样片段；函数开头处理已确认动作的部分暂放到后文：

<!-- source:app/agents/nodes/planner.py:186:229 -->
源码：[app/agents/nodes/planner.py](../../app/agents/nodes/planner.py)，第 186–229 行。

```python
    intent = state.get("intent")
    plan = await planner.create_plan(
        state["user_query"],
        intent.label.value if intent else "faq_policy",
        state.get("summary") or "",
    )
    write_steps = [
        step
        for step in plan.steps
        if step.tool_name and planner._registry.get(step.tool_name).risk in WRITE_RISKS
    ]
    if any(
        _explicit_write_request(state["user_query"], step.tool_name or "") for step in write_steps
    ):
        if any(not step.arguments.get("order_id") for step in write_steps):
            return {
                "plan": plan.model_copy(update={"steps": []}),
                "extra": {"missing_order_id": True},
            }
    plan = plan.model_copy(
        update={
            "steps": [
                step
                for step in plan.steps
                if step not in write_steps
                or _explicit_write_request(state["user_query"], step.tool_name or "")
            ]
        }
    )
    pending = None
    for step in plan.steps:
        if not step.tool_name:
            continue
        tool = planner._registry.get(step.tool_name)
        if tool.risk in WRITE_RISKS and not state.get("confirm_action_id"):
            pending = ActionProposal(
                action_id=new_id("act_"),
                tool_name=tool.name,
                arguments=step.arguments,
                risk=tool.risk,
                summary=_action_summary(tool.name, step.arguments),
            )
            break
    return {"plan": plan, "pending_action": pending}
```

这段做四件事：

1. `planner.create_plan(...)` 获取候选计划。
2. 从计划中找出风险为 `WRITE` 或 `DESTRUCTIVE` 的步骤。
3. 判断用户是在明确要求执行，还是只问政策；必要时删掉写步骤，或标记缺少订单号。
4. 对首个尚未确认的写步骤创建 `ActionProposal`，放进 `pending_action`。

`state` 是整个图传递的数据字典。`state.get("intent")` 用 `.get()` 读取，字段不存在时得到 `None`；`state["user_query"]` 用方括号读取，字段缺失会产生 `KeyError`。

`model_copy(update={...})` 是创建带有更新内容的数据模型副本。此处主要用于替换计划步骤列表，没有操作数据库。

`_explicit_write_request()` 的完整判断如下：

<!-- source:app/agents/nodes/planner.py:232:254 -->
源码：[app/agents/nodes/planner.py](../../app/agents/nodes/planner.py)，第 232–254 行。

```python
def _explicit_write_request(query: str, tool_name: str) -> bool:
    if any(
        cue in query
        for cue in (
            "规则",
            "政策",
            "条件",
            "怎么",
            "能不能",
            "是否",
            "可以吗",
            "能退吗",
            "能取消吗",
        )
    ):
        return False
    if tool_name == "cancel_order":
        return any(cue in query for cue in ("取消", "不想要了"))
    if tool_name == "create_refund_request":
        return any(cue in query for cue in ("申请退款", "帮我退款", "我要退款"))
    if tool_name == "create_return_request":
        return any(cue in query for cue in ("申请退货", "帮我退货", "我要退货", "提交退货"))
    return False
```

例如，“取消订单 ORD10003”含“取消”，可以被视为明确要求；“ORD10003 能取消吗”含“能取消吗”，会被当作询问而非执行。这是基于关键词的简单规则，存在自然语言覆盖不全的情况，不能当作完备的语义理解。

### 6.2 为什么生成动作后不会马上执行

执行器入口的这一段负责拦住尚未确认的计划。它是 `executor_node()` 第 95–105 行的连续原样片段：

<!-- source:app/agents/nodes/executor.py:95:105 -->
源码：[app/agents/nodes/executor.py](../../app/agents/nodes/executor.py)，第 95–105 行。

```python
async def executor_node(
    state: AgentState,
    *,
    registry: ToolRegistry,
    settings: Settings,
) -> dict:
    plan = state.get("plan")
    if plan is None:
        return {"tool_results": []}
    if state.get("pending_action") and not state.get("confirm_action_id"):
        return {"tool_results": []}
```

`if state.get("pending_action") and not state.get("confirm_action_id")` 的意思是：“有待确认动作，而且没有确认编号，就直接返回空工具结果”。

这里返回的是整个函数的结果，因此跳过的是整份计划，不仅是写步骤。即便 Skill 提示“先查询订单”，本轮也不会先执行查询再展示确认；订单归属与状态的强制检查实际在确认后的工具与仓储中完成。

风险标签也不会自己完成拦截。真正改变流程的是这段 `if`，以及后面执行器调用的 `assert_permitted()`。

## 7. ChatService：怎样保存提案，怎样验证确认

### 7.1 保存的是一个服务端对象

待确认记录的数据结构如下，包含动作本身及其所属用户、会话和过期时间：

<!-- source:app/services/chat_service.py:34:39 -->
源码：[app/services/chat_service.py](../../app/services/chat_service.py)，第 34–39 行。

```python
@dataclass(frozen=True)
class PendingConfirmation:
    session_id: str
    user_id: str
    action: ActionProposal
    expires_at: float
```

`@dataclass(frozen=True)` 表示用 Python 数据类保存这些字段，并禁止常规地重新给实例字段赋值。它不是数据库表。

[ChatService 初始化](../../app/services/chat_service.py) 的第 61 行创建了 `self._pending_actions: dict[str, PendingConfirmation] = {}`。这是**当前进程中的一个字典**。

下面是保存方法的完整实现：

<!-- source:app/services/chat_service.py:204:216 -->
源码：[app/services/chat_service.py](../../app/services/chat_service.py)，第 204–216 行。

```python
    def _remember_pending(self, request: ChatRequest, response: ChatResponse) -> None:
        now = time.monotonic()
        self._pending_actions = {
            key: value for key, value in self._pending_actions.items() if value.expires_at > now
        }
        action = response.pending_action
        if action is not None:
            self._pending_actions[action.action_id] = PendingConfirmation(
                session_id=response.session_id,
                user_id=request.user_id,
                action=action,
                expires_at=now + 15 * 60,
            )
```

`time.monotonic()` 适合计算经过了多久，不是给用户展示的日历时间。`now + 15 * 60` 表示从当前时刻起 15 分钟后过期。保存前还会过滤已过期记录。

字典中的逻辑结构可理解为以下**说明用伪代码**：

```text
act_example_001 → {
    user_id: u_demo,
    session_id: sess_demo_cancel,
    action: cancel_order(order_id=ORD10003),
    expires_at: 本进程单调时钟的未来时刻
}
```

`_remember_pending()` 在该次响应的会话消息提交后调用；客户端收到的 `action_id` 对应服务端保存的原动作。

### 7.2 第二次请求必须通过验证

下面是确认解析方法的完整实现：

<!-- source:app/services/chat_service.py:191:202 -->
源码：[app/services/chat_service.py](../../app/services/chat_service.py)，第 191–202 行。

```python
    def _resolve_confirmation(self, request: ChatRequest, session_id: str) -> ActionProposal | None:
        if not request.confirm_action_id:
            return None
        pending = self._pending_actions.get(request.confirm_action_id)
        if (
            pending is None
            or pending.expires_at < time.monotonic()
            or pending.session_id != session_id
            or pending.user_id != request.user_id
        ):
            raise InvalidRequestError("待确认操作不存在或已过期，请重新发起。")
        return pending.action
```

这几个 `or` 条件表示只要一个不满足，就报错：

- 找不到这个动作编号。
- 动作已经过期。
- 这次请求的会话与提案的会话不同。
- 这次请求的用户与提案的用户不同。

验证成功后返回的是 `pending.action`，也就是**服务端原先保存的动作**。客户端本次的自然语言消息不会替换这个动作里的工具参数。

`_initial_state()` 在 [chat_service.py 第 168 行](../../app/services/chat_service.py) 调用此方法，然后把结果放入状态的 `confirmed_action`；请求中的编号放入 `confirm_action_id`。

### 7.3 Planner 收到有效确认后恢复原动作

现在回看 `planner_node()` 的开头，这是第 169–185 行的连续原样片段：

<!-- source:app/agents/nodes/planner.py:169:185 -->
源码：[app/agents/nodes/planner.py](../../app/agents/nodes/planner.py)，第 169–185 行。

```python
async def planner_node(state: AgentState, planner: Planner) -> dict:
    confirmed = state.get("confirmed_action")
    if confirmed is not None:
        return {
            "plan": AgentPlan(
                steps=[
                    PlanStep(
                        step_id=1,
                        goal="confirmed action",
                        tool_name=confirmed.tool_name,
                        arguments=confirmed.arguments,
                    )
                ],
                rationale="confirmed_action",
            ),
            "pending_action": None,
        }
```

如果 `confirmed_action` 存在，就直接构造一个只有原确认动作的计划，并清空 `pending_action`。例如本例只恢复 `cancel_order({"order_id":"ORD10003"})`，不会重新让 LLM 解读“确认取消”。

[路由节点](../../app/agents/nodes/router.py) 也会在 `confirmed_action` 存在时选择 AGENT 策略，使请求进入计划和执行路径。这样的确认是应用代码跨两次 HTTP 请求完成的，没有使用 LangGraph 的 interrupt/checkpoint 来暂停和恢复图。

## 8. Executor：确认后还要检查权限和幂等

### 8.1 真正的权限判断函数

下面是风险集合与权限方法的完整实现：

<!-- source:app/tools/permissions.py:7:14 -->
源码：[app/tools/permissions.py](../../app/tools/permissions.py)，第 7–14 行。

```python
WRITE_RISKS = {ToolRisk.WRITE, ToolRisk.DESTRUCTIVE}


def assert_permitted(tool: Tool, *, confirmed: bool) -> None:
    if tool.risk in WRITE_RISKS and not confirmed:
        raise ToolPermissionError(
            f"{tool.name} is {tool.risk.value} and requires explicit user confirmation"
        )
```

只有 `WRITE` 和 `DESTRUCTIVE` 被归入这里的写风险集合。没有确认就抛出 `ToolPermissionError`。`EXTERNAL` 不在这个集合中，不能理解为“所有外部调用都要经过此确认”。

**`risk = ToolRisk.DESTRUCTIVE` 只是元数据；调用 `assert_permitted()` 才会进行这里的权限拦截。** 如果其他代码绕开执行器，直接调用工具的 `execute()`，这一检查不会自动发生。因此工具调用入口应保持受控。

这里的 `confirmed` 在执行器中只是 `bool(confirm_action_id)`。编号真实性不是这个布尔转换验证的；真正验证发生在前一节的 `ChatService._resolve_confirmation()`。

### 8.2 创建调用上下文，查找工具

以下是 `executor_node()` 第 107–129 行的连续原样片段：

<!-- source:app/agents/nodes/executor.py:107:129 -->
源码：[app/agents/nodes/executor.py](../../app/agents/nodes/executor.py)，第 107–129 行。

```python
    started = time.perf_counter()
    results: list[ToolResult] = []
    context = ToolContext(
        user_id=state["user_id"],
        session_id=state["session_id"],
        trace_id=state["trace_id"],
        confirmed_action_id=state.get("confirm_action_id"),
        idempotency_key=state.get("idempotency_key"),
    )
    idempotency = IdempotencyRepository(get_db_session())
    calls = 0
    query = state["user_query"]
    for step in plan.steps:
        if (
            calls >= settings.max_tool_calls
            or (state.get("step_count") or 0) >= settings.max_agent_steps
        ):
            break
        if not step.tool_name:
            continue
        tool = registry.get(step.tool_name)
        confirmed = bool(state.get("confirm_action_id"))
        try:
```

执行器创建 `ToolContext`，把状态中的用户、会话和确认编号交给工具。`IdempotencyRepository(get_db_session())` 使用当前请求的数据库会话，用于查询之前是否已执行过同一动作。

随后遍历计划步骤，用 `registry.get(step.tool_name)` 找到工具对象。达到工具调用预算时会 `break` 停止；工具名为空则 `continue` 跳过本步。代码没有并发执行这份计划，也没有执行结果驱动的重新规划。

这段结尾的 `try:` 接着下面的风险和幂等分支。

### 8.3 写工具的风险、去重和执行分支

以下是接续的第 130–150 行原样片段；它是外层 `try` 内的完整写风险分支：

<!-- source:app/agents/nodes/executor.py:130:150 -->
源码：[app/agents/nodes/executor.py](../../app/agents/nodes/executor.py)，第 130–150 行。

```python
            if tool.risk in WRITE_RISKS:
                assert_permitted(tool, confirmed=confirmed)
                idem_key = context.confirmed_action_id or context.idempotency_key
                if idem_key:
                    fp = _fingerprint(tool.name, step.arguments)
                    existing = await idempotency.get_if_matches(idem_key, fp)
                    if existing is not None:
                        results.append(
                            ToolResult(
                                name=tool.name, success=True, output=existing, risk=tool.risk
                            )
                        )
                        TOOL_CALLS.labels(tool=tool.name, status="idempotent").inc()
                        calls += 1
                        continue
                    result = await tool.run(step.arguments, context)
                    await idempotency.put(idem_key, fp, result.output)
                    TOOL_CALLS.labels(tool=tool.name, status="ok").inc()
                    results.append(result)
                    calls += 1
                    continue
```

本例会进入这条路径：

1. 检查工具风险，确认确实存在。
2. 选择幂等键 `idem_key`，优先用 `confirmed_action_id`，否则才用 `idempotency_key`。
3. 计算工具名与参数的指纹，查询是否已有结果。
4. 有相同结果就复用，不再调用工具。
5. 首次执行时调用 `await tool.run(...)`，然后保存结果。

“幂等”在这里可以理解为：同一个确认动作因网络重试提交多次，串行情况下应返回首次结果，避免再次取消或再次创建售后申请。`continue` 会开始下一轮循环，因此命中幂等结果后不会再落入下面的工具执行代码。

指纹函数的完整实现如下：

<!-- source:app/agents/nodes/executor.py:42:46 -->
源码：[app/agents/nodes/executor.py](../../app/agents/nodes/executor.py)，第 42–46 行。

```python
def _fingerprint(tool_name: str, arguments: dict) -> str:
    payload = json.dumps(
        {"tool": tool_name, "arguments": arguments}, sort_keys=True, ensure_ascii=False
    )
    return hashlib.sha256(payload.encode()).hexdigest()
```

`sort_keys=True` 让字典字段顺序一致，减少同一参数因键顺序不同产生不同指纹的情况；SHA-256 把工具名和参数转换为固定长度摘要。它用来比较请求内容，不是用户密码，也不是身份认证凭证。

幂等仓储的查询方法如下，展示完整方法：

<!-- source:app/repositories/idempotency.py:16:26 -->
源码：[app/repositories/idempotency.py](../../app/repositories/idempotency.py)，第 16–26 行。

```python
    async def get_if_matches(self, key: str, fingerprint: str) -> dict | None:
        row = await self._session.get(IdempotencyRow, key)
        if row is None:
            return None
        if row.fingerprint != fingerprint:
            raise AppError(
                "idempotency key reused with different payload",
                error_code=ErrorCode.IDEMPOTENCY_CONFLICT,
                http_status=409,
            )
        return json.loads(row.response_json)
```

相同键、相同指纹：读取原结果。相同键、不同指纹：抛出 `IDEMPOTENCY_CONFLICT`，避免把一个编号用于另一笔操作。这个底层异常配置了 HTTP 409，但在本条聊天执行路径中，Executor 会把 `AppError` 转成失败的 `ToolResult`，不能据此断言整个聊天接口一定返回 409。

`put()` 在 [同一个仓储文件](../../app/repositories/idempotency.py) 中把响应 JSON 加入数据库会话。它没有单独提交事务。

当前实现是“先查询，再执行业务，再写幂等记录”，没有对并发首次请求做原子占位或互斥锁。因此它具备常规重试去重逻辑，但不能直接宣称在所有并发情形下实现严格 exactly-once。并且确认记录仍须在本进程存在且未过期；只有持久化幂等结果，不代表重启后还可以凭旧确认编号重放。

### 8.4 出错时怎样返回

以下是执行器异常处理的连续原样片段：

<!-- source:app/agents/nodes/executor.py:154:166 -->
源码：[app/agents/nodes/executor.py](../../app/agents/nodes/executor.py)，第 154–166 行。

```python
        except ToolPermissionError as exc:
            TOOL_CALLS.labels(tool=tool.name, status="denied").inc()
            results.append(
                ToolResult(name=tool.name, success=False, error=str(exc), risk=tool.risk)
            )
        except AppError as exc:
            if exc.error_code.value == "IDEMPOTENCY_CONFLICT":
                TOOL_CALLS.labels(tool=tool.name, status="conflict").inc()
            else:
                TOOL_CALLS.labels(tool=tool.name, status="error").inc()
            results.append(
                ToolResult(name=tool.name, success=False, error=str(exc), risk=tool.risk)
            )
```

执行器将权限错误或业务错误变成 `success=False` 的结构化结果，并记录监控计数。`TOOL_CALLS.labels(...).inc()` 表示给某种工具调用结果增加一次计数，不是业务数据库变更。

如果数据库发生真正的事务错误，后续提交也可能失败；“捕获异常、包装 ToolResult”不会自动修复数据库会话，也不能被理解为成功完成操作。

## 9. CancelOrderTool：参数验证通过后，先验订单归属

下面展示 `CancelOrderTool` 的完整类定义：

<!-- source:app/tools/aftersale_tools.py:112:128 -->
源码：[app/tools/aftersale_tools.py](../../app/tools/aftersale_tools.py)，第 112–128 行。

```python
class CancelOrderTool(RepositoryBackedTool, Tool):
    name = "cancel_order"
    description = (
        "取消尚未发货的订单（destructive，须用户确认）。仅 status 为 created 或 paid 时可成功。"
        "何时用：用户要取消订单，且订单未发货。"
        "何时不用：已发货/已签收不要调用（会失败）；应改用 create_return_request。不要用本工具提交退款单。"
        "优先：未发货取消 → 本工具；已发货退货 → create_return_request。"
        "参数 order_id：例 ORD1001，不要填 sku。"
        "返回 {order_id, status:cancelled}；失败通常表示已发货或状态不允许取消。"
    )
    risk = ToolRisk.DESTRUCTIVE
    input_model = OrderIdInput

    async def execute(self, parsed: OrderIdInput, context: ToolContext) -> ToolResult:
        await self.repository.get_visible_order(parsed.order_id, context.user_id)
        output = await self.repository.cancel_order(parsed.order_id, trace_id=context.trace_id)
        return ToolResult(name=self.name, success=True, output=output)
```

`description` 主要给规划器/模型看，帮助选择正确工具。里面写“已发货不要调用”，本身不构成强制状态检查；真正的检查在仓储方法中。

两个父类分别提供两类能力：`RepositoryBackedTool` 提供 `repository` 属性；`Tool` 提供前面看到的 `parse()` 和 `run()`。执行器调用继承来的 `run()`，父类又会调用这里实现的 `execute()`。

这个 `execute()` 只有三行核心业务：

1. 根据订单号和 `context.user_id` 检查订单归属。
2. 调用仓储尝试取消订单，同时传入 `trace_id` 关联审计。
3. 把仓储返回的数据放入 `ToolResult`。

`get_visible_order()` 的完整实现如下：

<!-- source:app/repositories/order_repository.py:37:44 -->
源码：[app/repositories/order_repository.py](../../app/repositories/order_repository.py)，第 37–44 行。

```python
    async def get_visible_order(self, order_id: str, user_id: str) -> Order:
        """Return only an order owned by the supplied user, hiding others as not-found."""
        order = await self.get_order(order_id)
        if order is None:
            raise InvalidRequestError(f"order not found: {order_id}")
        if order.user_id != user_id:
            raise InvalidRequestError(f"order not found: {order_id}")
        return order
```

订单不存在，或者 `order.user_id` 与当前用户不同，都会抛出“order not found”。这可以避免接口直接告诉调用者“这个订单存在但属于另一个人”。

但此检查可靠的前提是 `context.user_id` 可信。当前请求体可自行填写用户编号，因此它完成了归属比较，却没有独立证明请求者身份。

### `self.repository` 究竟从哪里来

下面是 `RepositoryBackedTool` 的完整定义：

<!-- source:app/tools/deps.py:7:17 -->
源码：[app/tools/deps.py](../../app/tools/deps.py)，第 7–17 行。

```python
class RepositoryBackedTool:
    """Tools hold an optional repo for tests; production resolves the request session."""

    def __init__(self, repository: OrderRepository | None = None) -> None:
        self._repository = repository

    @property
    def repository(self) -> OrderRepository:
        if self._repository is not None:
            return self._repository
        return OrderRepository(get_db_session())
```

`__init__()` 是对象初始化方法。测试可以传入一个仓储对象；平时没有传，就调用 `get_db_session()` 取得当前请求的数据库会话，再创建 `OrderRepository`。

`@property` 让方法可以像字段一样使用：写 `self.repository`，而不是 `self.repository()`。

工具本身是进程级对象，但数据库会话通过 `ContextVar` 绑定到当前请求，相关实现见 [request_scope.py](../../app/core/request_scope.py)。这里的 `ContextVar` 是异步任务中的上下文存储机制，与存放用户信息的 `ToolContext` 不是同一种对象。

即使连续两次访问 `self.repository` 创建了两个仓储对象，它们也从当前请求取得同一个数据库会话，业务写入仍属于同一事务。

## 10. Repository：强制检查状态，然后修改数据库

下面是 `OrderRepository.cancel_order()` 的完整方法：

<!-- source:app/repositories/order_repository.py:123:137 -->
源码：[app/repositories/order_repository.py](../../app/repositories/order_repository.py)，第 123–137 行。

```python
    async def cancel_order(self, order_id: str, *, trace_id: str) -> dict:
        order = await self.get_order(order_id)
        if order is None:
            raise InvalidRequestError(f"order not found: {order_id}")
        if order.status not in {OrderStatus.CREATED, OrderStatus.PAID}:
            raise InvalidRequestError("only unpaid or paid-unshipped orders can be cancelled")
        try:
            await self.update_status(order.order_id, OrderStatus.CANCELLED)
            await self.write_audit(trace_id, "cancel_order", order.order_id)
            await self._session.flush()
        except InvalidRequestError:
            raise
        except Exception as exc:
            raise DatabaseError("failed to cancel order") from exc
        return {"order_id": order.order_id, "status": "cancelled"}
```

逐行按业务含义理解：

- `get_order(order_id)`：重新查询订单。这里没有在仓储方法内重复检查用户归属，因为正常工具路径已在前一步调用 `get_visible_order()`。
- `order is None`：订单不存在，停止操作。
- `status not in {CREATED, PAID}`：订单不是“待支付”或“已支付未发货”，停止操作。
- `update_status(..., CANCELLED)`：修改订单状态。
- `write_audit(...)`：增加取消操作的审计记录，关联 `trace_id`。
- `flush()`：把当前会话中的变更发送给数据库，仍然处于事务中。
- 返回字典：告诉上层本次方法准备好了哪些结果，最终成功还要等待外层事务提交。

函数参数里的 `*` 表示它之后的参数必须按名字传入，因此工具写的是 `trace_id=context.trace_id`，不能把这个值不带名字随意放到位置参数中。

状态更新方法如下，展示完整方法：

<!-- source:app/repositories/order_repository.py:56:60 -->
源码：[app/repositories/order_repository.py](../../app/repositories/order_repository.py)，第 56–60 行。

```python
    async def update_status(self, order_id: str, status: OrderStatus) -> None:
        order = await self._session.get(OrderRow, order_id)
        if order is None:
            return
        order.status = status.value
```

`self._session.get(OrderRow, order_id)` 通过主键取得数据库模型对象。`order.status = status.value` 修改 SQLAlchemy 管理的对象，后续 `flush()` 才把待写内容同步到数据库。

审计方法也很短，下面是完整实现：

<!-- source:app/repositories/order_repository.py:139:140 -->
源码：[app/repositories/order_repository.py](../../app/repositories/order_repository.py)，第 139–140 行。

```python
    async def write_audit(self, trace_id: str, action: str, order_id: str) -> None:
        self._session.add(AuditLogRow(trace_id=trace_id, action=action, payload=order_id))
```

`self._session.add(...)` 把新审计记录加入当前会话。此时没有单独提交。这样订单状态和审计记录可以放在同一事务中提交。

业务边界也要从代码读：这里确实检查了可取消状态，但没有调用支付平台、没有创建退款单，也没有把钱退到用户账户。不能因为订单状态变成 cancelled，就向用户承诺已支付款项已到账。

## 11. 最后是谁执行 `commit()`

下面是非流式聊天入口 `ChatService.chat()` 的完整实现：

<!-- source:app/services/chat_service.py:70:82 -->
源码：[app/services/chat_service.py](../../app/services/chat_service.py)，第 70–82 行。

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

`async with request_session(...) as db` 给这次聊天创建数据库会话。图中工具的业务写入、审计记录、幂等记录，以及这里保存的聊天消息，使用当前请求的会话。

`await db.commit()` 成功后，事务才正式提交。随后 `_remember_pending()` 保存当前响应中的提案，`return response` 才把结果返回给路由层。

可以把两次请求的事务分别理解为：

| 请求 | 事务内做什么 | 事务外的关键状态 |
| --- | --- | --- |
| 提出取消请求 | 保存本轮聊天消息；由于有 pending，不执行工具计划 | commit 后，把待确认提案保存到进程内字典 |
| 确认请求 | 修改订单、增加审计、保存幂等响应和聊天消息，最后统一 commit | 使用先前保存的提案验证确认 |

`flush()` 和 `commit()` 的区别尤其重要：前者同步待写入内容但不结束事务，后者正式提交。当前会话关闭时，未提交的事务会被回滚；如果异常传播使 `commit()` 未成功，不能把方法内部返回的 `{"status":"cancelled"}` 当作已持久化的最终结果。

这也说明了 Repository 方法为什么不各自 commit：否则可能出现“订单已改成功，审计或幂等记录失败”的部分提交。

## 12. 用一次取消操作串起所有检查

下面是代码阅读用的顺序表，不是运行日志：

| 阶段 | 本例检查或行为 | 不能跳过的原因 |
| --- | --- | --- |
| 请求解析 | 请求满足 `ChatRequest`，例如 message 非空 | 外部数据先要有合法结构 |
| Skill 与计划 | 取消 Skill 缩小工具列表，过滤不允许的工具名 | 限制模型可提出的业务能力 |
| 首次写计划 | 识别明确取消要求，生成原动作提案 | 准备向用户展示具体操作 |
| 未确认 | Executor 直接返回，不执行工具 | 不能把规划当成授权 |
| 确认验证 | action 存在、未过期、用户与会话匹配 | 绑定到此前展示的动作 |
| 恢复计划 | 只重放提案中原工具名和原参数 | 防止确认回合重新猜操作 |
| 风险权限 | `assert_permitted()` 要求确认 | 风险标签必须被代码实际检查 |
| 幂等检查 | 同一确认编号、相同指纹则复用结果 | 处理串行网络重试 |
| 输入验证 | Pydantic 验证订单号字段格式 | 格式错误不能流入业务执行 |
| 归属检查 | `order.user_id == context.user_id` | 只能处理当前用户订单 |
| 状态检查 | 仅 CREATED 或 PAID 可取消 | 已发货等订单不能通过该方法取消 |
| 数据写入 | 订单状态、审计、幂等结果写入同一会话 | 保持相关写入的一致性 |
| 事务提交 | `ChatService` 统一 commit | 方法返回不等于持久化完成 |

## 13. 读懂当前工程边界

这条链路已经把参数结构、动作确认、订单归属、状态规则、审计与幂等分成了清晰的代码层。理解它时，也要看到以下具体边界：

1. **确认记录在进程内。** 服务重启会丢失；多 worker 或多实例时，第二次请求如果到达另一个进程，可能找不到提案。Redis 已用于项目其他能力，不代表这里的 pending 已经保存进 Redis。
2. **用户身份尚需可信认证。** [ChatRequest](../../app/schemas/chat.py) 允许请求体传 `user_id`，默认 `anonymous`；[聊天路由](../../app/api/routes/chat.py) 未建立用户认证。实际产品应从经过验证的登录身份派生用户编号，并检查会话归属。不能仅靠客户端填写 `u_demo` 认定它就是订单所有者。
3. **幂等不是万能锁。** 持久化结果有助于重试，但先查后写仍有并发竞争；过期或重启丢失的确认也不会因为数据库里已有幂等结果就自动恢复。
4. **确认之前不保证已查询订单状态。** 当前 pending 分支会跳过整份工具计划，Skill 中“核对当前状态”只是步骤提示。实际归属和状态校验发生在确认后。
5. **工具注册、参数校验、用户确认、业务规则各做一件事。** 注册表决定有哪些代码可找；Pydantic 决定输入格式；确认逻辑决定是否获准执行这次动作；仓储规则决定该订单此刻能否取消。
6. **不要把取消订单解释成已退款。** 当前取消方法修改状态并写审计，没有实现支付平台退款。

继续阅读时，可以从 [Planner](../../app/agents/nodes/planner.py) 的 `planner_node()` 跟到 [Executor](../../app/agents/nodes/executor.py) 的 `executor_node()`，再进入 [CancelOrderTool](../../app/tools/aftersale_tools.py) 与 [OrderRepository](../../app/repositories/order_repository.py)。每一层都问同一个问题：“这一层验证了什么，还把什么责任交给下一层？”
