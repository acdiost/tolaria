---
type: Note
category: "[[development-and-ai-tools]]"
tags:
  - mcp
  - ai
  - protocol
  - tutorial
  - python
---

# 从零实现一个 MCP Server：原理到实战（Python 版）

> Model Context Protocol（MCP）是 Anthropic 于 2024 年底提出的开放协议，目标是统一 AI 模型与外部工具/数据源的集成方式。本文以 Python 为主，从协议原理出发，手写一个带完整注解的 MCP Server，不依赖任何 SDK。

---

## 一、MCP 是什么，为什么需要它

在 MCP 出现之前，每个 AI 应用都要自己设计"工具调用"格式：OpenAI 有 function calling、Anthropic 有 tool use、各家 Agent 框架各有一套。这造成了大量重复的胶水代码，工具无法跨模型复用。

MCP 的核心思想很简单：**把 AI 能调用的能力（工具、数据、提示词模板）抽象成一套标准接口，让 Server 和 Client 可以独立演化。**

```
┌─────────────┐        MCP Protocol        ┌─────────────────┐
│  MCP Client │ ◄────────────────────────► │   MCP Server    │
│ (Claude/IDE)│    JSON-RPC 2.0 over       │ (你自己写的服务)  │
└─────────────┘    stdio / SSE / HTTP      └─────────────────┘
```

一个 MCP Server 可以暴露三类能力：

| 能力 | 说明 | 典型用途 |
|------|------|----------|
| **Tools** | 模型可主动调用的函数 | 查数据库、发请求、执行代码 |
| **Resources** | 模型可读取的数据源 | 文件内容、API 响应、数据库记录 |
| **Prompts** | 可复用的提示词模板 | 系统提示、few-shot 示例 |

---

## 二、协议层解析

### 2.1 传输层

MCP 支持三种传输方式：

**stdio（最常用）**：Client 以子进程方式启动 Server，通过标准输入/输出通信。Claude Desktop 使用此方式。

```
Client                    Server (子进程)
  │  stdin ──────────────►  │
  │  stdout ◄─────────────  │
```

**HTTP + SSE**：Server 作为独立 HTTP 服务运行，Client 通过 SSE 接收流式响应。适合远程部署。

**Streamable HTTP**（新版）：统一用 HTTP POST，响应可以是普通 JSON 或 SSE 流。

### 2.2 消息格式：JSON-RPC 2.0

所有消息都是 JSON-RPC 2.0 格式，每条消息用换行符分隔（NDJSON）。

**请求（Client → Server）**：
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "city": "Beijing" }
  }
}
```

**响应（Server → Client）**：
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      { "type": "text", "text": "北京今天晴，25°C" }
    ]
  }
}
```

**通知（无需响应的单向消息）**：
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/progress",
  "params": { "progressToken": "abc", "progress": 50 }
}
```

### 2.3 生命周期

```
Client                         Server
  │                               │
  │──── initialize ──────────────►│  协商版本和能力
  │◄─── initialize result ────────│
  │──── initialized (通知) ───────►│  握手完成（通知无需响应）
  │                               │
  │──── tools/list ──────────────►│  发现可用工具
  │◄─── tools/list result ────────│
  │                               │
  │──── tools/call ──────────────►│  调用工具
  │◄─── tools/call result ────────│
  │                               │
```

握手阶段是必须的：Client 先发 `initialize` 协商协议版本，Server 返回自身能力声明，Client 再发 `initialized` 通知表示就绪，之后才能正常通信。

---

## 三、Python 实现：项目结构

```
my_mcp_server/
├── transport.py    # 传输层：负责读写 stdio
├── server.py       # 核心框架：消息分发、工具注册
├── tools.py        # 工具定义：业务逻辑
└── main.py         # 入口：组装并启动
```

只依赖 Python 标准库，无需任何第三方包。

---

## 四、传输层（transport.py）

```python
# transport.py
#
# 传输层职责单一：把字节流切割成 JSON 消息，并把 Python 对象序列化写回 stdout。
# stdio 传输的关键约束：
#   - stdout 只能写协议消息（JSON），所有日志必须走 stderr
#   - 每条消息以 \n 结尾，即 NDJSON（Newline-Delimited JSON）格式
#   - 必须立即 flush，否则消息会卡在缓冲区里，Client 永远收不到响应

