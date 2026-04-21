# Day 1 — 项目全局认知 & 核心架构

> 目标：理解项目是什么、整体架构如何运作、涉及哪些 Python 核心概念。
> 预计时间：3-4 小时。零基础友好，每个知识点都有代码示例 + 项目对应位置。

---

## 一、这个项目是什么？（20 分钟）

**nanobot** 是一个多渠道 AI Agent 框架。

用大白话说：你发一条消息给飞书/Telegram/Slack，它背后调用 Claude/GPT 等大模型，
然后把回复发回给你。不同平台的接入逻辑、大模型的调用逻辑、记忆管理等全部封装好了。

```
你在飞书发消息 "帮我查一下明天天气"
        ↓
  飞书 Channel 接收消息
        ↓
  消息总线 (Bus) 传递
        ↓
  Agent 处理：调用天气工具 → 调用 Claude API → 生成回复
        ↓
  消息总线回传
        ↓
  飞书 Channel 发送回复给你
```

**为什么能写进简历：**

| 涉及领域 | 具体技术 |
|---------|---------|
| LLM 工程 | Provider 抽象、工具调用（Tool Use）、上下文管理 |
| 后端架构 | 事件驱动、消息总线、插件化设计 |
| Python 进阶 | asyncio 异步编程、ABC 抽象类、Pydantic 数据建模 |
| 多平台集成 | Telegram Bot、飞书 Lark、Slack SDK |

---

## 二、读项目前必须掌握的 Python 概念（1.5 小时）

### 2.1 dataclass — 数据容器

Python 的 `@dataclass` 装饰器自动生成 `__init__`、`__repr__` 等方法，
让你专注于描述数据结构。

```python
# 普通写法（啰嗦）
class User:
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age

# dataclass 写法（简洁）
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    age: int
    tags: list[str] = field(default_factory=list)  # 可变默认值必须用 field
```

**项目中的使用** — `nanobot/bus/events.py`：

```python
@dataclass
class InboundMessage:
    channel: str        # 来自哪个渠道，如 "telegram"
    sender_id: str      # 发送者 ID
    chat_id: str        # 会话 ID
    content: str        # 消息内容
    media: list[str] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)

    @property
    def session_key(self) -> str:
        # 把 channel + chat_id 拼成唯一会话标识
        return f"{self.channel}:{self.chat_id}"
```

**练习：** 自己写一个 `OutboundMessage` dataclass，包含 channel、chat_id、content 字段。

---

### 2.2 ABC 抽象基类 — 定义接口规范

`ABC`（Abstract Base Class）让你定义"必须实现"的方法，起到接口的作用。
子类如果没有实现 `@abstractmethod` 方法，实例化时会报错。

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    @abstractmethod
    def speak(self) -> str:
        ...  # 不写实现，只定义签名

class Dog(Animal):
    def speak(self) -> str:
        return "Woof!"

class Cat(Animal):
    def speak(self) -> str:
        return "Meow!"

# Animal()  ← 报错！不能直接实例化抽象类
dog = Dog()  # ✓ 可以
```

**项目中的使用** — `nanobot/channels/base.py`：

```python
class BaseChannel(ABC):
    @abstractmethod
    async def start(self) -> None:
        ...   # 子类必须实现：如何连接到平台

    @abstractmethod
    async def stop(self) -> None:
        ...   # 子类必须实现：如何断开连接

    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None:
        ...   # 子类必须实现：如何发送消息
```

`TelegramChannel`、`FeishuChannel` 等都继承 `BaseChannel`，
各自实现 `start/stop/send`，但对外暴露统一接口。

---

### 2.3 async / await — 异步编程

同步代码：一个任务做完才做下一个（等待期间 CPU 闲着）。
异步代码：等待 IO（网络请求、文件读写）时，CPU 去做其他事情。

```python
import asyncio

# 同步版本（阻塞）
import time

def fetch_sync(url: str) -> str:
    time.sleep(2)      # 等 2 秒，CPU 什么都不做
    return f"data from {url}"

def main_sync():
    r1 = fetch_sync("url1")   # 等 2 秒
    r2 = fetch_sync("url2")   # 再等 2 秒，共 4 秒
    print(r1, r2)


