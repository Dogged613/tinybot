# Day 2 — Provider 层深度实战

> 目标：理解 nanobot 是如何把“不同大模型厂商的 API”统一成同一种调用方式的。
> 预计时间：3-4 小时。
> 学完结果：你能说清楚 Provider 抽象、注册表分发、重试策略、消息格式转换分别在解决什么问题。

---

## 一、先纠正 Day 1 里的一个预告（5 分钟）

`day01.md` 末尾写的是：

- 精读 `AnthropicProvider`
- 精读 `_run_with_retry`
- 仿照现有代码给 Ollama 写一个最简 Provider 骨架

前两条仍然成立，但第三条对当前仓库已经不完全适用。

**原因：**

现在仓库里的 Ollama 不是一个单独的 `OllamaProvider` 类，而是通过注册表声明为：

- `backend="openai_compat"`
- `default_api_base="http://localhost:11434/v1"`

也就是说，**Ollama 复用了 `OpenAICompatProvider` 这一套实现**。

所以今天更合理的目标不是“再写一个 OllamaProvider”，而是：

1. 看懂 Provider 抽象基类
2. 看懂 Provider 注册表如何选择具体实现
3. 看懂 `AnthropicProvider` 和 `OpenAICompatProvider` 分别怎么适配不同 API
4. 看懂统一重试机制为什么要放在基类里

---

## 二、今天要回答的核心问题

学完今天，你至少要能用自己的话回答这 4 个问题：

1. **为什么项目需要 Provider 抽象层？**
2. **同样是 `chat()`，为什么 Anthropic 和 OpenAI-Compatible 内部实现完全不同？**
3. **为什么重试逻辑不交给各家 SDK，而是统一放到 `LLMProvider` 基类？**
4. **为什么 Ollama、vLLM、OpenRouter 这些不同来源，最后很多都能复用同一个 `OpenAICompatProvider`？**

---

## 三、今天的阅读顺序

不要乱跳，按这个顺序读：

### 1. `nanobot/providers/base.py`

先看这几个对象：

- `ToolCallRequest`
- `LLMResponse`
- `GenerationSettings`
- `LLMProvider`

你要重点理解：

- `chat()` 是统一接口，具体厂商自己实现
- `LLMResponse` 把各家返回值统一成项目内部格式
- 错误信息不只是字符串，还会带：
  - `error_status_code`
  - `error_kind`
  - `error_type`
  - `error_code`
  - `error_retry_after_s`
  - `error_should_retry`

这说明项目不是“报错了就重试”，而是做了**结构化错误治理**。

### 2. `nanobot/providers/registry.py`

再看 `ProviderSpec` 和 `PROVIDERS`。

你要重点理解：

- `ProviderSpec` 只是**元数据声明**
- 它不负责真的发请求
- 它决定：
  - provider 名字
  - 识别关键词
  - 用哪个 backend
  - 默认 API Base
  - 是否本地模型 / 网关 / OAuth

你会发现很多 provider 实际并没有单独写类，而是都走：

- `backend="openai_compat"`

这就是一个非常典型的工程化设计：

**把“差异大的地方单独实现，把差异小的地方尽量收敛到一个通用后端”。**

### 3. `nanobot/providers/anthropic_provider.py`

重点看 4 件事：

- `_convert_messages()`
- `_assistant_blocks()`
- `_convert_tools()`
- `chat()`

Anthropic 最大的学习点不是“怎么调 SDK”，而是：

**项目内部消息格式 和 Anthropic Messages API 格式并不一样，所以必须做转换。**

比如：

- OpenAI 风格里有 `role=tool`
- Anthropic 里工具结果要变成 `tool_result`
- assistant 的工具调用要变成 `tool_use`
- 连续相同 role 的消息还要 merge

也就是说，`AnthropicProvider` 的本质工作是：

**把项目内部统一格式翻译成 Claude 能接受的格式，再把 Claude 的返回翻译回来。**

### 4. `nanobot/providers/openai_compat_provider.py`

重点看：

- `_sanitize_messages()`
- `_build_kwargs()`
- `_should_use_responses_api()`
- `chat()`

你要观察到：

- OpenAI-compatible 不是只支持 OpenAI
- 它其实是一整个“兼容协议家族”
- 包括：
  - OpenAI
  - DeepSeek
  - DashScope
  - Moonshot
  - Ollama
  - vLLM
  - LM Studio
  - OpenRouter
  - 各种兼容 OpenAI 协议的网关

这里最值得学的是：

**同一个 Provider 类内部，还会根据模型、base_url、provider spec 再做二次分流。**

例如：

- 有些模型不能传 `temperature`
- 有些 provider 要传 `max_completion_tokens`
- 有些 provider 支持 prompt caching
- 有些请求会优先走 OpenAI Responses API

这就是“统一抽象下的条件分派”。

### 5. 回头再看 `base.py` 里的重试逻辑

最后重点读：

- `chat_with_retry()`
- `chat_stream_with_retry()`
- `_is_transient_response()`
- `_extract_retry_after_from_response()`
- `_run_with_retry()`

你要带着这个问题读：

**为什么项目宁愿自己控制重试，也不完全依赖 SDK 默认重试？**

答案基本会落在这几点：

