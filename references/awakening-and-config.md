# 唤醒检查与配置档案（4.28.0 源码实测）

写「让 bot 在某个条件下回话」的插件时，两个地方最容易踩坑，唤醒检查和配置档案。
这份文档里的每一句都对着源码核过。

## 一、一条消息怎么才算「叫到 bot 了」

`core/pipeline/waking_check/stage.py` 的 `WakingCheckStage.process()` 依次判断：

| 条件 | 结果 |
| --- | --- |
| `event.message_str.startswith(wake_prefix)`（且首个组件不是别人的 At） | 唤醒，并把前缀从 message_str 里剥掉 |
| 消息链里有 `At(qq=self_id)` / `AtAll` / 引用（`Reply` 引用 bot 自己） | 唤醒（message_str 不动） |
| 私聊（且未开 `friend_message_needs_wake_prefix`） | 唤醒 |
| 某个插件的 handler filter 通过 | 唤醒 |

最后一条最有用。只要你的 handler 的 filter 返回 True，这条消息就算被唤醒，
后面 `process_stage` 会照常走插件处理器和 LLM。

### 坑，filter 通过不等于会进 LLM

handler filter 通过时，框架只设了：

```python
is_wake = True
event.is_wake = True          # ← 只设这个
```

但 `core/pipeline/process_stage/stage.py` 触发 LLM 的条件是：

```python
if not event._has_send_oper and event.is_at_or_wake_command and not event.call_llm:
    async for _ in self.agent_sub_stage.process(event): ...
```

也就是说，**自定义 filter 命中时必须自己把 `event.is_at_or_wake_command` 也置 True**，
否则消息虽然「算被唤醒」，插件跑完就结束了，压根不进模型。

```python
def filter(self, event, cfg) -> bool:
    if not 命中:
        return False
    event.is_wake = True
    event.is_at_or_wake_command = True   # ← 少了这行就不进 LLM
    return True
```

## 二、自定义 filter（不重启就能改规则）

`filter.regex` 的正则在 import 时就编译写死了，想读配置就得重启。
要按配置动态判断，用 `CustomFilter` 子类：

```python
from astrbot.api.event.filter import CustomFilter   # 注意：不是 astrbot.api.event

class MyFilter(CustomFilter):
    def __init__(self, raise_error: bool = True, **kwargs):
        super().__init__(raise_error=raise_error)

    def filter(self, event, cfg) -> bool:   # cfg 是 AstrBot 全局配置
        ...

class MyPlugin(Star):
    @filter.custom_filter(MyFilter)        # 传类，不要传实例
    async def hook_by_filter(self, event: AstrMessageEvent):
        return                              # 只负责「算被唤醒」，不产生回复
```

要点：

- 装饰器传类，框架会自己 `MyFilter(raise_error)` 实例化。
- `filter(event, cfg)` 里的 `cfg` 是 AstrBot 全局配置，拿不到插件自己的 `self.config`；
  需要插件配置就存一份模块级变量，在 `__init__` 里刷新。
- filter 在唤醒检查阶段被调用，早于 `ProcessStage`，所以是「决定这条消息要不要处理」的
  唯一早期入口。
- handler 本身可以空实现（`return`），它存在只是为了让 filter 被注册。

## 三、配置档案（多 bot 各用各的唤醒词）

唤醒词不是全局的，是配置档案级的。相关代码：

- `core/astrbot_config_mgr.py`：`get_conf(umo)` 把 umo 映射到某份 `abconf_*.json`，
  映射不到就落到默认档案（`cmd_config.json`）。
- `core/umop_config_router.py`：路由表存在 `preferences` 表的
  `key='umop_config_routing'`，形如：

```json
{"bot-a:*:*": "3f2c9a18-...", "bot-b:*:*": "a71d4e55-..."}
```

`*` 是通配（按 `[platform_id]:[message_type]:[session_id]` 三段匹配）。

### 由此得到的两条结论

