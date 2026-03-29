---
name: regex-vs-llm-structured-text
description: 構造化テキストの解析において正規表現とLLMのどちらを選択するかの判断フレームワーク — まず正規表現で始め、低信頼度のエッジケースにのみLLMを追加する。
origin: ECC
---

# 構造化テキスト解析における正規表現 vs LLM

構造化テキスト（クイズ、フォーム、請求書、ドキュメント）を解析するための実践的な判断フレームワーク。重要な知見: 正規表現はケースの95-98%を安価かつ決定論的に処理できます。残りのエッジケースにのみ高価なLLM呼び出しを使用します。

## アクティベーション条件

- 繰り返しパターンを持つ構造化テキスト（質問、フォーム、テーブル）を解析するとき
- テキスト抽出に正規表現とLLMのどちらを使うか決めるとき
- 両方のアプローチを組み合わせたハイブリッドパイプラインを構築するとき
- テキスト処理のコスト/精度のトレードオフを最適化するとき

## 判断フレームワーク

```
テキストフォーマットは一貫して繰り返されているか？
├── はい（>90%がパターンに従う） → 正規表現から始める
│   ├── 正規表現で95%以上処理可能 → 完了、LLM不要
│   └── 正規表現で95%未満 → エッジケースにのみLLMを追加
└── いいえ（自由形式、非常に可変的） → LLMを直接使用
```

## アーキテクチャパターン

```
ソーステキスト
    │
    ▼
[正規表現パーサー] ─── 構造を抽出（95-98%の精度）
    │
    ▼
[テキストクリーナー] ─── ノイズを除去（マーカー、ページ番号、アーティファクト）
    │
    ▼
[信頼度スコアラー] ─── 低信頼度の抽出をフラグ
    │
    ├── 高信頼度（≥0.95） → 直接出力
    │
    └── 低信頼度（<0.95） → [LLMバリデーター] → 出力
```

## 実装

### 1. 正規表現パーサー（大部分を処理）

```python
import re
from dataclasses import dataclass

@dataclass(frozen=True)
class ParsedItem:
    id: str
    text: str
    choices: tuple[str, ...]
    answer: str
    confidence: float = 1.0

def parse_structured_text(content: str) -> list[ParsedItem]:
    """Parse structured text using regex patterns."""
    pattern = re.compile(
        r"(?P<id>\d+)\.\s*(?P<text>.+?)\n"
        r"(?P<choices>(?:[A-D]\..+?\n)+)"
        r"Answer:\s*(?P<answer>[A-D])",
        re.MULTILINE | re.DOTALL,
    )
    items = []
    for match in pattern.finditer(content):
        choices = tuple(
            c.strip() for c in re.findall(r"[A-D]\.\s*(.+)", match.group("choices"))
        )
        items.append(ParsedItem(
            id=match.group("id"),
            text=match.group("text").strip(),
            choices=choices,
            answer=match.group("answer"),
        ))
    return items
```

### 2. 信頼度スコアリング

LLMレビューが必要な可能性のある項目をフラグ:

```python
@dataclass(frozen=True)
class ConfidenceFlag:
    item_id: str
    score: float
    reasons: tuple[str, ...]

def score_confidence(item: ParsedItem) -> ConfidenceFlag:
    """Score extraction confidence and flag issues."""
    reasons = []
    score = 1.0

    if len(item.choices) < 3:
        reasons.append("few_choices")
        score -= 0.3

    if not item.answer:
        reasons.append("missing_answer")
        score -= 0.5

    if len(item.text) < 10:
        reasons.append("short_text")
        score -= 0.2

    return ConfidenceFlag(
        item_id=item.id,
        score=max(0.0, score),
        reasons=tuple(reasons),
    )

def identify_low_confidence(
    items: list[ParsedItem],
    threshold: float = 0.95,
) -> list[ConfidenceFlag]:
    """Return items below confidence threshold."""
    flags = [score_confidence(item) for item in items]
    return [f for f in flags if f.score < threshold]
```

### 3. LLMバリデーター（エッジケースのみ）

```python
def validate_with_llm(
    item: ParsedItem,
    original_text: str,
    client,
) -> ParsedItem:
    """Use LLM to fix low-confidence extractions."""
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # Cheapest model for validation
        max_tokens=500,
        messages=[{
            "role": "user",
            "content": (
                f"Extract the question, choices, and answer from this text.\n\n"
                f"Text: {original_text}\n\n"
                f"Current extraction: {item}\n\n"
                f"Return corrected JSON if needed, or 'CORRECT' if accurate."
            ),
        }],
    )
    # Parse LLM response and return corrected item...
    return corrected_item
```

### 4. ハイブリッドパイプライン

```python
def process_document(
    content: str,
    *,
    llm_client=None,
    confidence_threshold: float = 0.95,
) -> list[ParsedItem]:
    """Full pipeline: regex -> confidence check -> LLM for edge cases."""
    # Step 1: 正規表現による抽出（95-98%を処理）
    items = parse_structured_text(content)

    # Step 2: 信頼度スコアリング
    low_confidence = identify_low_confidence(items, confidence_threshold)

    if not low_confidence or llm_client is None:
        return items

    # Step 3: LLMバリデーション（フラグされた項目のみ）
    low_conf_ids = {f.item_id for f in low_confidence}
    result = []
    for item in items:
        if item.id in low_conf_ids:
            result.append(validate_with_llm(item, content, llm_client))
        else:
            result.append(item)

    return result
```

## 本番環境のメトリクス

プロダクションクイズ解析パイプライン（410項目）より:

| メトリクス | 値 |
|--------|-------|
| 正規表現成功率 | 98.0% |
| 低信頼度項目 | 8（2.0%） |
| 必要なLLM呼び出し | 約5 |
| 全LLM使用と比較したコスト削減 | 約95% |
| テストカバレッジ | 93% |

## ベストプラクティス

- **正規表現から始める** — 不完全な正規表現でも改善のベースラインになる
- **信頼度スコアリングを使用**してLLMの助けが必要なものをプログラム的に特定
- **バリデーションには最も安価なLLMを使用**（Haikuクラスのモデルで十分）
- **解析済み項目をミューテートしない** — クリーニング/バリデーションステップからは新しいインスタンスを返す
- **TDDはパーサーに適している** — まず既知のパターンのテストを書き、次にエッジケース
- **メトリクスをログ**（正規表現成功率、LLM呼び出し回数）してパイプラインの健全性を追跡

## 避けるべきアンチパターン

- 正規表現で95%以上のケースを処理できるのにすべてのテキストをLLMに送る（高価で遅い）
- 自由形式で非常に可変的なテキストに正規表現を使用する（LLMの方が適している）
- 信頼度スコアリングをスキップして正規表現が「うまくいく」ことを期待する
- クリーニング/バリデーションステップ中に解析済みオブジェクトをミューテートする
- エッジケースをテストしない（不正な入力、欠落フィールド、エンコーディングの問題）

## 使用タイミング

- クイズ/試験問題の解析
- フォームデータの抽出
- 請求書/領収書の処理
- ドキュメント構造の解析（ヘッダー、セクション、テーブル）
- コストが重要な繰り返しパターンを持つ構造化テキスト全般