- 不同 SDK 的重试行为不一致
- 流式和非流式都需要统一策略
- 429 不一定都该重试
- quota 用尽 和 rate limit 不是一回事
- 项目还要支持“图片失败后自动降级为纯文本重试”

这已经不是“简单 retry”，而是**面向产品稳定性的容错策略**。

---

## 四、你今天真正要看懂的 3 个设计点

### 4.1 Provider 抽象解决的是“统一调用入口”

外层 AgentLoop / AgentRunner 不想知道你底层是：

- Claude
- GPT
- DeepSeek
- Ollama
- OpenRouter

它只想做一件事：

```python
response = await provider.chat_with_retry(...)
```

这就是抽象层的价值：

**上层只依赖能力，不依赖厂商细节。**

### 4.2 注册表解决的是“扩展新厂商时不要到处改代码”

如果没有注册表，新增 provider 可能要改很多地方：

- 配置定义
- 名称匹配
- env 变量处理
- 默认 base_url
- 状态展示
- backend 选择

现在的做法是：

- 大部分元信息写进 `ProviderSpec`
- 业务代码只认统一结构

这是一种典型的：

**声明式扩展，替代分散式硬编码。**

### 4.3 重试基类化解决的是“稳定性策略不能散落在每个 provider 里”

如果每个 provider 各自写重试：

- 行为会不一致
- bug 很难统一修
- 流式 / 非流式容易分叉
- 很难实现统一的回退策略

现在统一到 `LLMProvider` 后：

- 每个 provider 只负责“如何发请求”
- 基类负责“失败后怎么办”

这是非常值得你学的职责分离。

---

## 五、今天的动手练习

今天不建议直接写功能，先做“读代码 + 口头复述 + 小改造理解”。

### 练习 1：画 Provider 调用路径

自己写下来：

```text
AgentLoop
  -> AgentRunner
  -> provider.chat_with_retry()
  -> provider.chat()
  -> SDK / HTTP request
  -> LLMResponse
```

要求：每一层写一句“它负责什么”。

### 练习 2：解释为什么 Ollama 不需要单独 Provider 类

请你用自己的话回答：

- Ollama 和 OpenAI API 哪些地方足够相似？
- 为什么只要 registry 里声明 `backend="openai_compat"` 就够了？
- 这种复用会带来什么好处？

### 练习 3：追一次 429 重试判断

从 `base.py` 出发，回答：

- 哪些 429 会重试？
- 哪些 429 不会重试？
- `retry-after` 可能从哪里提取出来？

如果你能答出来，说明你已经不是“看懂表面代码”，而是看懂稳定性设计了。

### 练习 4：对比 Anthropic 和 OpenAI-Compatible 的消息转换

分别回答：

1. 为什么 `AnthropicProvider` 需要显式转换消息结构？
2. 为什么 `OpenAICompatProvider` 更多是在做 sanitize，而不是彻底重构格式？

这道题能帮你真正理解“统一接口，不代表底层协议相同”。

---

## 六、建议你今天产出的学习笔记

今天学完，建议你写一页自己的笔记，标题就叫：

**《nanobot Provider 层是怎么统一 30+ 模型厂商的》**

笔记至少包含这 5 句：

1. Provider 基类统一了 `chat()` / `chat_with_retry()` 的对外接口。
2. `LLMResponse` 是项目内部的统一返回格式。
3. `ProviderSpec` 是声明式元数据，不负责真实请求。
4. `AnthropicProvider` 的难点在消息格式转换。
5. `OpenAICompatProvider` 的价值在于复用一整类兼容 OpenAI 协议的服务。

---

## 七、今天可以直接回答在面试里的话

你可以这样讲：

> 我在 nanobot 里重点看了 Provider 抽象层。这个项目没有把每个模型厂商都当成独立系统硬写，而是先定义统一的 `LLMProvider` 接口和 `LLMResponse` 数据结构，再通过注册表把不同厂商映射到不同 backend。像 Claude 这种协议差异较大的，就用 `AnthropicProvider` 单独适配；像 Ollama、OpenRouter、DeepSeek 这类兼容 OpenAI 协议的，就复用 `OpenAICompatProvider`。另外，项目把重试逻辑集中放在基类里，统一处理 rate limit、quota、网络抖动和降级重试，这一点很有工程价值。 

---

## 八、今天结束前的自测题

1. `LLMProvider` 和 `ProviderSpec` 的区别是什么？
2. 为什么 `LLMResponse` 里不仅有 `content`，还有 `tool_calls` 和一整套错误字段？
3. 为什么 Ollama 属于本地模型，但还能复用 `openai_compat`？
4. 为什么 `AnthropicProvider` 需要 merge consecutive messages？
5. 为什么 SDK 内置重试不够，还要项目自己做 `_run_with_retry()`？

如果这 5 题你能不看代码回答出 70% 以上，就可以进入 Day 3。

---

## 九、明天预告

**Day 3 — Agent Loop 与一次完整对话是怎么跑起来的**

建议明天读：

- `nanobot/agent/loop.py`
- `nanobot/agent/runner.py`
- `nanobot/agent/context.py`

重点问题：

- 一条用户消息是怎么进入 Agent 的？
- 工具调用是在哪一层执行的？
- Session、Memory、Context 是怎么拼起来的？