import sys
import json
from typing import Any


class StdioTransport:
    def send(self, obj: Any) -> None:
        """把一个 Python 对象序列化为 JSON 并写入 stdout。"""
        # ensure_ascii=False 保证中文等非 ASCII 字符不被转义为 \uXXXX
        line = json.dumps(obj, ensure_ascii=False)
        # 写到 stdout.buffer（bytes 级别）而非 stdout（text 级别），
        # 避免 Windows 上的 CRLF 转换问题。
        sys.stdout.buffer.write((line + "\n").encode("utf-8"))
        sys.stdout.buffer.flush()

    def receive_lines(self):
        """生成器：从 stdin 逐行读取，跳过空行，yield 解析后的 dict。"""
        # sys.stdin.buffer 以字节模式读取，保证 UTF-8 解码由我们自己控制，
        # 不受系统 locale 影响（Windows 默认 GBK 会导致中文乱码）。
        for raw in sys.stdin.buffer:
            line = raw.decode("utf-8").strip()
            if not line:
                continue  # 跳过心跳空行
            try:
                yield json.loads(line)
            except json.JSONDecodeError as e:
                # 收到非法 JSON 不应崩溃，写 stderr 记录即可
                sys.stderr.write(f"[transport] JSON decode error: {e}\n")
                sys.stderr.flush()
```

---

## 五、Server 核心框架（server.py）

```python
# server.py
#
# McpServer 是整个服务的骨架：
#   1. 维护已注册的工具、资源、提示词
#   2. 接收每条消息，根据 method 分发到对应处理函数
#   3. 把处理结果或错误包装成 JSON-RPC 响应发回给 Client

import sys
import json
from dataclasses import dataclass, field
from typing import Any, Callable, Awaitable
from transport import StdioTransport


# ──────────────────────────────────────────────
# 数据类：工具定义
# ──────────────────────────────────────────────

@dataclass
class ToolDefinition:
    name: str                    # 工具名称，模型调用时用这个字符串指定
    description: str             # 工具功能描述，模型根据此决定何时调用
    input_schema: dict           # 参数的 JSON Schema，必须包含 "type": "object"
    handler: Callable            # 实际执行函数，接收 dict 参数，返回 str


# ──────────────────────────────────────────────
# 数据类：Resource 定义
# ──────────────────────────────────────────────

@dataclass
class ResourceDefinition:
    uri: str                     # 资源唯一标识符，格式自定义（如 file:///path 或 db://table）
    name: str                    # 人类可读名称
    description: str
    mime_type: str               # 内容类型，帮助 Client 决定如何展示
    reader: Callable             # 读取函数，返回资源的文本内容


# ──────────────────────────────────────────────
# 核心类：McpServer
# ──────────────────────────────────────────────

