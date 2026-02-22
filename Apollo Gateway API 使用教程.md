# Apollo Gateway API 使用教程

## 基础信息

| 项目 | 值 |
|------|-----|
| 测试环境 | `https://test.apolloinn.site` |
| 生产环境 | `https://api.apolloinn.site` |
| 认证方式 | Bearer Token |
| 协议 | OpenAI Chat Completions API 兼容 |

### 认证方式

所有请求都需要在 Header 中携带 API Key：

```http
Authorization: Bearer <your-api-key>
```

---

## 支持的模型

| 模型 ID | 说明 |
|---------|------|
| `claude-opus-4.6` | Claude Opus 4.6（最强） |
| `claude-sonnet-4.6` | Claude Sonnet 4.6（推荐） |
| `claude-opus-4.5` | Claude Opus 4.5 |
| `claude-sonnet-4.5` | Claude Sonnet 4.5 |
| `claude-sonnet-4` | Claude Sonnet 4 |
| `claude-haiku-4.5` | Claude Haiku 4.5（最快） |
| `auto-kiro` | 自动选择模型 |

> 模型名称不区分大小写，`Claude-Opus-4.6` 会自动转为 `claude-opus-4.6`。

**别名映射**（以下别名同样可用）：

| 别名 | 实际模型 |
|------|---------|
| `kiro-opus-4-6` | claude-opus-4.6 |
| `kiro-sonnet-4-6` | claude-sonnet-4.6 |
| `kiro-opus-4-5` | claude-opus-4.5 |
| `kiro-sonnet-4-5` | claude-sonnet-4.5 |
| `kiro-sonnet-4` | claude-sonnet-4 |
| `kiro-haiku-4-5` | claude-haiku-4.5 |
| `kiro-haiku` | claude-haiku-4.5 |
| `kiro-auto` | auto-kiro |

---

## 接口一：获取模型列表

### `GET /v1/models`

返回当前可用的所有模型。

### 请求示例

**cURL：**
```bash
curl https://api.apolloinn.site/v1/models \
  -H "Authorization: Bearer your-api-key"
```

**Python：**
```python
import httpx

resp = httpx.get(
    "https://api.apolloinn.site/v1/models",
    headers={"Authorization": "Bearer your-api-key"}
)
models = resp.json()
for model in models["data"]:
    print(model["id"])
```

**Python（OpenAI SDK）：**
```python
from openai import OpenAI

client = OpenAI(
    api_key="your-api-key",
    base_url="https://api.apolloinn.site/v1"
)

models = client.models.list()
for model in models.data:
    print(model.id)
```

### 响应示例

```json
{
  "object": "list",
  "data": [
    { "id": "claude-opus-4.6", "object": "model" },
    { "id": "claude-sonnet-4.6", "object": "model" },
    { "id": "claude-haiku-4.5", "object": "model" }
  ]
}
```

---

## 接口二：Chat Completions（Cursor 专用）

### `POST /v1/chat/completions`

专为 **Cursor IDE** 优化的端点，会自动将模型的 `reasoning_content`（思考内容）转换为 `<think>` 标签格式，方便 Cursor 展示。

### 使用场景

- 在 Cursor IDE 中配置 API
- 需要在 Cursor 中看到模型思考过程

### Cursor IDE 配置方法

打开 Cursor Settings → Models，填写：

- **Override OpenAI Base URL**：`https://api.apolloinn.site/v1`
- **API Key**：你的 API Key

Cursor 会自动调用 `/v1/chat/completions`，思考内容会自动转为 `<think>` 标签格式展示。

### 请求参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `model` | string | 是 | - | 模型 ID |
| `messages` | array | 是 | - | 消息列表，格式同 OpenAI |
| `stream` | boolean | 否 | false | 是否流式输出 |
| `tools` | array | 否 | - | 工具定义，格式同 OpenAI Function Calling |

### 请求示例

