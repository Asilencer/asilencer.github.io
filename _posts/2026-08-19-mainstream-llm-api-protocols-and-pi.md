---
layout: post
title: '主流大模型 API 协议：从 OpenAI、Claude、Gemini 到 DeepSeek、GLM、Kimi 与 Qwen'
date: 2026-08-19 20:51:21 +0800
categories: [AI]
tags: [LLM, API, OpenAI, Anthropic, Gemini, DeepSeek, GLM, Kimi, Qwen, Pi]
pin: false
excerpt: "从消息、流式事件、工具调用和思考上下文出发，比较国内外主流大模型 API 协议，并沿着 Pi 源码追踪一次请求如何被统一、路由和还原。"
---

接入大模型时，最容易混淆的不是某个字段，而是三个层次：厂商、模型和协议。

<strong>Chat Completions 不是一家模型厂商</strong>。它是 OpenAI 定义的一套文本生成 API 形式，核心端点是 <code>POST /v1/chat/completions</code>。由于大量平台兼容了这套形式，人们才会说某个服务“兼容 OpenAI”或“提供 Chat Completions 接口”。

反过来，一家厂商也不一定只有一套协议。仅讨论通用文本生成和 Agent 场景，OpenAI 就同时维护 Chat Completions 与 Responses；通义千问同时提供 OpenAI 兼容的 Chat Completions、OpenAI 兼容的 Responses、Anthropic 兼容的 Messages，以及原生 DashScope。模型厂商与 API 协议不是一一对应关系。

本文以 2026 年 8 月 19 日的公开文档为准，重点讨论消息、流式输出、思考内容和工具调用。Embedding、图像、语音、Realtime、Batch 等独立 API 不在范围内。文中的模型名称容易变化，协议结构比某个具体模型 ID 更值得长期关注。

## 先分清四个概念

| 概念 | 回答的问题 | 示例 |
| --- | --- | --- |
| 厂商 Provider | 请求发给谁，谁负责鉴权和计费？ | OpenAI、Anthropic、Google、DeepSeek、智谱、月之暗面、阿里云 |
| 模型 Model | 实际运行哪个模型？ | GPT、Claude、Gemini、DeepSeek、GLM、Kimi、Qwen 系列 |
| 协议 Protocol | HTTP 请求和响应长什么样？ | OpenAI Responses、OpenAI Chat Completions、Anthropic Messages、Gemini GenerateContent |
| SDK | 用什么客户端构造和解析协议？ | OpenAI SDK、Anthropic SDK、Google Gen AI SDK、厂商自有 SDK |

同一个模型可能通过多个平台和协议被调用；同一个协议也可能承载多个厂商的模型。一个 OpenAI SDK 实例指向 DeepSeek 的 Base URL，调用的仍然是 DeepSeek 模型，只是请求外形采用了 OpenAI Chat Completions。

![主流模型厂商与 API 协议的多对多关系](/assets/img/llm-api-protocol-map.svg)

这张图表达了两个关键事实：

1. OpenAI、Anthropic 和 Google 都有自己的原生协议。
2. DeepSeek、GLM、Kimi、Qwen 等国内平台大量提供兼容协议，但兼容通常是“主要结构兼容”，不是“所有语义完全相同”。

## 一套模型协议到底规定了什么

只看请求 URL，很难理解协议。对聊天和 Agent 应用，至少要检查下面八个部分：

| 层面 | 需要确认的内容 |
| --- | --- |
| 端点 | 路径、版本、区域、是否区分流式端点 |
| 鉴权 | Bearer Token、专用请求头、API 版本头 |
| 输入 | system 指令放在哪里，历史消息如何表达，多模态内容如何分块 |
| 输出 | 文本、思考、工具调用、拒绝和引用如何表示 |
| 流式 | SSE 事件是通用 chunk，还是有明确的事件类型和生命周期 |
| 工具 | 工具 Schema、调用 ID、参数增量、结果回传格式 |
| 状态 | 客户端是否必须重放完整历史，服务端能否用响应 ID 延续上下文 |
| 用量与错误 | Token 分类、缓存命中、停止原因、限流和协议错误 |

主流协议的骨架可以先用一张表概括：

| 协议 | 主要端点 | 核心输入 | 核心输出 | 流式特征 | 对话状态 |
| --- | --- | --- | --- | --- | --- |
| OpenAI Responses | <code>/v1/responses</code> | <code>instructions</code>、<code>input</code>、<code>tools</code> | <code>output[]</code> 类型化条目 | <code>response.output_text.delta</code> 等类型化事件 | 可重放输入，也可用 <code>previous_response_id</code> |
| OpenAI Chat Completions | <code>/v1/chat/completions</code> | <code>messages[]</code>、<code>tools[]</code> | <code>choices[].message</code> | <code>choices[].delta</code>，通常以 <code>[DONE]</code> 收尾 | 客户端重放消息历史 |
| Anthropic Messages | <code>/v1/messages</code> | 顶层 <code>system</code>、<code>messages[]</code>、<code>tools[]</code> | <code>content[]</code> 内容块 | 有名称的 SSE 事件和内容块生命周期 | 客户端重放消息历史 |
| Gemini GenerateContent | <code>:generateContent</code> | <code>systemInstruction</code>、<code>contents[].parts[]</code> | <code>candidates[].content.parts[]</code> | 独立 <code>:streamGenerateContent</code> 端点，可用 SSE | 客户端重放 <code>contents</code> |