# 异步版本（非阻塞）
async def fetch_async(url: str) -> str:
    await asyncio.sleep(2)    # 等待时让出控制权
    return f"data from {url}"

async def main_async():
    # 两个请求同时进行，只需 2 秒
    r1, r2 = await asyncio.gather(
        fetch_async("url1"),
        fetch_async("url2"),
    )
    print(r1, r2)

asyncio.run(main_async())
```

**项目中的使用** — `nanobot/providers/base.py`：

```python
class LLMProvider(ABC):
    @abstractmethod
    async def chat(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
        model: str | None = None,
        max_tokens: int = 4096,
        temperature: float = 0.7,
    ) -> LLMResponse:
        ...
        # async 因为要发 HTTP 请求到 Claude/GPT，等待响应时不阻塞其他任务
```

---

### 2.4 Pydantic — 数据验证与配置

Pydantic 让你用类型注解定义数据结构，并自动验证数据类型、提供默认值、读取环境变量。

```python
from pydantic import BaseModel, Field

class DatabaseConfig(BaseModel):
    host: str = "localhost"
    port: int = 5432
    name: str

    # 自动验证
    # DatabaseConfig(host="db", port="not_a_number", name="mydb")
    # → ValidationError: port 不是整数

config = DatabaseConfig(name="mydb")
print(config.host)   # "localhost"
print(config.port)   # 5432
```

**项目中的使用** — `nanobot/config/schema.py`：

```python
from pydantic import BaseModel, Field
from pydantic_settings import BaseSettings

class ProviderConfig(BaseModel):
    api_key: str | None = None      # 可以为空
    api_base: str | None = None     # 自定义 API 地址

class AgentDefaults(BaseModel):
    model: str = "anthropic/claude-opus-4-5"   # 默认模型
    max_tokens: int = 8192                      # 最大生成长度
    temperature: float = 0.1                    # 随机性（0=确定，1=随机）
    max_tool_iterations: int = 200              # 最多调用工具多少次

class Config(BaseSettings):
    # BaseSettings 会自动读取环境变量 NANOBOT_xxx
    providers: ProvidersConfig = Field(default_factory=ProvidersConfig)
    agents: AgentsConfig = Field(default_factory=AgentsConfig)

    model_config = ConfigDict(
        env_prefix="NANOBOT_",          # 环境变量前缀
        env_nested_delimiter="__"       # NANOBOT_PROVIDERS__ANTHROPIC__API_KEY
    )
```

---

### 2.5 asyncio.Queue — 异步队列（消息总线的基础）

队列是生产者-消费者模式的核心：一端放数据，另一端取数据，两端不需要直接通信。

```python
import asyncio

async def producer(queue: asyncio.Queue):
    for i in range(3):
        await queue.put(f"消息{i}")
        print(f"放入: 消息{i}")

async def consumer(queue: asyncio.Queue):
    while True:
        msg = await queue.get()     # 没有消息时等待，不阻塞
        print(f"取出: {msg}")
        queue.task_done()

async def main():
    queue = asyncio.Queue()
    await asyncio.gather(
        producer(queue),
        consumer(queue),
    )

asyncio.run(main())
```

**项目中的使用** — `nanobot/bus/queue.py`：

```python
class MessageBus:
    def __init__(self):
        self.inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()
        self.outbound: asyncio.Queue[OutboundMessage] = asyncio.Queue()

    async def publish_inbound(self, msg: InboundMessage) -> None:
        await self.inbound.put(msg)      # Channel → Bus

    async def consume_inbound(self) -> InboundMessage:
        return await self.inbound.get()  # Bus → Agent

    async def publish_outbound(self, msg: OutboundMessage) -> None:
        await self.outbound.put(msg)     # Agent → Bus

    async def consume_outbound(self) -> OutboundMessage:
        return await self.outbound.get() # Bus → Channel