**cURL：**
```bash
curl -X POST https://api.apolloinn.site/v1/chat/completions \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-4.6",
    "messages": [
      {"role": "user", "content": "你好，请介绍一下自己"}
    ],
    "stream": false
  }'
```

**Python（流式输出）：**
```python
from openai import OpenAI

client = OpenAI(
    api_key="your-api-key",
    base_url="https://api.apolloinn.site/v1"
)

response = client.chat.completions.create(
    model="claude-sonnet-4.6",
    messages=[{"role": "user", "content": "你好！"}],
    stream=True
)

for chunk in response:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

**Python（非流式）：**
```python
from openai import OpenAI

client = OpenAI(
    api_key="your-api-key",
    base_url="https://api.apolloinn.site/v1"
)

response = client.chat.completions.create(
    model="claude-sonnet-4.6",
    messages=[
        {"role": "system", "content": "你是一个有帮助的助手。"},
        {"role": "user", "content": "1+1等于几？"}
    ],
    stream=False
)

print(response.choices[0].message.content)
```

**带 Function Calling（工具调用）：**
```python
from openai import OpenAI
import json

client = OpenAI(
    api_key="your-api-key",
    base_url="https://api.apolloinn.site/v1"
)

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的天气",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "城市名称"}
                },
                "required": ["city"]
            }
        }
    }
]

response = client.chat.completions.create(
    model="claude-sonnet-4.6",
    messages=[{"role": "user", "content": "北京今天天气怎么样？"}],
    tools=tools
)

msg = response.choices[0].message
if msg.tool_calls:
    for tool_call in msg.tool_calls:
        print(f"调用工具: {tool_call.function.name}")
        print(f"参数: {tool_call.function.arguments}")
```

---

## 接口三：Chat Completions（标准 OpenAI 兼容）⭐ 推荐

### `POST /standard/v1/chat/completions`

完全兼容 OpenAI 格式的端点，适合所有应用场景、自定义客户端接入。支持通过 `reasoning_mode` 参数控制思考内容的处理方式。

### 请求参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `model` | string | 是 | - | 模型 ID |
| `messages` | array | 是 | - | 消息列表，格式同 OpenAI |
| `stream` | boolean | 否 | false | 是否流式输出 |
| `tools` | array | 否 | - | 工具定义，格式同 OpenAI Function Calling |
| `reasoning_mode` | string | 否 | `"drop"` | 思考内容处理方式，见下方说明 |

### `reasoning_mode` 参数详解

| 值 | 行为 |
|----|------|
| `"drop"` | **默认**，丢弃思考过程，只返回最终回答 |
| `"reasoning_content"` | 思考内容放在 `reasoning_content` 字段返回，与 OpenAI o1 格式一致 |
| `"content"` | 思考内容用 `<think>...</think>` 标签包裹，拼接到 `content` 字段 |

> **注意**：`reasoning_mode` 是扩展字段，使用 OpenAI SDK 时通过 `extra_body` 传递。

### 请求示例

**cURL（基础对话）：**
```bash
curl -X POST https://api.apolloinn.site/standard/v1/chat/completions \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-4.6",
    "messages": [{"role": "user", "content": "你好！"}],
    "reasoning_mode": "drop",
    "stream": false
  }'
```

**Python（OpenAI SDK，获取思考内容）：**
```python
from openai import OpenAI

client = OpenAI(
    api_key="your-api-key",
    base_url="https://api.apolloinn.site/standard/v1"
)

# 使用 reasoning_content 模式，获取思考过程
response = client.chat.completions.create(
    model="claude-sonnet-4.6",
    messages=[{"role": "user", "content": "25 * 37 等于多少？请一步步计算。"}],
    stream=False,
    extra_body={"reasoning_mode": "reasoning_content"}
)

msg = response.choices[0].message
print("思考过程：", getattr(msg, "reasoning_content", ""))
print("最终答案：", msg.content)
```

**Python（流式 + 思考内容）：**
```python
from openai import OpenAI