下面逐一展开。

## OpenAI：为什么会同时有 Responses 和 Chat Completions

“OpenAI 有两套 API”需要加一个范围限定：OpenAI 的完整平台当然不止两个端点；但在通用文本生成、推理和工具调用范围内，Responses 与 Chat Completions 是最常遇到的两套接口。

### Chat Completions：以消息数组为中心

Chat Completions 的核心抽象是“给定一组消息，补出 assistant 的下一条消息”。

~~~json
POST /v1/chat/completions
Authorization: Bearer $OPENAI_API_KEY
Content-Type: application/json

{
  "model": "MODEL_ID",
  "messages": [
    {"role": "system", "content": "回答要简洁"},
    {"role": "user", "content": "北京今天多少度？"}
  ],
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_weather",
      "description": "查询天气",
      "parameters": {
        "type": "object",
        "properties": {
          "city": {"type": "string"}
        },
        "required": ["city"]
      }
    }
  }],
  "stream": true
}
~~~

非流式文本位于 <code>choices[0].message.content</code>。如果模型要调用工具，assistant 消息会携带 <code>tool_calls</code>，其中 <code>function.arguments</code> 是 JSON 字符串，而不是已经验证过的对象。应用必须解析并校验参数，再用 <code>role: "tool"</code>、<code>tool_call_id</code> 把执行结果送回下一次请求。

流式响应通常是一系列 SSE 数据帧。文本从 <code>choices[0].delta.content</code> 累积；工具名和参数也可能被拆成多个 delta；最后根据 <code>finish_reason</code> 判断是正常结束、达到长度限制，还是等待工具调用。

这套协议结构直观、生态最大，因此成为事实上的兼容层。它的代价是所有输出都被压在 <code>choices</code> 与 <code>delta</code> 结构中，处理复杂推理、多种工具和更多类型化输出时，扩展空间不如 Responses 清晰。

### Responses：以类型化输入输出项为中心

Responses 不再把一切都理解成一条聊天消息，而是把输入和输出拆成不同类型的 item：

~~~json
POST /v1/responses
Authorization: Bearer $OPENAI_API_KEY
Content-Type: application/json

{
  "model": "MODEL_ID",
  "instructions": "回答要简洁",
  "input": [{
    "role": "user",
    "content": [{
      "type": "input_text",
      "text": "北京今天多少度？"
    }]
  }],
  "tools": [{
    "type": "function",
    "name": "get_weather",
    "description": "查询天气",
    "parameters": {
      "type": "object",
      "properties": {
        "city": {"type": "string"}
      },
      "required": ["city"]
    },
    "strict": true
  }],
  "stream": true
}
~~~

输出可能包含 <code>message</code>、<code>reasoning</code>、<code>function_call</code> 等不同 item。工具结果不是一条 <code>role: "tool"</code> 消息，而是：

~~~json
{
  "type": "function_call_output",
  "call_id": "call_xxx",
  "output": "{\"temperature\": 28}"
}
~~~

流式事件也更细，例如：

- <code>response.output_item.added</code>：开始一个输出 item；
- <code>response.output_text.delta</code>：文本增量；
- <code>response.function_call_arguments.delta</code>：工具参数增量；
- <code>response.output_item.done</code>：某个 item 完成；
- <code>response.completed</code>：整个响应完成。

Responses 还可以用 <code>previous_response_id</code> 引用上一轮响应，减少应用手工拼装历史的负担。OpenAI 当前对推理、工具调用和多轮工作流优先推荐 Responses，重要原因之一是它能在轮次之间延续推理上下文。

### 两套接口怎么选

| 场景 | 更合适的选择 |
| --- | --- |
| 新建 OpenAI 原生 Agent、需要推理和多种工具 | Responses |
| 已有大量 Chat Completions 代码 | 可以继续使用，再评估迁移 |
| 需要接入大量 OpenAI 兼容厂商 | Chat Completions 仍是最大公约数 |
| 需要 <code>previous_response_id</code>、类型化事件或托管工具 | Responses |

Responses 不是简单地把 <code>messages</code> 改名为 <code>input</code>。它改变了输出对象、工具回传、流式事件和状态管理方式，因此适配层必须单独实现。

