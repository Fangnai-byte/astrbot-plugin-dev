# 服务能力：LLM 工具、KV、等待器、Web API、渲染

路径依据本机 4.28.0 实测：

- 顶层导出（实测 `dir(astrbot.api)`）：`AstrBotConfig`、`FunctionTool`、`ToolSet`、`llm_tool`、
  `logger`、`html_renderer`、`BaseFunctionToolExecutor`。即 `from astrbot.api import FunctionTool`
  可用，且与 `astrbot.core.agent.tool.FunctionTool` 是同一个类。
- 工具定义：`astrbot/core/agent/tool.py` → `class FunctionTool`，字段为
  `name`、`description`、`parameters`、`handler`、`handler_module_path`、`active`、`is_background_task`。
- 工具管理：`astrbot/core/provider/func_tool_manager.py`（`FuncToolManager`）。
- LLM 请求对象：`astrbot/core/provider/entities.py` 的 `ProviderRequest` / `LLMResponse`。

## 1) 注册 LLM 工具

```python
from astrbot.api.event import filter, AstrMessageEvent
from astrbot.api import logger

class MyPlugin(Star):
    @filter.llm_tool(name="get_weather")
    async def get_weather(self, event: AstrMessageEvent, city: str):
        """查询指定城市天气。

        Args:
            city(string): 城市名，例如 北京
        """
        return f"{city} 今天晴。"
```

- docstring 的 `Args:` 会被解析成工具参数描述，类型只支持 `string`/`number`/`object`/`array`/`boolean`。
- 返回值是字符串时会回灌给 LLM 继续对话；返回 `None` 则不回灌。
- 需要在代码里动态增删工具时用 `self.context.get_llm_tool_manager()` 取管理器再操作，
  或直接构造 `FunctionTool(handler=..., handler_module_path=...)`（字段以实测为准）。

## 2) 主动调用 LLM

```python
from astrbot.api.provider import ProviderRequest      # 需核对导入路径
llm_resp = await self.context.get_using_provider(event.unified_msg_origin).text_chat(
    prompt="...", session_id=event.unified_msg_origin
)
```

多轮/带工具请用 agent 运行器（`astrbot/core/agent/`），参数以源码为准；
这里不要凭记忆写，先 `inspect.signature(provider.text_chat)`。

## 3) Star KV 存储

```python
await self.put_kv_data("key", {"a": 1})
data = await self.get_kv_data("key", None)
await self.delete_kv_data("key")
```

签名实测：`put_kv_data(self, key: str, value: int|float|str|bytes|bool|dict|list|None)`、
`get_kv_data(self, key, default)`。值支持 dict/list 等，可直接存复杂结构。
键建议拼上 `event.unified_msg_origin` 以避免多群串数据。

## 4) 会话等待器（追问用户）

```python
from astrbot.core.utils.session_waiter import session_waiter, SessionController
```

在命令里 `@session_waiter(timeout=30)` 修饰一个接收 `(controller, event)` 的内部函数，
用户下一条消息会进入该函数，超时走 `controller.stop()` 分支。装饰器参数与用法在
`astrbot/core/utils/session_waiter.py` 中确认后再写。

## 5) 发消息与后台任务

```python
await self.context.send_message(event.unified_msg_origin, MessageChain([Plain("定时提醒")]))
self.context.register_task(asyncio.create_task(coro), "每日推送")   # 签名: (task: Awaitable, desc: str)
```

`Context` 其他实测接口：`get_platform_inst(platform_id)`、`get_db()`、
`register_commands(star_name, command_name, desc, priority, awaitable, use_regex=False, ignore_prefix=False)`。
Web API 注册（`register_web_api`）与 `text_to_image` / `html_render` 请以 `dir(Context)` 核对参数后使用。

## 6) i18n 与 Dashboard 页面

- 翻译文件放在插件目录 `.astrbot-plugin/i18n/<lang>.json`，由 `PluginManager._load_plugin_i18n` 读取。
- 自定义 Web 页面放在插件 `pages/` 目录，并在 `metadata.yaml` 里用 `pages:` 列表声明。
- 插件 logo 固定文件名 `logo.png`（`StarManager` 中 `self.logo_fname = "logo.png"`）。
