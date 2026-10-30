# 过滤器、事件对象与消息链

依据本机 AstrBot 4.28.0：`astrbot/api/event/__init__.py`、`astrbot/api/message_components.py`、
`astrbot/core/star/register/star_handler.py`。

## 导入

```python
from astrbot.api.event import filter, AstrMessageEvent
from astrbot.api.message_components import Plain, Image, At, Reply, File
from astrbot.api import logger
```

## 常用过滤器装饰器

```python
@filter.command("hello", alias={"hi"}, priority=1, desc="打招呼")
async def hello(self, event: AstrMessageEvent):
    yield event.plain_result("你好")

@filter.command_group("admin")
def admin(): pass

@admin.command("kick")            # 子命令 admin kick
async def kick(self, event: AstrMessageEvent): ...

@filter.regex(r"^查\s*(.+)$")      # 正则触发，命令名由配置 alter_cmd 管理
@filter.event_message_type(filter.EventMessageType.GROUP_MESSAGE)
@filter.platform_adapter_type(filter.PlatformAdapterType.AIOCQHTTP)
@filter.permission_type(filter.PermissionType.ADMIN)
```

签名实测是 `filter.command(command_name=None, sub_command=None, alias: set | None = None, **kwargs)`，
`filter.regex(regex, **kwargs)`，`filter.llm_tool(name=None, **kwargs)`。
除 `desc` 被单独取作描述外，其余 kwargs 会整体存进 `handler.extras_configs`，
`priority` 就在里面（默认 0，数字越大越先执行，见 `core/star/star_handler.py` 的排序逻辑）。
所以 `priority=1`、`desc="..."` 可以直接写在装饰器里。

- 处理器可以是 `async def` 生成器（`yield` 结果）或普通 `async def`（用 `event.send()` 主动发送）。
- `CommandFilter` 构造是 `(command_name, alias=None, handler_md=None, parent_command_names=None)`，
  `handler_params` 只取第 3 个往后的参数（前两个固定为 `self` 和 `event`）。
- 命令是否会进 LLM、是否需要唤醒前缀，由 Dashboard 命令配置（`alter_cmd`）决定，不在代码里硬编。

## AstrMessageEvent 常用接口（调用前用 `dir()` 核对）

| 成员 | 说明 |
| --- | --- |
| `event.message_str` | 纯文本内容 |
| `event.message_obj` | 原始消息对象（含 `message`、`sender`、`group_id`、`self_id`） |
| `event.unified_msg_origin` | 会话唯一 ID，`平台:消息类型:会话ID`，用于主动发消息与 KV 键 |
| `event.get_sender_id()` / `get_sender_name()` | 发送者信息 |
| `event.get_group_id()` | 群号（私聊为空） |
| `event.get_self_id()` | 机器人自身 ID |
| `event.is_admin()` | 是否管理员 |
| `event.plain_result(text)` | 构造纯文本结果 |
| `event.chain_result([...])` | 构造消息链结果 |
| `event.get_result()` / `event.clear_result()` | 结果装饰阶段读写 |
| `event.stop_event()` | 终止后续流程（钩子里可拦截 LLM 请求） |
| `await event.send(chain)` | 立即发送一条消息 |
| `event.set_extra(k, v)` / `get_extra(k, default)` | 事件内传参 |

注意 `message_str` 与 `message_obj` 是 `AstrMessageEvent.__init__` 里赋的实例属性，不在类上，
用 `hasattr(AstrMessageEvent, "message_str")` 会返回 False，须在实例（`event.`）上访问。

`ResultContentType`（`astrbot/api/event/result_content_type.py` 附近）区分普通结果与 `STREAMING_RESULT`。

## 消息链组件

`astrbot/api/message_components.py` 导出的组件有 `Plain`、`Image`、`At`、`AtAll`、`Face`、`Record`、
`Video`、`File`、`Forward`、`Node`、`Nodes`、`Reply`、`Poke`、`Json`、`Location`、`Music`、
`Dice`、`RPS`、`Shake`、`Share`、`Contact`、`Unknown`
（同模块还可见 `BaseMessageComponent`、`ComponentType`、`ComponentTypes`、`MediaResolver` 等基类/辅助名）。
注意没有 `TTS`，旧文档常见此笔误。

```python
yield event.chain_result([
    At(qq=event.get_sender_id()),
    Plain(" 你的文件："),
    File(name="a.txt", file="file:///abs/path/a.txt"),
])
```

构造参数随版本变动，4.28.0 的 `Image` 工厂方法为 `fromBase64` / `fromBytes` / `fromFileSystem` /
`fromIO` / `fromURL`，没有 `fromLocalFile`。写之前用 `help(Image)` 或读源码确认。

## 配置注入

```python
@register("my_plugin", "author", "desc", "v1.0.0", "repo_url")
class MyPlugin(Star):
    def __init__(self, context: Context, config: AstrBotConfig):
        super().__init__(context)
        self.config = config          # 与 _conf_schema.json 对应
```

运行期也可 `self.context.get_config(umo=event.unified_msg_origin)` 取会话级配置。