## Anthropic Messages：内容块和事件生命周期

Anthropic 的核心端点是 <code>POST https://api.anthropic.com/v1/messages</code>。典型请求头包括：

~~~http
x-api-key: $ANTHROPIC_API_KEY
anthropic-version: 2023-06-01
content-type: application/json
~~~

它与 OpenAI Chat Completions 有三处结构差异最值得注意。

第一，system 指令位于请求顶层，而不是 <code>messages</code> 中的一条 system 消息。

第二，消息内容天然是块数组。文本、思考、工具调用和工具结果都有各自的块类型：

~~~json
{
  "model": "MODEL_ID",
  "max_tokens": 2048,
  "system": "回答要简洁",
  "messages": [{
    "role": "user",
    "content": [{"type": "text", "text": "北京今天多少度？"}]
  }],
  "tools": [{
    "name": "get_weather",
    "description": "查询天气",
    "input_schema": {
      "type": "object",
      "properties": {
        "city": {"type": "string"}
      },
      "required": ["city"]
    }
  }]
}
~~~

模型调用工具时返回 <code>tool_use</code> 块。应用执行后，在下一条 user 消息中放入 <code>tool_result</code> 块，并用 <code>tool_use_id</code> 关联。

第三，Anthropic 的 SSE 是有名称的状态机事件，而不是只有通用 data chunk：

~~~text
message_start
content_block_start
content_block_delta
content_block_stop
message_delta
message_stop
~~~

文本增量是 <code>text_delta</code>，工具参数可能通过 <code>input_json_delta</code> 分片到达，思考和签名也有独立 delta。解析器需要按 <code>index</code> 维护每个内容块的状态，不能把所有帧都当作字符串拼接。

## Gemini GenerateContent：一切都是 Part

Gemini 原生 REST 入口把模型 ID 放在 URL 中：

~~~http
POST https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent
x-goog-api-key: $GEMINI_API_KEY
content-type: application/json
~~~

它的基本单元不是 message content block，而是 <code>Content.parts</code>：

~~~json
{
  "systemInstruction": {
    "parts": [{"text": "回答要简洁"}]
  },
  "contents": [{
    "role": "user",
    "parts": [{"text": "北京今天多少度？"}]
  }],
  "tools": [{
    "functionDeclarations": [{
      "name": "get_weather",
      "description": "查询天气",
      "parameters": {
        "type": "object",
        "properties": {
          "city": {"type": "string"}
        },
        "required": ["city"]
      }
    }]
  }]
}
~~~

返回值位于 <code>candidates[].content.parts[]</code>。工具请求是 <code>functionCall</code> part，结果是 <code>functionResponse</code> part。流式调用使用独立的 <code>:streamGenerateContent</code> 方法，REST 场景可增加 <code>alt=sse</code>。

Gemini 还有一个容易被通用适配层破坏的字段：<code>thoughtSignature</code>。它是模型生成的不可解释签名，可能附着在思考、文本或工具调用 part 上。后续请求需要原样带回；即使可见文本为空，也不能轻易丢掉携带签名的 part。

## 国内主流厂商：大多兼容 OpenAI，但都有扩展

国内模型平台采用 OpenAI 兼容协议，主要是为了复用 SDK、IDE 和 Agent 生态。真正接入时仍要逐项核对，尤其是思考模式、工具调用、最大输出字段和流式用量。

| 厂商 | 典型入口 | 主要协议形态 | 关键扩展或差异 |
| --- | --- | --- | --- |
| DeepSeek | <code>https://api.deepseek.com/chat/completions</code> | OpenAI Chat Completions 兼容，也提供 Anthropic 兼容入口 | <code>thinking</code>、<code>reasoning_effort</code>、<code>reasoning_content</code>；严格工具模式位于 Beta 入口 |
| 智谱 GLM | <code>https://open.bigmodel.cn/api/paas/v4/chat/completions</code> | OpenAI Chat Completions 风格 | <code>thinking</code>、<code>clear_thinking</code>、<code>reasoning_content</code>、<code>tool_stream</code> |
| Kimi / Moonshot | <code>https://api.moonshot.cn/v1/chat/completions</code> | OpenAI Chat Completions 兼容 | <code>partial</code>、<code>thinking.keep</code>、<code>reasoning_content</code>、<code>cached_tokens</code> |
| 通义千问 / 百炼 | 地域或 Workspace 专属 Base URL | OpenAI Chat、OpenAI Responses、Anthropic Messages、原生 DashScope | 四套接口能力不完全相同，地域 URL 和支持参数需要按文档选择 |

### DeepSeek：兼容结构之外，必须保存思考上下文

DeepSeek 的消息角色、<code>choices</code>、<code>tool_calls</code> 和 SSE chunk 基本沿用 Chat Completions。思考模式增加：

~~~json
{
  "thinking": {"type": "enabled"},
  "reasoning_effort": "high"
}
~~~

