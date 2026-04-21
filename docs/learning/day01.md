# Day 1 — 项目全局认知 & 核心架构

## 今天的目标

理解这个项目是什么、为什么值得写进简历、以及整体代码是怎么组织的。**不写代码，只读和理解。**

---

## 一、项目定位（30 分钟）

**nanobot** 是一个极简的个人 AI Agent 框架，核心卖点：

| 特性 | 意义 |
|------|------|
| 多 LLM 支持 | Anthropic / OpenAI / Gemini 等，统一接口 |
| 多渠道接入 | Telegram、飞书、Slack、Discord 等一套代码全覆盖 |
| 技能（Skill）系统 | 可插拔功能模块：天气、记忆、定时任务、GitHub |
| MCP 集成 | 接入任意 MCP 工具服务器 |
| 极轻量 | 没有 LangChain/litellm，自己实现 provider 抽象 |

**为什么能写进简历：**
- 涉及 LLM 工程的全链路（prompt → 工具调用 → 记忆 → 多渠道输出）
- Python 异步编程、Pydantic 数据建模、WebSocket 实时通信
- 软件架构能力（Provider 抽象、Bus 消息总线、Channel 插件化）

---

## 二、读懂目录结构（1 小时）

按顺序读这几个文件，每个只读开头和类定义，不需要逐行理解：

```
nanobot/
├── config/        ← 第1步：整个系统的配置入口
├── providers/     ← 第2步：LLM 供应商抽象（核心设计）
├── agent/         ← 第3步：Agent 运行时生命周期
├── bus/           ← 第4步：消息总线（模块解耦的关键）
├── channels/      ← 第5步：渠道适配器（理解插件化）
├── skills/        ← 第6步：技能系统
├── session/       ← 第7步：会话与上下文管理
```

**读法建议：** 每个目录先看 `__init__.py`，再看主文件的类名和方法签名，理解"这个模块负责什么"。

---

## 三、核心架构图

```
用户消息 (Telegram/Slack/飞书)
       ↓
   Channel 适配器
       ↓
   消息总线 (Bus)
       ↓
   Agent Runtime
    ├── Session 管理 (历史上下文)
    ├── Skill 路由 (工具调用判断)
    ├── Provider 选择 (Anthropic/OpenAI...)
    └── Dream/Memory (长期记忆)
       ↓
   LLM API 请求
       ↓
   响应回传 → Channel → 用户
```

自己动手画一遍，加深理解。

---

## 四、动手验证（30 分钟）

不需要真正运行，只需要在本地搜索几个关键词，确认你的理解：

```bash
# 1. 看 provider 抽象的基类在哪
grep -r "class.*Provider" nanobot/providers/ --include="*.py" -l

# 2. 看 channel 是怎么注册的
grep -r "register\|Channel" nanobot/channels/ --include="*.py" -l

# 3. 看 skill 是怎么被 agent 调用的
grep -r "skill\|execute" nanobot/agent/ --include="*.py" -l

# 4. 看消息总线的事件类型
grep -r "class.*Event\|publish\|subscribe" nanobot/bus/ --include="*.py"
```

---

## 五、今天的输出（写下来，作为笔记）

用自己的话回答这 4 个问题：

1. nanobot 解决的核心问题是什么？和直接调用 OpenAI API 相比，它多了什么？
2. Provider 抽象层的好处是什么？如果没有它，代码会怎样？
3. 消息总线（Bus）为什么能让模块解耦？举例说明。
4. 一条来自飞书的消息，从进入系统到 LLM 回复，完整走了哪些模块？

---

## 六、简历相关技能点

今天读完后储备这些关键词（后续实战后再写进简历）：

- **LLM Provider 抽象设计** — 统一接口屏蔽多厂商差异
- **事件驱动架构** — 消息总线解耦 Agent 与 Channel
- **多渠道 AI Agent** — 单一 Agent 逻辑适配多平台
- **Python 异步编程** — asyncio + httpx + WebSocket
- **Pydantic 数据建模** — 配置管理与消息结构校验

---

## 明天预告

**Day 2 — Provider 层深度阅读 + 手写一个最简 Provider**

理解 Anthropic 和 OpenAI 的接入方式，然后仿照现有代码，为一个假设的新 LLM（比如本地 Ollama）写一个 Provider 骨架。