class McpServer:
    def __init__(self, name: str, version: str):
        self.name = name
        self.version = version
        self.transport = StdioTransport()

        # 用 dict 存储注册项，name/uri 作为 key，O(1) 查找
        self._tools: dict[str, ToolDefinition] = {}
        self._resources: dict[str, ResourceDefinition] = {}
        # Prompts 结构相似，此处简化为列表（不需要按 name 快速查找时够用）
        self._prompts: list[dict] = []

    # ── 注册 API ────────────────────────────────

    def register_tool(self, tool: ToolDefinition) -> "McpServer":
        """注册一个工具。支持链式调用：server.register_tool(a).register_tool(b)"""
        self._tools[tool.name] = tool
        return self

    def register_resource(self, resource: ResourceDefinition) -> "McpServer":
        self._resources[resource.uri] = resource
        return self

    def register_prompt(self, prompt: dict) -> "McpServer":
        self._prompts.append(prompt)
        return self

    # ── 主循环 ──────────────────────────────────

    def run(self) -> None:
        """启动服务，阻塞式地从 stdin 读取并处理消息。"""
        sys.stderr.write(f"[{self.name}] MCP Server started (v{self.version})\n")
        sys.stderr.flush()

        for msg in self.transport.receive_lines():
            self._handle_message(msg)

    # ── 消息处理 ────────────────────────────────

    def _handle_message(self, msg: dict) -> None:
        """处理单条 JSON-RPC 消息的入口。"""

        # 基础校验：MCP 只用 JSON-RPC 2.0
        if msg.get("jsonrpc") != "2.0":
            return

        msg_id = msg.get("id")     # 通知消息没有 id
        method = msg.get("method", "")
        params = msg.get("params") or {}

        # 没有 id 的是通知（Notification），无需响应
        # 例如 Client 握手后发的 "initialized" 通知
        if msg_id is None:
            self._handle_notification(method, params)
            return

        # 有 id 的是请求（Request），必须发回响应
        try:
            result = self._dispatch(method, params)
            self.transport.send({
                "jsonrpc": "2.0",
                "id": msg_id,
                "result": result,
            })
        except McpError as e:
            # McpError 携带标准 JSON-RPC 错误码
            self.transport.send({
                "jsonrpc": "2.0",
                "id": msg_id,
                "error": {"code": e.code, "message": str(e)},
            })
        except Exception as e:
            # 未预期异常统一用 -32000（通用服务端错误）
            sys.stderr.write(f"[{self.name}] Unhandled error: {e}\n")
            sys.stderr.flush()
            self.transport.send({
                "jsonrpc": "2.0",
                "id": msg_id,
                "error": {"code": -32000, "message": str(e)},
            })

    def _handle_notification(self, method: str, params: dict) -> None:
        """处理通知消息（不需要发送响应）。"""
        # "initialized" 是 Client 在握手完成后发来的，此处可以做初始化后的工作
        if method == "notifications/initialized":
            sys.stderr.write(f"[{self.name}] Client handshake complete\n")
            sys.stderr.flush()
        # 其他通知（如 cancelled、progress）按需处理，这里忽略

    # ── 方法分发 ────────────────────────────────

    def _dispatch(self, method: str, params: dict) -> Any:
        """根据 method 字符串路由到具体处理函数。"""
        routes = {
            "initialize":       self._handle_initialize,
            "tools/list":       self._handle_tools_list,
            "tools/call":       self._handle_tools_call,
            "resources/list":   self._handle_resources_list,
            "resources/read":   self._handle_resources_read,
            "prompts/list":     self._handle_prompts_list,
            "prompts/get":      self._handle_prompts_get,
        }
        handler = routes.get(method)
        if handler is None:
            # -32601 是 JSON-RPC 标准错误码：Method not found
            raise McpError(-32601, f"Method not found: {method}")
        return handler(params)

    # ── 握手 ────────────────────────────────────

    def _handle_initialize(self, params: dict) -> dict:
        """
        初始化握手。Client 在此协商协议版本并告知自身能力。
        Server 需要返回自己支持的 protocolVersion 和 capabilities。
        """
        client_version = params.get("protocolVersion", "unknown")
        sys.stderr.write(
            f"[{self.name}] Client requested version: {client_version}\n"
        )
        sys.stderr.flush()

        return {
            # protocolVersion 必须存在，且必须是 Server 实际支持的版本
            # 如果 Client 发来的版本 Server 不支持，应返回错误，
            # 但简单实现里通常直接声明自己支持的版本即可
            "protocolVersion": "2024-11-05",
            "capabilities": {
                # 声明支持 tools，listChanged=False 表示工具列表不会动态变化
                "tools": {"listChanged": False},
                # 声明支持 resources 和 prompts（空 dict 表示基础支持）
                "resources": {},
                "prompts": {},
            },
            "serverInfo": {
                "name": self.name,
                "version": self.version,
            },
        }

    # ── Tools ───────────────────────────────────

    def _handle_tools_list(self, params: dict) -> dict:
        """
        返回所有已注册工具的描述。
        注意：handler 函数本身不暴露给 Client，只暴露 name/description/inputSchema。
        模型根据 description 和 inputSchema 来决定何时调用、传什么参数。
        """
        tools = [
            {
                "name": t.name,
                "description": t.description,
                # MCP 规范要求 key 用驼峰 inputSchema，不是 input_schema
                "inputSchema": t.input_schema,
            }
            for t in self._tools.values()
        ]
        return {"tools": tools}

    def _handle_tools_call(self, params: dict) -> dict:
        """
        执行工具调用。
        params 结构：
          {
            "name": "tool_name",
            "arguments": { ...工具参数... }
          }
        """
        tool_name = params.get("name")
        arguments = params.get("arguments") or {}  # 参数可能为 null

        tool = self._tools.get(tool_name)
        if tool is None:
            raise McpError(-32000, f"Tool not found: {tool_name}")

        # 调用工具处理函数，捕获业务层抛出的异常
        try:
            text = tool.handler(arguments)
        except Exception as e:
            # 工具执行失败时，返回 isError=True 的 content，
            # 而不是 JSON-RPC error——这样模型可以看到错误信息并决定下一步
            return {
                "content": [{"type": "text", "text": f"Error: {e}"}],
                "isError": True,
            }

        # 正常返回：content 是一个列表，每项可以是 text、image 或 resource
        return {
            "content": [{"type": "text", "text": str(text)}],
        }

    # ── Resources ───────────────────────────────

    def _handle_resources_list(self, params: dict) -> dict:
        """列出所有可读资源的元信息（不包含内容本身）。"""
        resources = [
            {
                "uri": r.uri,
                "name": r.name,
                "description": r.description,
                "mimeType": r.mime_type,
            }
            for r in self._resources.values()
        ]
        return {"resources": resources}

    def _handle_resources_read(self, params: dict) -> dict:
        """
        读取指定 URI 的资源内容。
        params 结构：{ "uri": "file:///path/to/resource" }
        """
        uri = params.get("uri")
        resource = self._resources.get(uri)
        if resource is None:
            raise McpError(-32000, f"Resource not found: {uri}")

        content_text = resource.reader()
        return {
            "contents": [
                {
                    "uri": uri,
                    "mimeType": resource.mime_type,
                    "text": content_text,
                }
            ]
        }

    # ── Prompts ─────────────────────────────────

    def _handle_prompts_list(self, params: dict) -> dict:
        """列出所有可用的提示词模板。"""
        return {"prompts": self._prompts}

    def _handle_prompts_get(self, params: dict) -> dict:
        """
        获取一个提示词模板并填入动态参数。
        params 结构：
          {
            "name": "prompt_name",
            "arguments": { "arg1": "value1", ... }
          }
        此方法需要在子类或具体实现中覆盖，
        因为每个提示词的插值逻辑是业务相关的。
        """
        raise McpError(-32000, "prompts/get not implemented")


