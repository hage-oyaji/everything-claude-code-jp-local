---
name: claude-api
description: Python と TypeScript 向け Anthropic Claude API パターン。Messages API、ストリーミング、ツール使用、ビジョン、拡張思考、バッチ、プロンプトキャッシング、Claude Agent SDK をカバー。Claude API や Anthropic SDK を使ったアプリケーション構築時に使用。
origin: ECC
---

# Claude API

Anthropic Claude API と SDK を使ったアプリケーション構築。

## いつアクティベートするか

- Claude API を呼び出すアプリケーションの構築
- コードで `anthropic`（Python）または `@anthropic-ai/sdk`（TypeScript）をインポートしている
- ユーザーが Claude API パターン、ツール使用、ストリーミング、ビジョンについて質問した
- Claude Agent SDK を使ったエージェントワークフローの実装
- API コスト、トークン使用量、レイテンシの最適化

## モデル選択

| モデル | ID | 最適な用途 |
|-------|-----|----------|
| Opus 4.1 | `claude-opus-4-1` | 複雑な推論、アーキテクチャ、リサーチ |
| Sonnet 4 | `claude-sonnet-4-0` | バランスの取れたコーディング、大半の開発タスク |
| Haiku 3.5 | `claude-3-5-haiku-latest` | 高速レスポンス、大量処理、コスト重視 |

タスクに深い推論（Opus）や速度・コスト最適化（Haiku）が必要でない限り、デフォルトは Sonnet 4。本番環境ではエイリアスよりも固定スナップショット ID を推奨。

## Python SDK

### インストール

```bash
pip install anthropic
```

### 基本メッセージ

```python
import anthropic

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from env

message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Explain async/await in Python"}
    ]
)
print(message.content[0].text)
```

### ストリーミング

```python
with client.messages.stream(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a haiku about coding"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

### システムプロンプト

```python
message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    system="You are a senior Python developer. Be concise.",
    messages=[{"role": "user", "content": "Review this function"}]
)
```

## TypeScript SDK

### インストール

```bash
npm install @anthropic-ai/sdk
```

### 基本メッセージ

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic(); // reads ANTHROPIC_API_KEY from env

const message = await client.messages.create({
  model: "claude-sonnet-4-0",
  max_tokens: 1024,
  messages: [
    { role: "user", content: "Explain async/await in TypeScript" }
  ],
});
console.log(message.content[0].text);
```

### ストリーミング

```typescript
const stream = client.messages.stream({
  model: "claude-sonnet-4-0",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Write a haiku" }],
});

for await (const event of stream) {
  if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
    process.stdout.write(event.delta.text);
  }
}
```

## ツール使用

ツールを定義して Claude に呼び出させる：

```python
tools = [
    {
        "name": "get_weather",
        "description": "Get current weather for a location",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City name"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["location"]
        }
    }
]

message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "What's the weather in SF?"}]
)

# Handle tool use response
for block in message.content:
    if block.type == "tool_use":
        # Execute the tool with block.input
        result = get_weather(**block.input)
        # Send result back
        follow_up = client.messages.create(
            model="claude-sonnet-4-0",
            max_tokens=1024,
            tools=tools,
            messages=[
                {"role": "user", "content": "What's the weather in SF?"},
                {"role": "assistant", "content": message.content},
                {"role": "user", "content": [
                    {"type": "tool_result", "tool_use_id": block.id, "content": str(result)}
                ]}
            ]
        )
```

## ビジョン

画像を送信して分析：

```python
import base64

with open("diagram.png", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": image_data}},
            {"type": "text", "text": "Describe this diagram"}
        ]
    }]
)
```

## 拡張思考

複雑な推論タスク向け：

```python
message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000
    },
    messages=[{"role": "user", "content": "Solve this math problem step by step..."}]
)

for block in message.content:
    if block.type == "thinking":
        print(f"Thinking: {block.thinking}")
    elif block.type == "text":
        print(f"Answer: {block.text}")
```

## プロンプトキャッシング

大きなシステムプロンプトやコンテキストをキャッシュしてコストを削減：

```python
message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    system=[
        {"type": "text", "text": large_system_prompt, "cache_control": {"type": "ephemeral"}}
    ],
    messages=[{"role": "user", "content": "Question about the cached context"}]
)
# Check cache usage
print(f"Cache read: {message.usage.cache_read_input_tokens}")
print(f"Cache creation: {message.usage.cache_creation_input_tokens}")
```

## Batches API

大量処理を非同期で実行し、50% のコスト削減：

```python
import time

batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": f"request-{i}",
            "params": {
                "model": "claude-sonnet-4-0",
                "max_tokens": 1024,
                "messages": [{"role": "user", "content": prompt}]
            }
        }
        for i, prompt in enumerate(prompts)
    ]
)

# Poll for completion
while True:
    status = client.messages.batches.retrieve(batch.id)
    if status.processing_status == "ended":
        break
    time.sleep(30)

# Get results
for result in client.messages.batches.results(batch.id):
    print(result.result.message.content[0].text)
```

## Claude Agent SDK

マルチステップエージェントの構築：

```python
# Note: Agent SDK API surface may change — check official docs
import anthropic

# Define tools as functions
tools = [{
    "name": "search_codebase",
    "description": "Search the codebase for relevant code",
    "input_schema": {
        "type": "object",
        "properties": {"query": {"type": "string"}},
        "required": ["query"]
    }
}]

# Run an agentic loop with tool use
client = anthropic.Anthropic()
messages = [{"role": "user", "content": "Review the auth module for security issues"}]

while True:
    response = client.messages.create(
        model="claude-sonnet-4-0",
        max_tokens=4096,
        tools=tools,
        messages=messages,
    )
    if response.stop_reason == "end_turn":
        break
    # Handle tool calls and continue the loop
    messages.append({"role": "assistant", "content": response.content})
    # ... execute tools and append tool_result messages
```

## コスト最適化

| 戦略 | 削減率 | 使用タイミング |
|----------|---------|-------------|
| プロンプトキャッシング | キャッシュ済みトークンで最大 90% | 繰り返しのシステムプロンプトやコンテキスト |
| Batches API | 50% | 時間制約のない大量処理 |
| Sonnet の代わりに Haiku | 約 75% | シンプルなタスク、分類、抽出 |
| max_tokens を短く | 可変 | 出力が短いと分かっている場合 |
| ストリーミング | なし（同コスト） | UX 向上、価格は同じ |

## エラーハンドリング

```python
import time

from anthropic import APIError, RateLimitError, APIConnectionError

try:
    message = client.messages.create(...)
except RateLimitError:
    # Back off and retry
    time.sleep(60)
except APIConnectionError:
    # Network issue, retry with backoff
    pass
except APIError as e:
    print(f"API error {e.status_code}: {e.message}")
```

## 環境セットアップ

```bash
# Required
export ANTHROPIC_API_KEY="your-api-key-here"

# Optional: set default model
export ANTHROPIC_MODEL="claude-sonnet-4-0"
```

API キーをハードコードしないこと。必ず環境変数を使用すること。
