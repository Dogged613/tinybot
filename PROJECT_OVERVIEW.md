# nanobot — 轻量级个人 AI Agent 框架

> 项目地址：https://github.com/HKUDS/nanobot  
> 版本：v0.1.5.post1  
> 语言：Python 3.11+  
> 许可证：MIT

---

## 项目简介

nanobot 是一个**超轻量级、生产可用的个人 AI Agent 框架**，由香港大学数据科学学院（HKUDS）发布。其核心理念是：用最少的代码实现完整的 Agent 能力——相比同类项目（如 OpenClaw）减少 99% 的代码量，同时覆盖多渠道接入、持久记忆、工具调用、技能系统、定时任务等完整特性。

本人基于该项目进行二次开发，完成了配置适配、中转 API 接入、本地环境搭建，并深入研究其架构设计，作为 AI Agent 工程实践的学习与展示项目。

---

## 技术栈

| 类别 | 技术 |
|------|------|
| 运行时 | Python 3.11、asyncio |
| LLM 接入 | OpenAI SDK、Anthropic SDK、兼容 OpenAI 格式的中转 API |
| 渠道集成 | Telegram、飞书、钉钉、Slack、Discord、微信、WhatsApp、QQ 等 13+ |
| 数据格式 | Pydantic v2、JSONL、YAML |
| 工具生态 | MCP（Model Context Protocol）、Jinja2 模板、Cron 调度 |
| 存储 | 本地文件 + Git（dulwich）版本追踪 |
| 包管理 | uv、hatch |

---

## 核心架构

nanobot 采用**六层解耦架构**，每层职责单一，可独立扩展：

```
┌─────────────────────────────────────────────┐
│         Chat Platforms（消息渠道层）          │
│  Telegram / 飞书 / Discord / Slack / 微信…  │
└──────────────────┬──────────────────────────┘
                   │ InboundMessage / OutboundMessage
┌──────────────────▼──────────────────────────┐
│           Message Bus（消息总线）             │
│        asyncio.Queue 解耦收发两端            │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│          Agent Loop（调度引擎）               │
│  Session 管理 · 工具注册 · 并发控制          │
└──────────────────┬──────────────────────────┘
                   │ AgentRunSpec
┌──────────────────▼──────────────────────────┐
│         Agent Runner（LLM 执行引擎）          │
│      无状态 ReAct 循环 · Context 治理        │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│         Provider Layer（模型抽象层）          │
│  Anthropic · OpenAI · DeepSeek · 30+ 提供商 │
└─────────────────────────────────────────────┘
```

### 关键设计决策

**1. AgentRunner / AgentLoop 二层解耦**

AgentRunner 是完全无状态的执行引擎，只接受 `AgentRunSpec` 数据对象，返回 `AgentRunResult`。AgentLoop 是有状态的业务层，负责 Session、Memory、Channel 等产品关注点。两者解耦使得 Dream 记忆处理器、SubAgent 等可以直接复用同一 Runner，无需继承或 mock。

**2. Mid-turn 消息注入**

同一 Session 内，当 Agent 正在处理任务时，新消息不会排队等待，而是通过 `injection_callback` 在工具调用间隙被注入到当前对话轮次，减少等待延迟，实现"打断并追加"语义。

**3. Context Governance 分层保护**

每次 LLM 调用前按顺序执行 5 步上下文治理：
- 清除孤儿工具结果 → 补填占位符 → 微压缩旧结果 → token 预算截断 → 历史裁剪
- 原始 `messages` 不被修改，治理仅影响发给模型的副本，保证持久化数据完整性

**4. Provider 注册表驱动**

新增 LLM Provider 只需在 `registry.py` 添加一条 `ProviderSpec` 数据声明，无需修改任何业务代码。env 变量、状态展示、model 匹配、重试策略全部自动推导。

---

## 核心功能模块

### 记忆系统（三层）

| 层级 | 文件 | 职责 |
|------|------|------|
| 持久存储层 | `MEMORY.md` / `USER.md` / `SOUL.md` / `history.jsonl` | 文件 I/O，Git 追踪变更 |
| Consolidator | 内嵌于 memory.py | token 触发，用 LLM 摘要旧消息，追加到 history.jsonl |
| Dream | 后台 cron 任务 | 两阶段深度处理：Phase 1 纯分析，Phase 2 用工具精确增量编辑记忆文件 |

Dream 的行龄标注机制：通过 `git blame` 为超过 14 天未更新的记忆行附加 `← Nd` 标记，引导 LLM 主动清理过时信息。