# ──────────────────────────────────────────────
# 辅助类：携带 JSON-RPC 错误码的异常
# ──────────────────────────────────────────────

class McpError(Exception):
    def __init__(self, code: int, message: str):
        super().__init__(message)
        self.code = code
```

---

## 六、工具定义（tools.py）

```python
# tools.py
#
# 把业务逻辑集中在这里，与框架代码解耦。
# 每个工具函数只关心：拿到 dict 参数，返回 str 结果。

import json
import datetime
from zoneinfo import ZoneInfo          # Python 3.9+，不需要 pytz
from server import ToolDefinition


# ──────────────────────────────────────────────
# 工具 1：数学计算器
# ──────────────────────────────────────────────

def _calculate(args: dict) -> str:
    """
    用 Python 内置 eval 计算数学表达式。
    生产环境建议替换为 simpleeval 等沙盒库，避免任意代码执行风险。
    """
    expression = args.get("expression", "")
    if not expression:
        raise ValueError("expression 不能为空")

    # 提供一个受限的全局命名空间，禁止访问 __builtins__ 中的危险函数
    # 这里只是示例，真实场景要用专门的表达式解析库
    allowed_names = {
        "abs": abs, "round": round,
        "min": min, "max": max,
        "pow": pow, "sum": sum,
    }
    try:
        result = eval(expression, {"__builtins__": {}}, allowed_names)
    except Exception as e:
        raise ValueError(f"无效的表达式：{e}")

    return f"{expression} = {result}"


