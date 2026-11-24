# 从基础语法到完整问答：源码精读

这组文档根据当前工作区源码编写，面向能接触 Python、但尚不熟悉后端和 RAG 的读者。主例是“退货政策是什么？”，业务工具例子是“帮我取消 ORD10003”。

代码块标为源码的，来自仓库；部分是明确标注范围的节选。标为“教学示意”的数据和展开写法只帮助理解，不代表运行结果。没有调用真实模型、执行订单变更或重新评测。

## 阅读路线

1. [Python 语法、共享状态和工作流](00-python-and-graph.md)：先知道函数、对象、字典、类型、异步、节点和边是什么。
2. [启动与对话主流程](01-startup-and-chat.md)：从应用启动，追踪 HTTP 请求怎样变成最终响应。
3. [从文档到检索证据](02-retrieval.md)：切块、BM25、向量、改写、RRF、精排、多轮补检。
4. [工具调用与人工确认](03-tools-and-confirmation.md)：从取消订单看参数校验、权限、确认、幂等与事务。
5. [生成、校验、记忆和流式输出](04-generation-and-memory.md)：证据如何进入模型，答案如何被检查，历史消息实际上怎样处理。

第一次阅读可先读 00、01，再读 02 中的 `retrieval_node`，最后读 04 的 `generate_node`。工具分支可以单独阅读。

## 先沿一条实际调用链走

```text
HTTP POST /api/v1/chat
  → API 的 chat(payload, request, container)
  → ChatService.chat(request, trace_id=...)
  → _initial_state(...)：准备 state
  → _invoke(state)：调用编译好的 LangGraph
      → apply_guardrail
      → classify_intent
      → route_complexity
      → SINGLE / MULTI：retrieve_docs
      → AGENT：plan_tasks → execute_tools
      → DIRECT：直接进入 generate_answer
      → generate_answer → check_faithfulness → compact_memory
      → REFUSE：refuse_answer → compact_memory
  → _to_response(...)：整理字段
  → _persist(...) 与 db.commit()：保存并提交
  → 返回 ChatResponse，FastAPI 序列化为 JSON
```

这张图是正常控制流的概括。护栏可以提前进入拒答；错误也可能在不同阶段抛出。`DIRECT` 虽经过 `check_faithfulness` 节点，该节点内部会直接跳过模型检查。

## 三个经常混淆的概念

| 概念 | 这个项目里的具体含义 |
|---|---|
| 会话 session_id | 把用户的多轮聊天归为一组 |
| 数据库 session | 一轮处理过程中访问数据库、管理事务的对象 |
| state | 在 LangGraph 各节点间传递的本轮工作数据 |

`session_id` 是字符串，数据库 session 是对象，`state` 是带类型说明的字典。它们不能混为一谈。

## 用源码判断能力是否真正接通

- 有 `messages` 字段，不自动意味着生成时使用了历史。需要继续看 `llm.generate(messages)` 中的 `messages` 到底从哪里来。
- 定义了工具风险级别，不自动意味着已拦截。需要找到执行器的权限判断。
- 声明了模型名称，不自动意味着权重加载成功。应看 provider 的选择、加载和降级路径。
- 有 `flush()` 不表示事务已经提交。最终提交由 `ChatService` 的 `commit()` 完成。
- 示例里的改写、排序和答案仅用于说明数据格式；本组文档没有运行模型验证这些结果。

## 自检

读完后，尝试用自己的话回答：

1. API 的 `chat` 为什么没有直接调用模型？
2. 节点返回 `{"retrieved_chunks": chunks}` 后，下一个节点怎样读到这些数据？
3. 为什么 BM25 和向量检索后，还需要 RRF 与精排？
4. 用户说“确认”之后，为什么执行的是保存的动作，而不是重新让模型决定？
5. 聊天消息已保存到数据库，为什么仍可能没有真正的多轮上下文？

对应答案分别在 01、00、02、03、04。先找到实际赋值和调用，再解释设计目的。
