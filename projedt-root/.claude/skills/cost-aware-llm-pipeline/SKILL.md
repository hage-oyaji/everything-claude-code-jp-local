---
name: cost-aware-llm-pipeline
description: LLM API利用のコスト最適化パターン — タスク複雑度によるモデルルーティング、予算追跡、リトライロジック、プロンプトキャッシュ。
origin: ECC
---

# コスト意識型LLMパイプライン

品質を維持しながらLLM APIコストを制御するためのパターン。モデルルーティング、予算追跡、リトライロジック、プロンプトキャッシュをコンポーザブルなパイプラインに統合します。

## 有効化するタイミング

- LLM API（Claude、GPTなど）を呼び出すアプリケーションを構築する場合
- 複雑度の異なるアイテムをバッチ処理する場合
- API支出の予算内に収める必要がある場合
- 複雑なタスクの品質を犠牲にせずコストを最適化する場合

## コアコンセプト

### 1. タスク複雑度によるモデルルーティング

シンプルなタスクには安価なモデルを自動選択し、複雑なタスクには高価なモデルを使用します。

```python
MODEL_SONNET = "claude-sonnet-4-6"
MODEL_HAIKU = "claude-haiku-4-5-20251001"

_SONNET_TEXT_THRESHOLD = 10_000  # chars
_SONNET_ITEM_THRESHOLD = 30     # items

def select_model(
    text_length: int,
    item_count: int,
    force_model: str | None = None,
) -> str:
    """Select model based on task complexity."""
    if force_model is not None:
        return force_model
    if text_length >= _SONNET_TEXT_THRESHOLD or item_count >= _SONNET_ITEM_THRESHOLD:
        return MODEL_SONNET  # Complex task
    return MODEL_HAIKU  # Simple task (3-4x cheaper)
```

### 2. イミュータブルなコスト追跡

凍結されたデータクラスで累積コストを追跡します。各API呼び出しは新しいトラッカーを返し、状態を変更しません。

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class CostRecord:
    model: str
    input_tokens: int
    output_tokens: int
    cost_usd: float

@dataclass(frozen=True, slots=True)
class CostTracker:
    budget_limit: float = 1.00
    records: tuple[CostRecord, ...] = ()

    def add(self, record: CostRecord) -> "CostTracker":
        """Return new tracker with added record (never mutates self)."""
        return CostTracker(
            budget_limit=self.budget_limit,
            records=(*self.records, record),
        )

    @property
    def total_cost(self) -> float:
        return sum(r.cost_usd for r in self.records)

    @property
    def over_budget(self) -> bool:
        return self.total_cost > self.budget_limit
```

### 3. 限定的なリトライロジック

一時的なエラーのみリトライします。認証エラーや不正リクエストエラーでは即座に失敗します。

```python
from anthropic import (
    APIConnectionError,
    InternalServerError,
    RateLimitError,
)

_RETRYABLE_ERRORS = (APIConnectionError, RateLimitError, InternalServerError)
_MAX_RETRIES = 3

def call_with_retry(func, *, max_retries: int = _MAX_RETRIES):
    """Retry only on transient errors, fail fast on others."""
    for attempt in range(max_retries):
        try:
            return func()
        except _RETRYABLE_ERRORS:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
    # AuthenticationError, BadRequestError etc. → raise immediately
```

### 4. プロンプトキャッシュ

長いシステムプロンプトをキャッシュし、リクエストごとの再送信を回避します。

```python
messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": system_prompt,
                "cache_control": {"type": "ephemeral"},  # Cache this
            },
            {
                "type": "text",
                "text": user_input,  # Variable part
            },
        ],
    }
]
```

## 組み合わせ

4つの手法すべてを1つのパイプライン関数に統合：

```python
def process(text: str, config: Config, tracker: CostTracker) -> tuple[Result, CostTracker]:
    # 1. Route model
    model = select_model(len(text), estimated_items, config.force_model)

    # 2. Check budget
    if tracker.over_budget:
        raise BudgetExceededError(tracker.total_cost, tracker.budget_limit)

    # 3. Call with retry + caching
    response = call_with_retry(lambda: client.messages.create(
        model=model,
        messages=build_cached_messages(system_prompt, text),
    ))

    # 4. Track cost (immutable)
    record = CostRecord(model=model, input_tokens=..., output_tokens=..., cost_usd=...)
    tracker = tracker.add(record)

    return parse_result(response), tracker
```

## 料金リファレンス（2025-2026）

| モデル | 入力 ($/1Mトークン) | 出力 ($/1Mトークン) | 相対コスト |
|-------|---------------------|----------------------|-----------|
| Haiku 4.5 | $0.80 | $4.00 | 1x |
| Sonnet 4.6 | $3.00 | $15.00 | ～4x |
| Opus 4.5 | $15.00 | $75.00 | ～19x |

## ベストプラクティス

- **最も安価なモデルから始め**、複雑度のしきい値が満たされた場合のみ高価なモデルにルーティング
- **バッチ処理前に明示的な予算制限を設定** — 過剰支出よりも早期失敗を優先
- **モデル選択の判断をログに記録**し、実データに基づいてしきい値を調整可能にする
- 1024トークン以上のシステムプロンプトには**プロンプトキャッシュを使用** — コストとレイテンシの両方を節約
- **認証やバリデーションエラーでは絶対にリトライしない** — 一時的な障害（ネットワーク、レートリミット、サーバーエラー）のみ

## 避けるべきアンチパターン

- 複雑度に関係なくすべてのリクエストに最も高価なモデルを使用
- すべてのエラーでリトライ（永続的な障害で予算を無駄にする）
- コスト追跡の状態を変更（デバッグと監査が困難になる）
- コードベース全体にモデル名をハードコーディング（定数または設定を使用すべき）
- 繰り返しのシステムプロンプトでプロンプトキャッシュを無視

## 使用する場面

- Claude、OpenAI、または同様のLLM APIを呼び出すすべてのアプリケーション
- コストが急速に蓄積するバッチ処理パイプライン
- インテリジェントなルーティングが必要なマルチモデルアーキテクチャ
- 予算のガードレールが必要な本番システム