思考内容与最终答案平级，分别位于 <code>reasoning_content</code> 和 <code>content</code>。流式响应也分别通过 <code>delta.reasoning_content</code> 与 <code>delta.content</code> 到达。

最重要的规则发生在工具调用之后：如果某轮思考产生了工具调用，后续请求必须完整回传该 assistant 消息的 <code>reasoning_content</code>。遗漏会导致 400。严格遵循 JSON Schema 的工具调用目前需要使用 Beta Base URL，并为所有函数设置 <code>strict: true</code>。

因此，“能用 OpenAI SDK 发出请求”不等于“只替换 Base URL 就完成了可靠适配”。如果中间层只保存可见答案和工具调用，丢掉 <code>reasoning_content</code>，普通聊天可能正常，Agent 多步工具链却会失败。

### 智谱 GLM：交错思考与保留式思考

GLM 的标准对话补全同样采用 <code>messages</code>、<code>tools</code>、<code>tool_calls</code> 和 <code>role: "tool"</code>。鉴权是 Bearer Token，流式输出使用 SSE。

它在思考上下文上提供了更细的控制：

~~~json
{
  "thinking": {
    "type": "enabled",
    "clear_thinking": false
  },
  "reasoning_effort": "high"
}
~~~

<code>clear_thinking: false</code> 表示保留式思考，适合 Coding 和 Agent 场景。使用交错思考加工具时，应用要把完整、未修改的 <code>reasoning_content</code> 与工具结果一起回传。部分模型还通过 <code>tool_stream</code> 控制 Function Call 参数是否流式输出。

工具选择目前应按官方文档的能力约束处理。例如文档中的通用 Function Calling 说明只支持 <code>tool_choice: "auto"</code>，不能因为 OpenAI Schema 接受更多枚举值，就假设 GLM 端点也全部接受。

### Kimi：Chat 兼容入口与 Coding 入口不是同一协议

Moonshot 开放平台的标准入口是 OpenAI Chat Completions 兼容协议。它支持：

- <code>tool_calls</code> 与严格工具定义；
- <code>reasoning_content</code>；
- <code>stream_options.include_usage</code>；
- 用量中的 <code>cached_tokens</code>；
- 在最后一条 assistant 消息上设置 <code>partial: true</code>，预填输出前缀；
- <code>prompt_cache_key</code> 等缓存相关字段。

思考模式下，多轮请求需要按模型规则保留历史 <code>reasoning_content</code>。SSE 的总体外形仍是 <code>chat.completion.chunk</code>，通常以 <code>data: [DONE]</code> 结束。

但 Kimi For Coding 是另一个接入面。它面向编码订阅场景，可以采用 Anthropic Messages 兼容协议。这意味着“Kimi 使用哪套协议”没有脱离具体入口的唯一答案：Moonshot 标准 API 通常走 OpenAI Chat 兼容，Coding 订阅入口可以走 Anthropic Messages 兼容。

### Qwen：多协议平台最典型的例子

阿里云百炼当前为文本生成明确列出四种接口：

1. OpenAI 兼容 Chat Completions；
2. OpenAI 兼容 Responses；
3. Anthropic 兼容 Messages；
4. 原生 DashScope。

选择兼容协议可以复用既有客户端；选择原生协议通常能获得更完整的参数和能力。即使选择 Responses，百炼文档也明确说明：它只处理文档列出的参数，未列出的 OpenAI 参数可能被忽略，部分能力和行为与 OpenAI 原生 Responses 不同。

此外，百炼越来越多地使用地域和 Workspace 专属域名。生产配置不能只复制一个网上常见 Base URL，还要确认 API Key 所属地域、Workspace、协议路径和模型可用范围。

## 工具调用：四套协议描述的是同一个闭环

不同协议的字段名差异很大，但应用层真正需要完成的是同一个五步循环：

![大模型工具调用闭环](/assets/img/llm-tool-call-roundtrip.svg)

对应关系如下：

| 语义 | OpenAI Chat | OpenAI Responses | Anthropic Messages | Gemini |
| --- | --- | --- | --- | --- |
| 系统指令 | system/developer 消息 | <code>instructions</code> | 顶层 <code>system</code> | <code>systemInstruction</code> |
| 工具定义 | <code>tools[].function.parameters</code> | <code>tools[].parameters</code> | <code>tools[].input_schema</code> | <code>functionDeclarations[].parameters</code> |
| 模型请求工具 | <code>assistant.tool_calls[]</code> | <code>function_call</code> item | <code>tool_use</code> block | <code>functionCall</code> part |
| 调用参数 | JSON 字符串 | JSON 字符串及其 delta | 对象，流中可能是 partial JSON | 对象 |
| 工具结果 | <code>role: tool</code> + <code>tool_call_id</code> | <code>function_call_output</code> + <code>call_id</code> | user 中的 <code>tool_result</code> + <code>tool_use_id</code> | <code>functionResponse</code> |
| 停止信号 | <code>finish_reason: tool_calls</code> | output item 与响应状态 | <code>stop_reason: tool_use</code> | <code>finishReason</code> 与 part 类型 |

