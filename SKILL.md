---
name: astrbot-plugin-dev
description: "Develop, modify, and review AstrBot plugins against the local AstrBot source tree (currently 4.28.0): commands and event filters, event hooks, LLM tools, _conf_schema.json, metadata.yaml packaging, Star KV storage, session waiters, Web APIs and image rendering. Use when the user asks to create a new plugin, add commands/hooks/LLM tools, fix a plugin that fails to load, or verify plugin APIs instead of relying on third-party docs."
---

# AstrBot 插件开发（以本机源码为准）

**本 Skill 的唯一依据是本机安装的 AstrBot 源码（4.28.0）。** 文中每个类名、字段名、函数签名
都取自 `site-packages/astrbot/` 的实际代码。动笔前先用 `inspect.signature` / `getsource` 复核一遍，
版本升级后旧写法会静默失效。

## 铁律

1. 先探测版本与源码路径，再动笔；不确定的 API 一律 `inspect` 或读源码，禁止凭记忆。
2. 不改动与本任务无关的文件；不动 `data/cmd_config.json`、`data/config/` 等运行期配置。
3. 写用户真机插件目录（`data/plugins/`）前先确认目标；有疑问就问，不要静默覆盖已有插件。
4. 交付前必须真正加载验证（见「自检」），只报“应该能用”不算完成。

## 步骤

### 1. 探测环境

```powershell
& "<AstrBot>\.venv\Scripts\python.exe" -c "import astrbot; print(astrbot.__file__)"
```

源码根即该路径所在目录（形如 `<AstrBot>\.venv\Lib\site-packages\astrbot\`），
用户插件目录为 `<AstrBot>\data\plugins\`。`<AstrBot>` 指 AstrBot 安装根目录，
在 Skill 运行环境中由工作区与 AstrBot 配置确定，不要写死成某个人的盘符。

校验任意 API：

```python
import inspect
from astrbot.api.star import Context, Star
print(inspect.signature(Context.register_task))
print([f for f in dir(Star) if not f.startswith('_')])
```

### 2. 确认插件骨架与打包文件

看 [references/plugin-package.md](references/plugin-package.md)：目录结构、`metadata.yaml` 字段、
`_conf_schema.json` 各类型写法、`main.py` 注册模板、版本要求写法。

### 3. 写命令与消息处理

看 [references/filters-and-events.md](references/filters-and-events.md)：`@filter.command` /
`command_group` / `regex` / 权限与平台过滤、`AstrMessageEvent` 接口、消息链组件。

### 4. 挂事件钩子

看 [references/hooks.md](references/hooks.md)：**每个钩子的实际参数个数**（例如 `on_plugin_error`
在 `event` 之外还有 4 个参数，很容易写错）、如何用 `event.stop_event()` 拦截 LLM 请求。

### 5. 让 bot 在某种条件下回话（唤醒 / 配置档案）

看 [references/awakening-and-config.md](references/awakening-and-config.md)：唤醒检查的 4 个条件、
**自定义 filter 命中后必须自己置 `is_at_or_wake_command`（否则不进 LLM）**、
配置档案（`abconf_*.json` + `umop_config_routing`，多 bot 各用各的唤醒词靠它）、
群聊上下文注入的 `[At: …]` / `⚠️[DIRECTED AT YOU]` 标记、平台实例上拿不到 `self_id`。

### 6. 用服务能力

看 [references/services.md](references/services.md)：LLM 工具、KV 存储、会话等待器、
主动发消息、后台任务、i18n 与 `pages/`、`logo.png` 约定。

## 常见故障定位

| 现象 | 优先排查 |
| --- | --- |
| 插件不出现 | 目录下有没有 `main.py`（或与目录同名的 `.py`）、`metadata.yaml` 是否解析失败 |
| 加载即报错 | 顶层 import 写错（如 `from astrbot.api.provider import FunctionTool`）、`_conf_schema.json` JSON 语法错 |
| 命令无响应 | 处理器签名是否为 `(self, event)`；是否被 Dashboard 命令配置屏蔽；是否需要唤醒前缀 |
| 钩子报 TypeError | 钩子参数个数与 `call_event_hook` 传参一致，对照 hooks.md 表格 |
| LLM 工具不被调用 | docstring 的 `Args:` 格式、参数类型是否在 string/number/object/array/boolean 内 |
| 自定义 filter 命中了但 bot 不回 | 少了 `event.is_at_or_wake_command = True`（见 awakening-and-config.md 第一节） |
| 一个唤醒词把所有 bot 都叫醒了 | 多个 bot 绑了同一份配置档案；按 `平台实例名:*:*` 分开绑，与插件无关 |
| 报 `unable to open database file` | 未必是库坏了：可能是 TEMP 目录不可写导致临时文件建不出来，加 `pragma temp_store = memory` 并把库路径打进日志 |

## 自检

1. 目录/文件齐全：`main.py`、`metadata.yaml`（版本、`astrbot_version`）、按需 `_conf_schema.json`、
   `requirements.txt`、`logo.png`、`README.md`。
2. 语法与导入：`python -m py_compile main.py`；再 import 一次确认无循环依赖。
3. 事件钩子参数个数逐个对照 [references/hooks.md](references/hooks.md)。
4. 让插件真实加载后读日志确认无 traceback；有报错就按上表定位，不许含糊带过。
