# 插件包结构与配置

依据本机 4.28.0 `astrbot/core/star/star_manager.py`、`astrbot/core/star/star.py`(StarMetadata) 实测。

## 目录

```
data/plugins/my_plugin/
├── main.py                # 必需；或与目录同名的 my_plugin.py
├── metadata.yaml           # 必需（也接受 metadata.yml）
├── _conf_schema.json       # 可选，Dashboard 配置项
├── requirements.txt        # 可选，缺失依赖由框架提示安装
├── logo.png                # 可选，固定文件名 logo.png
├── README.md               # 可选，Dashboard 插件页展示
├── pages/                  # 可选，自定义 Web 页面（在 metadata.yaml 的 pages 里声明）
└── .astrbot-plugin/i18n/zh_CN.json   # 可选，i18n
```

`main.py` 与同名 `.py` 都没有时框架直接跳过该目录并打日志。

## metadata.yaml 字段

`StarMetadata` 的 dataclass 字段：`name`、`author`、`desc`、`short_desc`、`version`、`repo`、
`display_name`、`support_platforms`、`astrbot_version`、`pages`、`i18n`、`logo_path`（后两个加载时填充）。

```yaml
name: my_plugin
display_name: 我的插件
desc: 完整描述，会显示在插件列表
short_desc: 一句话简介
version: v1.0.0
author: your-github-name
repo: https://github.com/你的账号/my_plugin
astrbot_version: ">=4.0.0"
support_platforms:
  - aiocqhttp
  - telegram
```

- `astrbot_version` 是版本范围字符串，加载时会校验；写错会拒绝加载。
- `support_platforms` 必须是列表，否则不生效。
- 缺少 `metadata.yaml` 时框架报“无法获取插件目录名”。

## main.py 模板

```python
from astrbot.api import logger
from astrbot.api.event import filter, AstrMessageEvent
from astrbot.api import AstrBotConfig
from astrbot.api.star import Context, Star, register

@register("my_plugin", "your-github-name", "插件描述", "v1.0.0", "repo_url")
class MyPlugin(Star):
    def __init__(self, context: Context, config: AstrBotConfig):
        super().__init__(context)
        self.config = config

    async def initialize(self):
        logger.info("my_plugin 已加载")

    @filter.command("ping")
    async def ping(self, event: AstrMessageEvent):
        yield event.plain_result("pong")

    async def terminate(self):
        logger.info("my_plugin 已卸载")
```

- `initialize()` / `terminate()` 可选，由加载器检查后调用。
- 未用到的名字别 import，`py_compile` 与加载都会更干净。

## _conf_schema.json 写法

顶层是「字段名 → 描述对象」，`type` 决定 Dashboard 控件：

```json
{
  "enable_feature": {
    "description": "是否启用该功能",
    "type": "bool",
    "default": true
  },
  "reply_text": {
    "description": "回复内容，支持 {name} 占位",
    "type": "string",
    "default": "你好，{name}"
  },
  "interval": {
    "description": "检查间隔（秒）",
    "type": "int",
    "default": 60
  },
  "mode": {
    "description": "运行模式",
    "type": "string",
    "options": ["safe", "fast"],
    "default": "safe"
  },
  "groups": {
    "description": "生效群号",
    "type": "list",
    "default": []
  },
  "advanced": {
    "description": "高级设置",
    "type": "object",
    "items": {
      "timeout": { "description": "超时", "type": "int", "default": 5 },
      "debug": { "description": "调试日志", "type": "bool", "default": false }
    }
  }
}
```

常用 `type`：`bool`、`string`、`text`、`int`、`float`、`list`、`object`，另有 `file` 与
`template_list`。没有 `dict` 这个类型（想存键值对写 `object`），写成 `dict` 后端不会按对象校验。
JSON 语法错误会导致配置整体失效，改完用 `python -m json.tool _conf_schema.json` 校验。

## 交付前自检

```powershell
& "<AstrBot>\.venv\Scripts\python.exe" -m py_compile <AstrBot>\data\plugins\my_plugin\main.py
& "<AstrBot>\.venv\Scripts\python.exe" -m json.tool <AstrBot>\data\plugins\my_plugin\_conf_schema.json
```