TOOL_CALCULATE = ToolDefinition(
    name="calculate",
    description="执行数学计算，支持加减乘除、幂运算等基本运算。示例：'2 ** 10'、'(3 + 5) * 2'",
    input_schema={
        "type": "object",          # 顶层必须是 object
        "properties": {
            "expression": {
                "type": "string",
                "description": "要计算的数学表达式，例如 '2 + 3 * 4'",
            }
        },
        "required": ["expression"],
    },
    handler=_calculate,
)


# ──────────────────────────────────────────────
# 工具 2：获取当前时间
# ──────────────────────────────────────────────

def _get_current_time(args: dict) -> str:
    """返回指定时区的当前时间。timezone 参数可选，默认 Asia/Shanghai。"""
    tz_name = args.get("timezone", "Asia/Shanghai")
    try:
        tz = ZoneInfo(tz_name)
    except Exception:
        raise ValueError(f"无效的时区：{tz_name}，请使用 IANA 时区格式，如 'Asia/Shanghai'")

    now = datetime.datetime.now(tz)
    # 格式：2024-11-05 14:30:00 CST
    formatted = now.strftime("%Y-%m-%d %H:%M:%S %Z")
    return f"当前时间（{tz_name}）：{formatted}"


TOOL_GET_TIME = ToolDefinition(
    name="get_current_time",
    description="获取指定时区的当前日期和时间",
    input_schema={
        "type": "object",
        "properties": {
            "timezone": {
                "type": "string",
                "description": "IANA 时区名称，例如 'Asia/Shanghai'、'America/New_York'。不填则默认上海时间",
            }
        },
        # timezone 不是必填，所以不加 required
    },
    handler=_get_current_time,
)


# ──────────────────────────────────────────────
# 工具 3：查询天气（模拟数据）
# ──────────────────────────────────────────────

# 模拟数据库。真实场景替换为 requests.get("https://api.weather.com/...") 调用
_WEATHER_DB = {
    "北京": {"condition": "晴", "temp": 25, "wind": "东风 3 级"},
    "上海": {"condition": "多云", "temp": 22, "wind": "东南风 2 级"},
    "广州": {"condition": "阵雨", "temp": 28, "wind": "南风 4 级"},
    "成都": {"condition": "阴", "temp": 18, "wind": "微风"},
    "杭州": {"condition": "小雨", "temp": 20, "wind": "东北风 2 级"},
}


def _get_weather(args: dict) -> str:
    city = args.get("city", "").strip()
    if not city:
        raise ValueError("city 参数不能为空")

    data = _WEATHER_DB.get(city)
    if data is None:
        # 返回"暂无数据"而不是抛异常——让模型知道数据不存在，而不是工具出错
        return f"抱歉，暂无 {city} 的天气数据。目前支持的城市：{', '.join(_WEATHER_DB.keys())}"

    return (
        f"{city}天气：{data['condition']}，"
        f"{data['temp']}°C，{data['wind']}"
    )


TOOL_GET_WEATHER = ToolDefinition(
    name="get_weather",
    description="查询指定中国城市的实时天气，包括天气状况、温度和风力",
    input_schema={
        "type": "object",
        "properties": {
            "city": {
                "type": "string",
                "description": "城市名称，例如 '北京'、'上海'",
            }
        },
        "required": ["city"],
    },
    handler=_get_weather,
)


# ──────────────────────────────────────────────
# 工具 4：读取文件内容
# ──────────────────────────────────────────────

import os


def _read_file(args: dict) -> str:
    """
    读取本地文件内容。
    安全注意：生产环境必须做路径白名单校验，防止路径穿越攻击（../../../etc/passwd）。
    """
    path = args.get("path", "")
    if not path:
        raise ValueError("path 不能为空")

    # 路径规范化，消除 ../ 等相对路径符号
    abs_path = os.path.realpath(path)

    # 示例白名单：只允许读取 /tmp 下的文件
    ALLOWED_DIR = "/tmp"
    if not abs_path.startswith(ALLOWED_DIR):
        raise PermissionError(f"只允许读取 {ALLOWED_DIR} 目录下的文件")

    if not os.path.isfile(abs_path):
        raise FileNotFoundError(f"文件不存在：{abs_path}")

    with open(abs_path, "r", encoding="utf-8") as f:
        content = f.read()

    # 防止返回超大文件占满 context window
    MAX_CHARS = 10_000
    if len(content) > MAX_CHARS:
        content = content[:MAX_CHARS] + f"\n\n[内容已截断，共 {len(content)} 字符，只显示前 {MAX_CHARS} 字符]"

    return content