client = OpenAI(
    api_key="your-api-key",
    base_url="https://api.apolloinn.site/standard/v1"
)

response = client.chat.completions.create(
    model="claude-sonnet-4.6",
    messages=[{"role": "user", "content": "解释一下量子纠缠"}],
    stream=True,
    extra_body={"reasoning_mode": "reasoning_content"}
)

print("=== 思考过程 ===")
for chunk in response:
    delta = chunk.choices[0].delta
    # 思考内容
    if hasattr(delta, "reasoning_content") and delta.reasoning_content:
        print(delta.reasoning_content, end="", flush=True)
    # 正式回答
    if delta.content:
        print(delta.content, end="", flush=True)
```

**Python（httpx，reasoning_content 模式，非流式）：**
```python
import httpx
import json

resp = httpx.post(
    "https://api.apolloinn.site/standard/v1/chat/completions",
    headers={"Authorization": "Bearer your-api-key"},
    json={
        "model": "claude-sonnet-4.6",
        "messages": [{"role": "user", "content": "What is 25 * 37?"}],
        "reasoning_mode": "reasoning_content",
        "stream": False
    }
)

data = resp.json()
msg = data["choices"][0]["message"]
print("思考过程：", msg.get("reasoning_content", ""))
print("最终答案：", msg["content"])
```

**Python（content 模式，思考内容嵌入正文）：**
```python
import httpx

resp = httpx.post(
    "https://api.apolloinn.site/standard/v1/chat/completions",
    headers={"Authorization": "Bearer your-api-key"},
    json={
        "model": "claude-sonnet-4.6",
        "messages": [{"role": "user", "content": "分析一下这道题的解法"}],
        "reasoning_mode": "content",
        "stream": False
    }
)

data = resp.json()
# content 字段中包含 <think>...</think> 标签
content = data["choices"][0]["message"]["content"]
print(content)
# 输出示例：
# <think>
# 让我思考一下这道题...
# </think>
# 这道题的解法是...
```

**JavaScript / Node.js：**
```javascript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "your-api-key",
  baseURL: "https://api.apolloinn.site/standard/v1",
});

const response = await client.chat.completions.create({
  model: "claude-sonnet-4.6",
  messages: [{ role: "user", content: "Hello!" }],
  stream: true,
});

for await (const chunk of response) {
  const delta = chunk.choices[0]?.delta;
  if (delta?.content) {
    process.stdout.write(delta.content);
  }
}
```

### 响应格式说明

**流式响应（`reasoning_mode: "reasoning_content"`）：**
```
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"role":"assistant","reasoning_content":"让我思考一下..."},"finish_reason":null}]}
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"你好！有什么可以帮你的？"},"finish_reason":null}]}
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}
data: [DONE]
```

**非流式响应（`reasoning_mode: "reasoning_content"`）：**
```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "model": "claude-sonnet-4.6",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "你好！有什么可以帮你的？",
        "reasoning_content": "让我思考一下..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 25,
    "completion_tokens": 50,
    "total_tokens": 75
  }
}
```

---

## 接口四：Anthropic Messages API

### `POST /v1/messages`

原生 Anthropic Messages API 格式端点，适合已经使用 Anthropic SDK 的项目直接迁移。

### 请求参数

与 Anthropic 官方 API 格式完全一致。

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `model` | string | 是 | 模型 ID |
| `messages` | array | 是 | 消息列表 |
| `max_tokens` | integer | 是 | 最大输出 token 数 |
| `system` | string | 否 | 系统提示词 |
| `stream` | boolean | 否 | 是否流式输出 |

### 请求示例

**cURL：**
```bash
curl -X POST https://api.apolloinn.site/v1/messages \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-4.6",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "你好，请介绍一下自己"}
    ]
  }'
```

**Python（Anthropic SDK）：**
```python
import anthropic

client = anthropic.Anthropic(
    api_key="your-api-key",
    base_url="https://api.apolloinn.site"
)

message = client.messages.create(
    model="claude-sonnet-4.6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "你好！"}
    ]
)

