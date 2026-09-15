# 事件钩子（EventType）实测签名表

依据本机 AstrBot 4.28.0（`astrbot/core/astr_agent_hooks.py`、`core/star/star_manager.py`、
`core/platform/manager.py`、`core/core_lifecycle.py`、`core/pipeline/process_stage/method/star_request.py`）。

统一约定：钩子写在 `Star` 子类里，签名 = `(self, event, *额外参数)`。
`call_event_hook` 用 `await handler.handler(event, *args)` 调用，参数个数必须完全匹配，
否则运行期 TypeError（会在日志里以 traceback 形式出现）。

注意类归属：`astrbot/core/astr_agent_hooks.py` 的 `MainAgentHooks` 只有 4 个方法
（`on_agent_begin`、`on_agent_done`、`on_tool_start`、`on_tool_end`），`on_llm_request` 的调用点
不在这里，而在 `core/pipeline/process_stage/method/agent_sub_stages/internal.py` 与
`.../agent_sub_stages/third_party.py`。本机签名：`on_llm_request(self, event, req)`、
`on_llm_response(self, event, llm_response)`；`on_plugin_error` 为 4 个额外参数
（`plugin_name`、`handler_name`、`error`、`traceback_text`，连 `event` 共 5 参）。
照表核对，参数个数不符会在运行期报 TypeError。

## 表

| 写法 | 触发时机 | 实际参数 |
| --- | --- | --- |
| `async def on_astrbot_loaded(self)` | 全框架启动完成 | 无（只有 self） |
| `async def on_platform_loaded(self)` | 平台适配器加载后 | 无 |
| `async def on_plugin_loaded(self, metadata)` | 每个插件加载后 | `StarMetadata` |
| `async def on_plugin_unloaded(self, metadata)` | 插件卸载后 | `StarMetadata` |
| `async def on_message(self, event)`（`EventType.AdapterMessageEvent`） | 收到适配器消息，即普通消息处理器本身 | 无额外（`self, event`） |
| `async def my_tool(self, event, **kwargs)`（`EventType.OnCallingFuncToolEvent`） | LLM 调用函数工具（`@filter.llm_tool`） | 工具入参以关键字传入 |
| `async def on_waiting_llm_request(self, event)` | 即将进入 LLM 请求 | 无额外 |
| `async def on_llm_request(self, event, request)` | LLM 请求前（可改 prompt） | `ProviderRequest` |
| `async def on_llm_response(self, event, llm_response)` | LLM 返回后 | `LLMResponse` |
| `async def on_agent_begin(self, event, run_context)` | Agent 开始运行 | `ContextWrapper[AstrAgentContext]` |
| `async def on_agent_done(self, event, run_context, llm_response)` | Agent 运行完成 | `ContextWrapper`、`LLMResponse`（可能为 None） |
| `async def on_using_llm_tool(self, event, tool, tool_args)` | 调用 LLM 工具前 | `FunctionTool`、`dict \| None` |
| `async def on_llm_tool_respond(self, event, tool, tool_args, tool_result)` | 工具调用返回后 | `FunctionTool`、`dict`、`CallToolResult \| None` |
| `async def on_decorating_result(self, event)` | 回复消息前（可改/清空消息链） | 无额外 |
| `async def after_message_sent(self, event)` | 消息发送后 | 无额外 |
| `async def on_plugin_error(self, event, plugin_name, handler_name, error, traceback_text)` | 事件处理器抛异常时 | 4 个额外参数，注意版本差异 |

`EventType` 在 4.28.0 共 **16** 个成员：`OnAstrBotLoadedEvent`、`OnPlatformLoadedEvent`、
`AdapterMessageEvent`、`OnWaitingLLMRequestEvent`、`OnLLMRequestEvent`、`OnLLMResponseEvent`、
`OnAgentBeginEvent`、`OnAgentDoneEvent`、`OnDecoratingResultEvent`、`OnCallingFuncToolEvent`、
`OnUsingLLMToolEvent`、`OnLLMToolRespondEvent`、`OnAfterMessageSentEvent`、`OnPluginErrorEvent`、
`OnPluginLoadedEvent`、`OnPluginUnloadedEvent`。上表未逐一列全的成员，用法与相邻同类一致。

## 中断语义

`on_waiting_llm_request` / `on_llm_request` 的调用点写法是：

```python
if await call_event_hook(event, EventType.OnLLMRequestEvent, req):
    return   # 直接终止本次 LLM 请求
```

`call_event_hook` 在钩子执行完检查 `event.is_stopped()`，为真则返回 True。
所以**在钩子里调用 `event.stop_event()`** 才会拦截 LLM 请求；单纯 `return` 不会。
`on_llm_tool_respond` 调用前会执行 `event.clear_result()`，即工具返回值不会自动当回复发出。

## on_llm_request 常用改法

```python
async def on_llm_request(self, event: AstrMessageEvent, req: ProviderRequest):
    req.system_prompt += "\n本群禁止刷屏。"
```

## on_decorating_result 常用改法

在回复前拿到 `event.get_result()`，改 `result.chain`（`MessageChain`）或对纯文本做二次加工；
清空消息链可阻止发送。流式输出（`ResultContentType.STREAMING_RESULT`）时部分钩子行为受限，
4.28.0 会打印 “Plugins that depend on the pre-send event hook may not work…” 警告，需实测。

## 核对方式

```python
import inspect
from astrbot.core.star.star_handler import EventType
print([e.name for e in EventType])
import astrbot.core.astr_agent_hooks as h, inspect
inspect.getsource(h.MainAgentHooks)          # 看每个钩子传了几个参数
```