```

---

## 三、核心架构：从消息到回复的完整旅程（45 分钟）

### 3.1 整体数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                         nanobot 系统                             │
│                                                                  │
│  飞书/Telegram/Slack                                             │
│       │                                                          │
│       ↓  ①用户发消息                                             │
│  ┌──────────┐     ②InboundMessage     ┌─────────────────────┐   │
│  │ Channel  │ ──────────────────────→ │    MessageBus        │   │
│  │ 适配器   │                         │  inbound: Queue      │   │
│  │          │ ←────────────────────── │  outbound: Queue     │   │
│  └──────────┘    ⑥OutboundMessage     └──────────┬──────────┘   │
│                                                  │③取出消息      │
│                                                  ↓               │
│                                       ┌─────────────────────┐   │
│                                       │     AgentLoop        │   │
│                                       │  ┌───────────────┐  │   │
│                                       │  │ Session 管理  │  │   │
│                                       │  │ (历史上下文)  │  │   │
│                                       │  └───────────────┘  │   │
│                                       │          │④构建上下文 │   │
│                                       │          ↓           │   │
│                                       │  ┌───────────────┐  │   │
│                                       │  │  AgentRunner  │  │   │
│                                       │  │ (工具调用循环) │  │   │
│                                       │  └───────┬───────┘  │   │
│                                       └──────────┼──────────┘   │
│                                                  │⑤调用LLM      │
│                                                  ↓               │
│                                       ┌─────────────────────┐   │
│                                       │    LLMProvider       │   │
│                                       │  AnthropicProvider   │   │
│                                       │  OpenAIProvider      │   │
│                                       └─────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Provider 抽象：为什么这样设计？

**问题：** 项目支持 25+ 个 LLM 提供商。如果每个业务逻辑都写 `if provider == "anthropic": ... elif provider == "openai": ...`，代码会极其混乱。

**解决方案：** 用抽象基类定义统一接口，每个 Provider 各自实现细节。

```python
# nanobot/providers/base.py — 定义接口（所有 Provider 必须遵守）
class LLMProvider(ABC):
    @abstractmethod
    async def chat(self, messages, tools, model, max_tokens, temperature) -> LLMResponse:
        ...

    @abstractmethod
    def get_default_model(self) -> str:
        ...

# nanobot/providers/anthropic_provider.py — Anthropic 的具体实现
class AnthropicProvider(LLMProvider):
    async def chat(self, messages, tools, model, max_tokens, temperature) -> LLMResponse:
        # 把通用 messages 格式转成 Anthropic 要求的格式
        system, anthropic_msgs = self._convert_messages(messages)
        # 调用 Anthropic SDK
        response = await self._client.messages.create(...)
        # 把 Anthropic 响应转回通用 LLMResponse 格式
        return self._parse_response(response)

    def get_default_model(self) -> str:
        return "claude-sonnet-4-20250514"

# nanobot/providers/openai_compat.py — OpenAI 兼容的实现（适用于 20+ 个提供商）
class OpenAICompatProvider(LLMProvider):
    async def chat(self, messages, tools, model, max_tokens, temperature) -> LLMResponse:
        # OpenAI 格式不需要转换，直接发
        response = await self._client.chat.completions.create(...)
        return self._parse_response(response)
```

**效果：** `AgentLoop` 只需要调用 `provider.chat(messages)`，
完全不知道背后是 Anthropic 还是 OpenAI，新增 Provider 只需加一个类。

### 3.3 消息总线：为什么不让 Channel 直接调用 Agent？

**直接调用的问题：**
```python
# 糟糕的设计
class TelegramChannel:
    def on_message(self, msg):
        agent = AgentLoop(...)
        response = agent.process(msg)   # Channel 和 Agent 紧耦合
        self.send(response)
```

如果 Agent 换了实现，所有 Channel 都要改。如果要加一个新 Channel，要知道 Agent 的内部结构。

**消息总线的设计：**
```python
# 好的设计
class TelegramChannel(BaseChannel):
    async def on_message(self, msg):
        inbound = InboundMessage(
            channel="telegram",
            sender_id=msg.from_user.id,
            chat_id=msg.chat.id,
            content=msg.text,
        )
        await self.bus.publish_inbound(inbound)   # 只管发到总线

class AgentLoop:
    async def run(self):
        while True:
            inbound = await self.bus.consume_inbound()  # 只管从总线取
            response = await self._process(inbound)
            outbound = OutboundMessage(...)
            await self.bus.publish_outbound(outbound)   # 发回总线
```

Channel 和 Agent 都只和 Bus 交互，互相不知道对方的存在。

### 3.4 两层 Agent 设计

```
AgentLoop（产品层）
  - 管理 Session（记住历史对话）
  - 构建系统提示词（system prompt）
  - 注册可用工具（天气、文件读写、代码执行...）
  - 处理渠道相关逻辑（进度提示、流式输出）
        ↓ 调用