TOOL_READ_FILE = ToolDefinition(
    name="read_file",
    description="读取本地文件内容（仅限 /tmp 目录）",
    input_schema={
        "type": "object",
        "properties": {
            "path": {
                "type": "string",
                "description": "文件路径，例如 '/tmp/data.txt'",
            }
        },
        "required": ["path"],
    },
    handler=_read_file,
)


# 导出所有工具的列表，方便 main.py 批量注册
ALL_TOOLS = [TOOL_CALCULATE, TOOL_GET_TIME, TOOL_GET_WEATHER, TOOL_READ_FILE]
```

---

## 七、Resource 与 Prompt 实现

### 7.1 实现 Resources

Resources 适合暴露"可被模型引用的数据"，而非直接调用的函数。

```python
# 在 main.py 中注册 Resource

from server import ResourceDefinition
import json

def read_app_config() -> str:
    """
    读取并返回应用配置。
    这里是静态数据；真实场景可以读配置文件、查数据库等。
    """
    config = {
        "version": "1.0",
        "env": "production",
        "features": {
            "dark_mode": True,
            "beta_tools": False,
        }
    }
    # 返回格式化的 JSON 字符串，让模型更容易解析
    return json.dumps(config, ensure_ascii=False, indent=2)


APP_CONFIG_RESOURCE = ResourceDefinition(
    # URI 格式自定义，但要保持一致性；file:// 适合对应本地文件的语义
    uri="config://app/settings",
    name="应用配置",
    description="当前应用的运行时配置，包含功能开关和版本信息",
    mime_type="application/json",
    reader=read_app_config,
)
```

### 7.2 实现 Prompts

Prompts 是可复用的提示词模板，支持参数插值。

```python
# 在 McpServer 的子类或 main.py 中实现 prompts/get 路由

def handle_prompts_get_impl(params: dict) -> dict:
    """
    prompts/get 的具体实现。
    params 结构：{ "name": "...", "arguments": { ... } }
    """
    name = params.get("name")
    arguments = params.get("arguments") or {}

    if name == "code_review":
        language = arguments.get("language", "python")
        code = arguments.get("code", "")
        if not code:
            raise ValueError("code 参数不能为空")

        # 返回格式：messages 列表，每项包含 role 和 content
        return {
            "description": "代码审查提示词",
            "messages": [
                {
                    "role": "user",
                    "content": {
                        "type": "text",
                        # 用 f-string 插值，生成最终发给模型的提示词
                        "text": (
                            f"请对以下 {language} 代码进行审查，"
                            f"指出潜在的 Bug、安全问题和可读性改进建议：\n\n"
                            f"```{language}\n{code}\n```"
                        ),
                    },
                }
            ],
        }

    raise McpError(-32000, f"Prompt not found: {name}")


# 对应的 prompts/list 中声明这个模板：
CODE_REVIEW_PROMPT = {
    "name": "code_review",
    "description": "自动生成代码审查请求",
    "arguments": [
        {
            "name": "language",
            "description": "编程语言，例如 'python'、'go'、'typescript'",
            "required": True,
        },
        {
            "name": "code",
            "description": "要审查的代码片段",
            "required": True,
        },
    ],
}
```

---

## 八、入口文件（main.py）

```python
# main.py
#
# 组装各模块，启动服务。
# 尽量保持这个文件简短——它只做"把部件拼起来"这一件事。

from server import McpServer, McpError
from tools import ALL_TOOLS
from server import ResourceDefinition
import json


def read_app_config() -> str:
    config = {"version": "1.0", "env": "production"}
    return json.dumps(config, ensure_ascii=False, indent=2)