工具参数来自模型输出，即使开启了 Schema 约束，也不应直接执行。应用至少要做三件事：按调用 ID 拼完参数、验证 JSON 与业务 Schema、检查工具权限。工具执行结果还必须与原调用 ID 精确关联，否则下一轮模型无法判断哪个结果属于哪个请求。

## 流式协议不是“把字符串一点点返回”

对纯文本演示，把 SSE 看成字符流似乎够用；一旦加入思考和工具调用，这种实现很快失效。

一个可靠的流解析器至少要维护：

1. 当前响应是否已经开始；
2. 当前 content item 或 block 的索引和类型；
3. 文本、思考和工具参数各自的累积缓冲区；
4. 工具调用 ID、名称和 partial JSON；
5. 最终停止原因和 usage；
6. 错误、中断与不完整输出状态。

例如 Anthropic 的 <code>input_json_delta</code> 和 OpenAI 的 <code>function_call_arguments.delta</code> 都可能从任意字符边界切开 JSON。每个 delta 到达时直接 <code>JSON.parse</code> 并不可靠，必须先累积，完成后再做最终解析与校验。

这也是统一事件层的价值：UI 和 Agent 循环不必理解每家 SSE 的细节，只接收“文本开始、文本增量、工具调用完成、整个响应完成”这样的稳定事件。

## Pi 如何统一这些协议

下面结合 Pi 源码追踪一次请求。代码分析固定在仓库
[earendil-works/pi 的 58302d3 提交](https://github.com/earendil-works/pi/tree/58302d34e703e0453ea13bdd10c7e423589ce177)，
避免后续重构导致行号和结论漂移。

![Pi 从 Agent 消息到厂商协议的请求链路](/assets/img/pi-llm-request-flow.svg)

Pi 没有强迫所有厂商在 HTTP 层长得一样。它在厂商协议之上定义内部消息和事件，再由每个适配器负责双向翻译。

### 第一层：统一内部语义，而不是统一厂商 JSON

<code>packages/ai/src/types.ts</code> 定义了稳定的领域模型：

~~~ts
interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}

type Message =
  | UserMessage
  | AssistantMessage
  | ToolResultMessage;
~~~

assistant 内容进一步拆成 <code>text</code>、<code>thinking</code> 和 <code>toolCall</code>。工具结果是独立的 <code>toolResult</code> 消息。用量也被归一成输入、输出、缓存读取、缓存写入、推理 Token 和成本。