AgentRunner（执行层）
  - 纯粹的 LLM + 工具调用循环
  - 不关心来自哪个渠道
  - 不关心 Session 是怎么管理的
  - 管理上下文窗口（防止超出 token 限制）
```

**为什么分两层？**

`AgentRunner` 是可复用的通用引擎，将来可以用在 CLI、API Server 等任何地方。
`AgentLoop` 包含业务逻辑，只用于多渠道对话场景。

---

## 四、动手读代码（30 分钟）

按顺序读以下文件，每个只需要看类定义和方法签名：

```bash
# 在项目根目录执行

# 1. 看 Provider 基类定义了哪些抽象方法
grep -n "abstractmethod\|def chat\|def get_default" nanobot/providers/base.py

# 2. 看消息总线有哪些方法
grep -n "async def" nanobot/bus/queue.py

# 3. 看 Channel 基类要求子类实现什么
grep -n "abstractmethod\|async def" nanobot/channels/base.py

# 4. 看 AgentRunner 的核心循环入口
grep -n "async def run\|async def _execute" nanobot/agent/runner.py

# 5. 看配置里支持哪些 Provider
grep -n "class.*Config\|anthropic\|openai" nanobot/config/schema.py | head -30
```

---

## 五、关键数据结构速查

### InboundMessage（入站消息）

```python
# nanobot/bus/events.py
@dataclass
class InboundMessage:
    channel: str        # "telegram" | "feishu" | "slack" | ...
    sender_id: str      # 用户 ID
    chat_id: str        # 会话 ID（群组或私聊）
    content: str        # 文字内容
    media: list[str]    # 附件路径列表（语音、图片等）
    metadata: dict      # 渠道特有数据

    @property
    def session_key(self) -> str:
        return f"{self.channel}:{self.chat_id}"
        # 例如 "telegram:123456789"
```

### LLMResponse（LLM 返回）

```python
# nanobot/providers/base.py
@dataclass
class LLMResponse:
    content: str | None             # 文字回复
    tool_calls: list[ToolCallRequest]  # 要调用的工具
    finish_reason: str              # "stop" | "tool_use" | "length" | "error"
    usage: dict[str, int]           # token 消耗统计

    @property
    def has_tool_calls(self) -> bool:
        return len(self.tool_calls) > 0
```

### ToolCallRequest（工具调用请求）

```python
# nanobot/providers/base.py
@dataclass
class ToolCallRequest:
    id: str             # 工具调用 ID（用于返回结果时匹配）
    name: str           # 工具名称，如 "web_search"
    arguments: dict     # 参数，如 {"query": "今天天气"}
```

---

## 六、用自己的话回答（写下来加深理解）

1. **什么是 Provider 抽象？** 用一句话解释为什么需要它。

2. **消息总线解决了什么问题？** 如果去掉总线，Channel 和 Agent 怎么通信？有什么坏处？

3. **AgentLoop 和 AgentRunner 各自负责什么？** 为什么要分两层？

4. **一条来自 Telegram 的消息"帮我查天气"，走过哪些模块？** 尝试用代码或文字描述完整路径。

---

## 七、今天学到的简历关键词

> 实战后再填写，今天只是理解

- **LLM Provider 抽象层** — 用 ABC + 统一接口屏蔽 25+ 个大模型厂商的 API 差异
- **事件驱动 + 消息总线** — asyncio.Queue 解耦 Channel 适配器与 Agent 执行层
- **多渠道 AI Agent** — 单一 Agent 逻辑通过插件化 Channel 适配 Telegram/飞书/Slack 等平台
- **两层 Agent 架构** — AgentRunner（通用执行引擎）与 AgentLoop（产品业务层）分离
- **Pydantic 配置建模** — BaseSettings 统一管理 25+ Provider 的配置和环境变量

---

## 八、明天预告

**Day 2 — Provider 层深度实战**

- 精读 `AnthropicProvider` 实现，理解消息格式转换
- 精读重试机制（`_run_with_retry`），理解为什么要自己管重试而不用 SDK 内置的
- **动手实践：** 仿照现有代码，为 Ollama（本地模型）写一个最简 Provider 骨架