def main():
    server = McpServer(name="my-python-server", version="1.0.0")

    # 批量注册工具
    for tool in ALL_TOOLS:
        server.register_tool(tool)

    # 注册 Resource
    server.register_resource(ResourceDefinition(
        uri="config://app/settings",
        name="应用配置",
        description="当前应用的运行时配置",
        mime_type="application/json",
        reader=read_app_config,
    ))

    # 注册 Prompt 元信息（实际渲染逻辑在 server.py 的 _handle_prompts_get 中扩展）
    server.register_prompt({
        "name": "code_review",
        "description": "自动生成代码审查请求",
        "arguments": [
            {"name": "language", "description": "编程语言", "required": True},
            {"name": "code", "description": "待审查代码", "required": True},
        ],
    })

    # 启动主循环，阻塞直到 stdin 关闭（即 Client 断开连接）
    server.run()


if __name__ == "__main__":
    main()
```

---

## 九、接入 Claude Desktop

编辑配置文件（macOS 路径：`~/Library/Application Support/Claude/claude_desktop_config.json`）：

```json
{
  "mcpServers": {
    "my-python-server": {
      "command": "python3",
      "args": ["/path/to/my_mcp_server/main.py"],
      "env": {
        "PYTHONIOENCODING": "utf-8"
      }
    }
  }
}
```

`PYTHONIOENCODING=utf-8` 是必要的，在 Windows 上尤其重要，防止中文输出乱码。

重启 Claude Desktop，对话界面出现锤子图标即表示接入成功。

---

## 十、调试技巧

### 手动测试（无需 Client）

```bash
# 直接用 echo 发送 JSON-RPC 消息到 stdin
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' | python3 main.py

# 多条消息测试（heredoc）
python3 main.py << 'EOF'
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}
{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}
{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"get_weather","arguments":{"city":"北京"}}}
{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"calculate","arguments":{"expression":"2**10"}}}
EOF
```

### MCP Inspector（官方可视化调试工具）

```bash
npx @modelcontextprotocol/inspector python3 /path/to/main.py
```

打开浏览器 `http://localhost:5173`，可以点击界面调用工具、查看请求/响应的原始 JSON。

### 用 stderr 打印日志

```python
import sys

def log(msg: str) -> None:
    """所有调试日志走 stderr，绝不能写 stdout（stdout 专用于协议消息）。"""
    sys.stderr.write(f"[DEBUG] {msg}\n")
    sys.stderr.flush()
```

---

## 十一、常见坑

| 问题 | 原因 | 解决 |
|------|------|------|
| Client 收不到响应 | `print()` 默认写 stdout，污染了协议流 | 所有日志改用 `sys.stderr.write()` |
| 消息卡住不发送 | `stdout` 有缓冲，消息憋在缓冲区 | 每次 `send` 后调用 `flush()` |
| 中文响应乱码 | Windows 默认编码非 UTF-8 | 用 `sys.stdout.buffer` 写字节，或设置 `PYTHONIOENCODING=utf-8` |
| 握手失败 | `initialize` 响应缺少 `protocolVersion` 字段 | 确认字段名称和格式完全正确 |
| 工具不在列表里 | `inputSchema` 缺少 `"type": "object"` | JSON Schema 顶层必须是 object |
| 工具调用后进程崩溃 | handler 抛异常未捕获 | 在 `_handle_tools_call` 中包一层 try/except |
| 模型传参格式错误 | description 写得不够精确 | 在 description 中给出参数的格式示例 |

---

## 总结

MCP 协议的核心可以用一句话概括：**JSON-RPC 2.0 over stdio，三类能力（Tools / Resources / Prompts），一次 initialize 握手**。

Python 实现的要点：

1. **stdout 只写协议 JSON**，日志全走 stderr，flush 不能忘
2. **通知消息（无 id）不回响应**，请求消息（有 id）必须回
3. **工具的 inputSchema 要精准**，模型靠它决定传什么参数
4. **工具执行失败用 `isError: true` 的 content** 而非 JSON-RPC error，让模型看到错误信息
5. **路径、权限等安全边界在工具 handler 里做**，不要依赖框架层

理解了这些之后，再去使用官方 SDK（`mcp` Python 包）会更清晰，知道 SDK 在哪些地方帮你做了抽象，遇到问题时也更容易定位根因。