print(message.content[0].text)
```

**Python（带系统提示词）：**
```python
import anthropic

client = anthropic.Anthropic(
    api_key="your-api-key",
    base_url="https://api.apolloinn.site"
)

message = client.messages.create(
    model="claude-sonnet-4.6",
    max_tokens=2048,
    system="你是一个专业的代码审查助手，请用中文回答。",
    messages=[
        {"role": "user", "content": "帮我审查这段 Python 代码：\n\ndef add(a, b):\n    return a + b"}
    ]
)

print(message.content[0].text)
```

**Python（流式输出）：**
```python
import anthropic

client = anthropic.Anthropic(
    api_key="your-api-key",
    base_url="https://api.apolloinn.site"
)

with client.messages.stream(
    model="claude-sonnet-4.6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "写一首关于春天的诗"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

**Python（多轮对话）：**
```python
import anthropic

client = anthropic.Anthropic(
    api_key="your-api-key",
    base_url="https://api.apolloinn.site"
)

conversation = []

def chat(user_input):
    conversation.append({"role": "user", "content": user_input})
    response = client.messages.create(
        model="claude-sonnet-4.6",
        max_tokens=1024,
        messages=conversation
    )
    reply = response.content[0].text
    conversation.append({"role": "assistant", "content": reply})
    return reply

print(chat("你好！"))
print(chat("你刚才说了什么？"))  # 模型会记住上下文
```

**JavaScript（Anthropic SDK）：**
```javascript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({
  apiKey: "your-api-key",
  baseURL: "https://api.apolloinn.site",
});

const message = await client.messages.create({
  model: "claude-sonnet-4.6",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Hello!" }],
});

console.log(message.content[0].text);
```

### 响应格式

```json
{
  "id": "msg_xxx",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "你好！我是 Claude，有什么可以帮你的？"
    }
  ],
  "model": "claude-sonnet-4.6",
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 10,
    "output_tokens": 25
  }
}
```

---

## 错误码说明

| HTTP 状态码 | 说明 | 处理建议 |
|------------|------|---------|
| 401 | API Key 无效或缺失 | 检查 Authorization Header 是否正确 |
| 403 | 账户已禁用或无权限 | 联系管理员 |
| 422 | 请求参数格式错误 | 检查请求体格式是否符合规范 |
| 429 | 请求频率超限 | 降低请求频率，加入重试逻辑 |
| 503 | 服务暂时不可用（无可用 token） | 稍后重试 |

### Python 错误处理示例

```python
import httpx
import time

def chat_with_retry(messages, max_retries=3):
    for attempt in range(max_retries):
        try:
            resp = httpx.post(
                "https://api.apolloinn.site/standard/v1/chat/completions",
                headers={"Authorization": "Bearer your-api-key"},
                json={
                    "model": "claude-sonnet-4.6",
                    "messages": messages,
                    "stream": False
                },
                timeout=60
            )
            if resp.status_code == 429:
                wait = 2 ** attempt  # 指数退避
                print(f"频率限制，{wait}秒后重试...")
                time.sleep(wait)
                continue
            if resp.status_code == 503:
                print("服务暂时不可用，稍后重试...")
                time.sleep(5)
                continue
            resp.raise_for_status()
            return resp.json()
        except httpx.HTTPStatusError as e:
            print(f"HTTP 错误: {e.response.status_code}")
            raise
    raise Exception("超过最大重试次数")
```

---

## 接口选择建议

| 使用场景 | 推荐接口 |
|---------|---------|
| Cursor IDE 接入 | `POST /v1/chat/completions` |
| 自建应用（OpenAI SDK） | `POST /standard/v1/chat/completions` |
| 需要获取思考过程 | `POST /standard/v1/chat/completions` + `reasoning_mode: "reasoning_content"` |
| 已有 Anthropic SDK 项目 | `POST /v1/messages` |
| 查询可用模型 | `GET /v1/models` |