### 技能系统（渐进加载）

技能以 Markdown 文件（`SKILL.md`）定义，支持两种加载模式：
- **Always Skills**：`always: true` 的技能全文注入 system prompt
- **Progressive Skills**：其余技能仅注入摘要 + 路径，LLM 按需通过 `read_file` 加载

这使得支持数十个技能而不超出 context window。Dream 可在运行中动态发现并创建新技能文件。

### 工具体系

| 工具类别 | 工具 |
|----------|------|
| 文件操作 | read_file, write_file, edit_file, list_dir, glob, grep |
| Shell 执行 | exec（支持 sandbox 隔离） |
| 网络 | web_search, web_fetch |
| 消息 | send_message（跨渠道发送） |
| 子 Agent | spawn（创建子 Agent 执行子任务） |
| 定时任务 | cron_add, cron_list, cron_remove |
| MCP | 动态挂载任意 MCP Server 工具 |
| Notebook | Jupyter Notebook 单元格编辑 |

### Provider 支持（30+）

商业 API：OpenAI、Anthropic、DeepSeek、Gemini、通义千问、月之暗面、MiniMax、Mistral、StepFun  
网关中转：OpenRouter、AiHubMix、SiliconFlow、火山引擎、BytePlus、自定义兼容端点  
本地部署：Ollama、vLLM、LM Studio  
OAuth：GitHub Copilot、OpenAI Codex

---

## 渠道集成

| 渠道 | 特性 |
|------|------|
| Telegram | 流式输出、媒体收发、代理支持 |
| 飞书（Feishu/Lark） | 流式卡片、富文本、全球域名支持 |
| 钉钉 | 富媒体消息 |
| Slack | Socket/Webhook 双模式、线程隔离 |
| Discord | 长消息分割、流式输出 |
| 微信（个人号） | 多媒体、语音 |
| WeCom（企业微信） | Bot 集成 |
| WhatsApp | Bridge 方案 |
| QQ | 群聊、媒体 |
| Email | IMAP/SMTP、附件 |
| WebSocket | 自定义客户端接入 |
| MS Teams | Bot Framework |
| Matrix | E2E 加密 |

渠道通过**插件机制**扩展：`entry_points(group="nanobot.channels")` 自动发现外部渠道插件，无需修改核心代码。

---

## 项目亮点（用于简历）

1. **极简架构实现完整 Agent 能力**：核心 Agent 循环约 600 行代码，实现工具调用、上下文管理、记忆持久化、多渠道接入的完整闭环。

2. **生产级 Context 治理**：五步上下文治理流水线，在有限 context window 内最大化信息密度，同时保证消息格式合法性。

3. **智能记忆系统**：三层记忆架构（即时存储、轻量整合、重量级 Dream），通过 git blame 行龄追踪实现记忆的自动老化与清理。

4. **异步并发设计**：per-session 锁保证串行，跨 session 并行，全局 Semaphore 限流，mid-turn 注入减少等待，充分利用 asyncio 并发能力。

5. **多 Provider 统一抽象**：注册表驱动的 Provider 系统，支持 30+ LLM 提供商，具备细粒度重试策略（区分 rate limit 与配额耗尽）、图片内容自动降级、Anthropic prompt cache 优化。

6. **渐进技能加载**：技能数量与 prompt token 消耗解耦，支持大规模技能库而不膨胀 system prompt。

---

## 本地运行

```bash
# 克隆并安装
git clone https://github.com/HKUDS/nanobot.git
cd nanobot
uv venv --python 3.11
source .venv/bin/activate
uv pip install -e .

# 初始化
nanobot onboard

# 配置 ~/.nanobot/config.json（API Key + 模型）

# 启动 CLI 对话
nanobot

# 或后台服务模式
nanobot start
```

---

## 二次开发方向

基于 nanobot 的架构，以下方向具有较低的扩展成本：

| 方向 | 扩展点 |
|------|--------|
| 接入新 LLM | 在 `providers/registry.py` 添加 `ProviderSpec` |
| 添加新渠道 | 继承 `BaseChannel`，注册 `entry_point` |
| 自定义工具 | 继承 `BaseTool`，注册到 `ToolRegistry` |
| 自定义技能 | 在 `workspace/skills/` 下添加 `SKILL.md` |
| 挂载 MCP Server | 在 `config.json` 的 `mcpServers` 添加服务地址 |
| 自定义记忆结构 | 修改 `MemoryStore` 文件集合和 `ContextBuilder` 组装逻辑 |

---

*文档生成时间：2026-04-19*