1. 一个 AstrBot 挂多个 bot 时，若它们绑同一份档案（或都没绑），就共用一个 `wake_prefix`
   ，一个唤醒词会同时唤醒所有 bot。分开的办法是给每个 bot 的 `平台实例名:*:*`
   绑各自的档案，档案里写自己的 `wake_prefix`。这是配置问题，不是插件问题。
2. 插件里要「按 bot 取配置」就用 `self.context.get_config(umo=event.unified_msg_origin)`
   （`Context.get_config(umo)`），别直接读 `data/cmd_config.json`。

```python
def session_conf(self, event):
    get = getattr(self.context, "get_config", None)
    if not callable(get):
        return {}
    try:
        return get(umo=event.unified_msg_origin) or {}
    except Exception:
        return {}
```

## 四、群聊上下文会把标记塞进提示词

`builtin_stars/astrbot/group_chat_context.py` 会把群消息渲染成这样交给模型：

```
[昵称/14:11:57]: ⚠️[DIRECTED AT YOU]  [At: 名字] 这句话有几个字
[别人的昵称/14:11:58]: [Quote(某人: 原话)] [At: 昵称(qq)] …
```

- `⚠️[DIRECTED AT YOU]`：这条 @ 了 bot（`comp.qq in (self_id, "all")` 时插到第 1 位）
- `[At: 名字]`：At 组件
- `[Quote(某人: …)]`：引用
- `[Shared Card…]`、`[图片]`、`[表情:id]`、`[语音]` 等组件占位

写「数一下这句话有几个字」「提取原话」这类插件时，要意识到这些标记也在提示词里，
否则模型会连标记一起数、把 At 的名字当成句子内容。补 @ 时把它写成 `@名字 `（带 @ 和空格），
比写成裸名字安全得多。

## 五、平台实例上没有 `self_id`

`AiocqhttpAdapter` 的实例没有 `self_id` 属性，`self_id` 只在消息事件上
（`event.self_id` / `event.get_self_id()`）。

要枚举「本机有哪些 bot 账号」：

```python
for inst in self.context.platform_manager.get_insts():
    sid = getattr(inst, "self_id", "")           # 有的实现有
    clients = getattr(getattr(inst, "bot", None), "_wsr_api_clients", None)
    if isinstance(clients, dict):                # aiocqhttp 反向 WS：连上来的号都在这里
        ids |= {str(k) for k in clients}
```

这三条路都要留着兜底。平台实例 `self_id`（不一定有）→ `_wsr_api_clients`（连上才有）
→ 运行中见过的 `event.get_self_id()`。并且要在每次判定时重算，不能只在 `__init__` 里算一次。
插件加载时平台往往还没连上。

## 六、插件自己的数据放哪

| 用途 | 路径 |
| --- | --- |
| 插件私有数据（自己记的库、缓存、日志） | `data/plugin_data/<插件名>/` |
| 用户可见配置 | `data/config/<插件名>_config.json`（由 `_conf_schema.json` 生成，别手写） |
| AstrBot 全局配置 | `data/cmd_config.json`（不要动） |
| 配置档案 | `data/config/abconf_<uuid>.json` |

插件里取数据根目录用 AstrBot 提供的工具函数，别拼死路径。

## 七、SQLite 的两个连带坑

长时间跑的插件写库时，这两条建议默认加上：

```python
conn = sqlite3.connect(path, timeout=10)
conn.execute("pragma busy_timeout = 10000")   # 撞上别人的写事务先等，别立刻抛
conn.execute("pragma temp_store = memory")    # 排序/去重不走临时文件
```

原因在于 `unable to open database file` 这句话既可能是库打不开，也可能是临时文件建不出来
（TEMP 指向不存在的目录、或目录不可写），两种原因同一个措辞，非常容易误判成「库坏了」。
真出问题时把库路径写进日志，否则根本不知道是哪个 bot 的库。
`logger.warning(f"[X] 检索失败（库={path}）: {e}")`。