更关键的是统一的
[<code>AssistantMessageEvent</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/types.ts#L509-L539)：

~~~text
start
text_start → text_delta → text_end
thinking_start → thinking_delta → thinking_end
toolcall_start → toolcall_delta → toolcall_end
done | error
~~~

上层 TUI、日志和 Agent 循环只依赖这套事件。OpenAI 的 <code>response.output_text.delta</code>、Anthropic 的 <code>text_delta</code> 和 Gemini part 文本，最终都会变成同一种 <code>text_delta</code>。

### 第二层：Agent 消息先变成 LLM 上下文

Coding Agent 还包含 shell 执行、压缩摘要、自定义消息等上层类型。它们先由
[<code>convertToLlm()</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/coding-agent/src/core/messages.ts#L140-L180)
转换成 <code>pi-ai</code> 能理解的消息。

随后
[<code>streamAssistantResponse()</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/agent/src/agent-loop.ts#L281-L335)
完成这条链：

~~~ts
let messages = context.messages;
messages = await transformContext(messages);
const llmMessages = await convertToLlm(messages);

const llmContext = {
  systemPrompt: context.systemPrompt,
  messages: llmMessages,
  tools: context.tools,
};

const response = await streamFunction(model, llmContext, options);
~~~

这里有意把“Agent 的长期上下文管理”和“厂商 HTTP 协议转换”分开。摘要、裁剪和业务消息转换发生在前者；system、messages、contents 或 input 的具体组装发生在后者。

### 第三层：Provider 与 API 实现是两个对象

Pi 的 Provider 负责：

- 厂商 ID 与显示名称；
- Base URL；
- API Key 或 OAuth；
- 模型目录；
- 应该使用哪一个 API 实现。

API 实现负责：

- 把内部 <code>Context</code> 转成厂商请求；
- 调用 SDK；
- 把厂商流解析成统一事件；
- 统一停止原因、用量和错误。

[<code>createProvider()</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/models.ts#L739-L792)
既支持一个 Provider 共享单一协议实现，也支持按 <code>model.api</code> 分派到多个实现。这正是“厂商”和“协议”解耦在代码中的落点。

### 第四层：每套原生协议有独立适配器

| Pi API ID | 主要适配器 | 关键职责 |
| --- | --- | --- |
| <code>openai-responses</code> | [<code>openai-responses.ts</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/api/openai-responses.ts#L103-L329) | 构造 Responses input/tools，解析类型化 response 事件 |
| <code>openai-completions</code> | [<code>openai-completions.ts</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/api/openai-completions.ts#L202-L245) | 构造 Chat messages，解析 <code>choices[].delta</code>，处理兼容厂商差异 |
| <code>anthropic-messages</code> | [<code>anthropic-messages.ts</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/api/anthropic-messages.ts#L502-L734) | 维护内容块生命周期，解析文本、思考、签名和 partial JSON |
| <code>google-generative-ai</code> | [<code>google-generative-ai.ts</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/api/google-generative-ai.ts#L52-L235) | 解析 Gemini parts、functionCall、thoughtSignature、usage |

OpenAI Responses 适配器有一个值得注意的选择：它设置 <code>store: false</code>，由 Pi 重放自己的完整上下文，而不是依赖服务端保存的上一轮状态。这样做让会话持久化、分支、切换模型和跨厂商回放仍由 Pi 控制；代价是适配器必须正确保存并重放推理签名等隐藏状态。

Responses 的函数调用还可能同时拥有 <code>call_id</code> 和输出 item ID。Pi 在内部组合这两个标识，回传时再拆开，避免把“调用关联 ID”和“输出条目 ID”混为一谈。相关逻辑位于
[<code>openai-responses-shared.ts</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/api/openai-responses-shared.ts#L488-L525)。

Anthropic 适配器则按照 <code>content_block_start</code>、<code>content_block_delta</code>、<code>content_block_stop</code> 更新当前块。遇到 <code>input_json_delta</code> 时，它先累积工具参数，再在块结束时生成统一 <code>ToolCall</code>。

Gemini 转换器会特别保留 <code>thoughtSignature</code>。即使文本为空，只要携带签名，part 就不能丢弃；否则模型的推理链可能在后续工具轮次中断。对应实现见
[<code>google-shared.ts</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/api/google-shared.ts#L133-L180)。

### 第五层：OpenAI 兼容不是一个布尔值

如果兼容性只有“是”或“否”，DeepSeek、GLM、Kimi 和 Qwen 都可以共用一个简单 Base URL 配置。Pi 的实现不是这样。

[<code>OpenAICompletionsCompat</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/types.ts#L541-L620)
把差异拆成多项能力，例如：

- 是否支持 <code>store</code>；
- 是否支持 developer role；
- 最大输出字段是 <code>max_tokens</code> 还是 <code>max_completion_tokens</code>；
- 流式响应是否包含 usage；
- 工具是否支持 <code>strict</code>；
- 工具结果是否必须带名称；
- 思考开关采用 OpenAI、DeepSeek、Z.AI、Qwen 还是其他格式；
- 是否要求 assistant 历史始终携带 <code>reasoning_content</code>。

[<code>detectCompat()</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/api/openai-completions.ts#L1450-L1539)
会依据 provider ID 与 Base URL 识别 DeepSeek、Z.AI 和 Moonshot 等端点。以当前代码为例：

- DeepSeek 使用 <code>thinking: {type}</code>，并要求回放 assistant 的 <code>reasoning_content</code>；
- Z.AI 使用自己的 <code>thinking</code> 形式；
- DeepSeek、Z.AI、Moonshot 使用 <code>max_tokens</code>；
- 非标准端点默认不会收到 OpenAI 专属的 <code>store</code> 和 developer role。

源码快照和最新厂商文档之间还暴露出一个现实问题：协议能力会持续变化。本文核对的 Kimi Chat 文档已经把 <code>max_tokens</code> 标为弃用、推荐 <code>max_completion_tokens</code>，工具定义中也列出了 <code>strict</code>；而这个 Pi 提交中的 Moonshot 回退规则仍选择 <code>max_tokens</code>，并默认关闭 strict。模型目录中的显式 <code>compat</code> 可以覆盖 URL 推断，但生产接入仍应以目标模型的实测和当期文档为准。适配器不是写完一次就永久正确，兼容元数据也需要随上游协议演进。

这比“捕获 400 后删除一个字段再试”更可靠，因为兼容规则是显式、可检查的模型元数据和适配策略。

### 国内厂商在 Pi 中的实际映射

Pi 的 Provider factory 很短，但信息密度很高：

| Pi Provider | Base URL | API 实现 |
| --- | --- | --- |
| [DeepSeek](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/providers/deepseek.ts#L6-L14) | <code>https://api.deepseek.com</code> | <code>openai-completions</code> |
| [Moonshot AI CN](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/providers/moonshotai-cn.ts#L6-L14) | <code>https://api.moonshot.cn/v1</code> | <code>openai-completions</code> |
| [Kimi For Coding](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/providers/kimi-coding.ts#L7-L23) | <code>https://api.kimi.com/coding</code> | <code>anthropic-messages</code> |
| [Z.AI Coding CN](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/providers/zai-coding-cn.ts#L6-L14) | <code>https://open.bigmodel.cn/api/coding/paas/v4</code> | <code>openai-completions</code> |
| [Qwen Token Plan CN](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/providers/qwen-token-plan-cn.ts#L6-L14) | Token Plan OpenAI 兼容入口 | <code>openai-completions</code> |

这里必须避免过度概括：

- Pi 的 Z.AI 内置项面向 Coding Plan 端点，不等于智谱所有标准 API 都固定使用这一个 URL；
- Pi 的 Qwen 内置项面向 Token Plan，并没有在这里把百炼的四套协议全部注册一遍；
- Moonshot 标准 API 与 Kimi For Coding 是两个 Provider，协议不同；
- 任何新的兼容端点仍应按它实际支持的字段配置，而不是只改厂商名称。

### 第六层：跨模型切换时，隐藏状态不能乱带

思考签名通常只对产生它的厂商、协议和模型有效。把 Gemini 的 <code>thoughtSignature</code> 发送给 Anthropic，或者把 OpenAI 的加密 reasoning item 发送给另一个模型，轻则被忽略，重则请求失败。

Pi 的
[<code>transformMessages()</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/src/api/transform-messages.ts#L64-L180)
使用 provider、api 和 model 三者判断是否仍是同一个模型：

- 同模型回放时保留不透明签名；
- 跨模型时丢弃不可复用的 redacted thinking；
- 可见思考文本降级为普通文本；
- 移除 Gemini 工具调用上的 <code>thoughtSignature</code>；
- 规范化不被目标协议接受的工具调用 ID；
- 如果历史中存在孤立的工具调用，补一个明确失败的合成工具结果，保持消息序列合法。

这段逻辑体现了一个通用原则：统一层应该保存足够多的厂商元数据，但只能在有效范围内重放它。

### 第七层：Agent 循环只处理 ToolCall 和 ToolResult

适配器完成流解析后，Agent 不再关心上游是 <code>tool_calls</code>、<code>tool_use</code> 还是 <code>functionCall</code>。

[Agent 主循环](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/agent/src/agent-loop.ts#L192-L224)
从 assistant 内容中筛出统一 <code>toolCall</code>。随后它：

1. 找到本地工具；
2. 验证参数；
3. 执行安全钩子；
4. 根据工具声明选择顺序或并行执行；
5. 生成统一 <code>ToolResultMessage</code>；
6. 把结果加入上下文，再进入下一轮模型请求。

参数验证与阻断逻辑位于
[<code>prepareToolCall()</code>](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/agent/src/agent-loop.ts#L600-L668)。
如果模型因为长度上限截断了工具参数，主循环不会尝试执行残缺调用，而是把整批调用标记为失败。

于是一次 DeepSeek 天气工具调用的完整路径是：

~~~text
AgentMessage[]
  → convertToLlm()
  → Context
  → DeepSeek Provider
  → openai-completions buildParams()
  → POST /chat/completions
  → delta.reasoning_content / delta.tool_calls
  → thinking_delta / toolcall_delta / toolcall_end
  → 参数验证与本地工具执行
  → ToolResultMessage
  → role: tool + tool_call_id
  → 连同 assistant reasoning_content 发起下一轮
  → 最终 text_delta / done
~~~

这条链路同时解释了为什么统一接口不能只保存最终文本：Agent 的正确性依赖工具 ID、参数分片、停止原因、思考签名和工具结果的完整往返。

## 设计多厂商接入层时，应该学 Pi 的什么

第一，内部模型应描述业务语义，而不是复制某一家 JSON。<code>ToolCall</code> 和 <code>ToolResult</code> 是稳定语义，<code>tool_calls</code>、<code>tool_use</code>、<code>functionCall</code> 只是线上的不同编码。

第二，Provider 与协议实现要解耦。鉴权、Base URL、模型目录属于厂商；消息转换和流解析属于协议。这样才能表达“一家厂商多种协议”和“多家厂商共用兼容协议”。

第三，流式输出应归一成有生命周期的事件，而不是只暴露文本回调。工具参数、思考内容和错误都需要独立事件。

第四，兼容性应是能力矩阵，不是单个布尔值。字段名称、严格 Schema、usage、思考格式和消息角色都可能不同。

第五，不透明签名也是上下文。看不懂不代表可以丢掉，更不代表可以跨模型复制。

第六，工具调用必须由 Agent 循环闭环。模型只提出调用；参数校验、权限、执行、错误编码和结果回传都属于应用责任。

## 实际选型建议

如果只接 OpenAI 且新建复杂 Agent，优先从 Responses 开始；如果重点是跨厂商覆盖，OpenAI Chat Completions 仍然是最实用的公共入口，但需要兼容性配置。

如果深度使用 Claude 的思考、内容块和工具流，直接实现 Anthropic Messages，通常比经由兼容代理更清晰。Gemini 的多模态 Part 与 thoughtSignature 也值得使用原生适配器，不宜强行压成纯文本 Chat。

接入 DeepSeek、GLM、Kimi 或 Qwen 时，至少用真实流式工具调用验证下面这份清单：

- Base URL、地域和 API Key 是否匹配；
- system/developer role 是否支持；
- 最大输出字段用哪一个；
- 思考开关和 effort 如何表达；
- <code>reasoning_content</code> 是否必须回放；
- 工具参数是否流式、是否保证 JSON Schema；
- <code>tool_choice</code> 支持哪些值；
- usage 在哪个 chunk 出现；
- 缓存 Token 如何统计；
- 中断、长度截断和内容过滤分别返回什么停止原因。

只验证一次“你好”远远不够。最小的协议验收用例应包含：多轮对话、流式文本、单工具、并行工具、错误工具结果、思考加工具、长参数分片、主动中断和跨模型恢复。

## 结语

大模型 API 的真正难点不是发出一个 HTTP 请求，而是在多轮、流式、思考和工具调用中保持语义完整。

Chat Completions 是 OpenAI 定义并被广泛兼容的一套协议，不是厂商；Responses 是 OpenAI 面向新一代推理和 Agent 工作流的另一套协议；Anthropic Messages 与 Gemini GenerateContent 又各自建立了内容块和 Part 模型。DeepSeek、GLM、Kimi 和 Qwen 降低了迁移成本，但也通过思考、缓存、工具和多入口扩展了兼容协议。

Pi 的实现给出了一个清楚的工程答案：先建立稳定的内部消息、工具和事件模型，再把每套厂商协议放在边缘翻译。这样，上层 Agent 只处理“模型说了什么、要调用什么工具、工具返回了什么”，而不必在业务循环里到处判断 <code>choices</code>、<code>content_block</code> 或 <code>parts</code>。

理解这层边界之后，面对任何新厂商都可以先问三个问题：它真正实现了哪套协议，扩展或删减了哪些语义，现有适配层能否完整保存一次 Agent 往返所需的状态。答案比“是否兼容 OpenAI”更有价值。

## 参考资料

- [OpenAI：模型与 Responses 迁移指南](https://developers.openai.com/api/docs/guides/latest-model)
- [OpenAI：Chat API Reference](https://developers.openai.com/api/reference/resources/chat)
- [Anthropic：Messages API](https://platform.claude.com/docs/en/api/messages/create)
- [Anthropic：Streaming Messages](https://platform.claude.com/docs/en/build-with-claude/streaming)
- [Anthropic：API Versioning](https://platform.claude.com/docs/en/api/versioning)
- [Google：GenerateContent 与 StreamGenerateContent](https://ai.google.dev/api/generate-content)
- [Google：Function Calling](https://ai.google.dev/gemini-api/docs/function-calling)
- [Google：Thought Signatures](https://ai.google.dev/gemini-api/docs/thought-signatures)
- [DeepSeek：Chat Completion](https://api-docs.deepseek.com/api/create-chat-completion)
- [DeepSeek：Thinking Mode](https://api-docs.deepseek.com/guides/thinking_mode)
- [DeepSeek：Tool Calls](https://api-docs.deepseek.com/guides/tool_calls)
- [智谱：对话补全](https://docs.bigmodel.cn/api-reference/%E6%A8%A1%E5%9E%8B-api/%E5%AF%B9%E8%AF%9D%E8%A1%A5%E5%85%A8)
- [智谱：思考模式](https://docs.bigmodel.cn/cn/guide/capabilities/thinking-mode)
- [智谱：工具调用](https://docs.bigmodel.cn/cn/guide/capabilities/function-calling)
- [Kimi：创建对话补全](https://platform.kimi.com/docs/api/chat)
- [Kimi：工具调用](https://platform.kimi.com/docs/api/tool-use)
- [阿里云百炼：文本生成模型 API 参考](https://help.aliyun.com/zh/model-studio/qwen-api-reference)
- [阿里云百炼：OpenAI 兼容 Responses](https://help.aliyun.com/zh/model-studio/qwen-api-via-openai-responses)
- [阿里云百炼：Anthropic 兼容 Messages](https://help.aliyun.com/zh/model-studio/anthropic-api-messages)
- [Pi：<code>packages/ai</code> 文档](https://github.com/earendil-works/pi/blob/58302d34e703e0453ea13bdd10c7e423589ce177/packages/ai/README.md)
