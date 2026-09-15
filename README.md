# astrbot-plugin-dev

AstrBot 插件开发参考 Skill（**以本机安装的 AstrBot 源码为唯一依据**，当前核对版本 **4.28.0**）。

写这个 Skill 的起因：网上流传的“AI 写插件知识库”普遍停留在旧版本，照抄会直接写出跑不起来的插件。
例如把 `FunctionTool` 写成在 `astrbot.api.provider` 且字段名为 `func`，而 4.28.0 实际在
`astrbot.core.agent.tool`，字段为 `handler` / `handler_module_path` / `active` / `is_background_task`。
本仓库里每个类名、字段名、函数签名都取自 `site-packages/astrbot/` 的实际代码，并逐条 `inspect`
或读源码复核过；拿不准的地方宁可写“先 inspect 再动笔”。

## 内容结构

```
SKILL.md                          总览、铁律、开发步骤、常见故障定位、自检流程
references/
├── plugin-package.md             目录结构、metadata.yaml、_conf_schema.json、main.py 模板
├── filters-and-events.md         命令/正则/权限过滤、AstrMessageEvent 接口、消息链组件
├── hooks.md                      各事件钩子的实际参数个数与调用点
└── services.md                   LLM 工具、KV 存储、会话等待器、主动发消息、i18n、pages/logo
```

## 安装

把整个目录放进 AstrBot 的 skills 目录：

```
<AstrBot>/data/skills/astrbot-plugin-dev/
```

放到该位置后，AstrBot 会把它作为一个 Skill 提供给模型；也可以让它作为参考资料直接阅读。

## 覆盖范围

- 命令与事件过滤器（`@filter.command` / `command_group` / `regex` / 权限与平台过滤）
- 事件钩子（`EventType` 全部成员、`call_event_hook` 的实参个数、`event.stop_event()` 拦截语义）
- LLM 工具（`@filter.llm_tool` 的注册点与调用点、入参传递方式）
- 打包与配置（`metadata.yaml`、`_conf_schema.json`、`logo.png`、i18n、`pages/`）
- 运行时能力（Star KV、会话等待器、后台任务、主动发消息、Web API、图片渲染）

## 使用注意

- **版本敏感**：本 Skill 核对的是 4.28.0。AstrBot 升级后请以线上源码为准，动笔前先
  `inspect.signature` / `getsource` 复核，旧写法会静默失效。
- **不以第三方文档为准**：本仓库刻意不引用任何第三方教程，避免把过期写法带进来。
- 所有示例路径都用 `<AstrBot>` 占位，指 AstrBot 安装根目录，不绑定具体机器。

## 许可

MIT License，见 [LICENSE](LICENSE)。
