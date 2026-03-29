---
name: santa-method
description: "収束ループを持つマルチエージェント対抗検証。2つの独立したレビューエージェントが両方パスしなければ出力はリリースされない。"
origin: "Ronald Skelton - Founder, RapportScore.ai"
---

# Santa Method

マルチエージェント対抗検証フレームワーク。リストを作り、2回チェック。問題があれば良くなるまで修正。

核心的な洞察：単一エージェントが自身の出力をレビューすると、出力を生成した際と同じバイアス、知識のギャップ、体系的エラーを共有します。共有コンテキストを持たない2つの独立したレビュアーがこの障害モードを解消します。

## 発動タイミング

以下の場合にこのスキルを起動します：
- 出力が公開、デプロイ、またはエンドユーザーに消費される場合
- コンプライアンス、規制、ブランド制約を遵守する必要がある場合
- 人間のレビューなしでコードがプロダクションにリリースされる場合
- コンテンツの正確性が重要な場合（技術ドキュメント、教育資料、顧客向けコピー）
- スポットチェックでは体系的パターンを見逃す大規模バッチ生成
- ハルシネーションリスクが高い場合（主張、統計、APIリファレンス、法的言語）

内部ドラフト、探索的リサーチ、決定論的検証を持つタスクには使用しないでください（それらにはbuild/test/lintパイプラインを使用）。

## アーキテクチャ

```
┌─────────────┐
│  GENERATOR   │  Phase 1: Make a List
│  (Agent A)   │  Produce the deliverable
└──────┬───────┘
       │ output
       ▼
┌──────────────────────────────┐
│     DUAL INDEPENDENT REVIEW   │  Phase 2: Check It Twice
│                                │
│  ┌───────────┐ ┌───────────┐  │  Two agents, same rubric,
│  │ Reviewer B │ │ Reviewer C │  │  no shared context
│  └─────┬─────┘ └─────┬─────┘  │
│        │              │        │
└────────┼──────────────┼────────┘
         │              │
         ▼              ▼
┌──────────────────────────────┐
│        VERDICT GATE           │  Phase 3: Naughty or Nice
│                                │
│  B passes AND C passes → NICE  │  Both must pass.
│  Otherwise → NAUGHTY           │  No exceptions.
└──────┬──────────────┬─────────┘
       │              │
    NICE           NAUGHTY
       │              │
       ▼              ▼
   [ SHIP ]    ┌─────────────┐
               │  FIX CYCLE   │  Phase 4: Fix Until Nice
               │              │
               │ iteration++  │  Collect all flags.
               │ if i > MAX:  │  Fix all issues.
               │   escalate   │  Re-run both reviewers.
               │ else:        │  Loop until convergence.
               │   goto Ph.2  │
               └──────────────┘
```

## フェーズ詳細

### フェーズ 1: リスト作成（生成）

主要タスクを実行します。通常の生成ワークフローに変更はありません。Santa Methodは生成後の検証レイヤーであり、生成戦略ではありません。

```python
# The generator runs as normal
output = generate(task_spec)
```

### フェーズ 2: 2回チェック（独立デュアルレビュー）

2つのレビューエージェントを並列で起動します。重要な不変条件：

1. **コンテキストの分離** — どちらのレビュアーも他方の評価を見ない
2. **同一の評価基準** — 両方が同じ評価基準を受け取る
3. **同じ入力** — 両方が元の仕様と生成された出力を受け取る
4. **構造化出力** — 各レビュアーは散文ではなく型付き判定を返す

```python
REVIEWER_PROMPT = """
You are an independent quality reviewer. You have NOT seen any other review of this output.

## Task Specification
{task_spec}

## Output Under Review
{output}

## Evaluation Rubric
{rubric}

## Instructions
Evaluate the output against EACH rubric criterion. For each:
- PASS: criterion fully met, no issues
- FAIL: specific issue found (cite the exact problem)

Return your assessment as structured JSON:
{
  "verdict": "PASS" | "FAIL",
  "checks": [
    {"criterion": "...", "result": "PASS|FAIL", "detail": "..."}
  ],
  "critical_issues": ["..."],   // blockers that must be fixed
  "suggestions": ["..."]         // non-blocking improvements
}

Be rigorous. Your job is to find problems, not to approve.
"""
```

```python
# Spawn reviewers in parallel (Claude Code subagents)
review_b = Agent(prompt=REVIEWER_PROMPT.format(...), description="Santa Reviewer B")
review_c = Agent(prompt=REVIEWER_PROMPT.format(...), description="Santa Reviewer C")

# Both run concurrently — neither sees the other
```

### 評価基準の設計

評価基準は最も重要な入力です。曖昧な基準は曖昧なレビューを生みます。すべての基準に客観的な合否条件が必要です。

| 基準 | 合格条件 | 失敗シグナル |
|-----------|---------------|----------------|
| 事実の正確性 | すべての主張がソース資料や一般知識で検証可能 | 架空の統計、誤ったバージョン番号、存在しないAPI |
| ハルシネーションなし | 架空のエンティティ、引用、URL、参照がない | 存在しないページへのリンク、出典のない引用 |
| 完全性 | 仕様のすべての要件に対応 | 欠落セクション、スキップされたエッジケース、不完全なカバレッジ |
| コンプライアンス | プロジェクト固有の制約をすべてパス | 禁止用語の使用、トーン違反、規制非準拠 |
| 内部一貫性 | 出力内に矛盾がない | セクションAがXと言い、セクションBがXでないと言う |
| 技術的正確性 | コードがコンパイル/実行可能、アルゴリズムが健全 | 構文エラー、ロジックバグ、誤った計算量の主張 |

#### ドメイン固有の評価基準拡張

**コンテンツ/マーケティング:**
- ブランドボイスの遵守
- SEO要件の充足（キーワード密度、メタタグ、構造）
- 競合他社の商標誤用なし
- CTAの存在と正しいリンク

**コード:**
- 型安全性（`any`のリーク防止、適切なnull処理）
- エラーハンドリングカバレッジ
- セキュリティ（コード内の秘密情報なし、入力バリデーション、インジェクション防止）
- 新しいパスのテストカバレッジ

**コンプライアンス重視（規制、法務、金融）:**
- 結果保証や根拠のない主張がない
- 必要な免責事項の記載
- 承認済み用語のみ使用
- 管轄区域に適した言語

### フェーズ 3: 判定ゲート（Naughty or Nice）

```python
def santa_verdict(review_b, review_c):
    """Both reviewers must pass. No partial credit."""
    if review_b.verdict == "PASS" and review_c.verdict == "PASS":
        return "NICE"  # Ship it

    # Merge flags from both reviewers, deduplicate
    all_issues = dedupe(review_b.critical_issues + review_c.critical_issues)
    all_suggestions = dedupe(review_b.suggestions + review_c.suggestions)

    return "NAUGHTY", all_issues, all_suggestions
```

両方がパスしなければならない理由：片方のレビュアーだけが問題を検出した場合、その問題は実在します。もう片方のレビュアーの盲点こそ、Santa Methodが排除すべき障害モードです。

### フェーズ 4: 修正と収束（Fix Until Nice）

```python
MAX_ITERATIONS = 3

for iteration in range(MAX_ITERATIONS):
    verdict, issues, suggestions = santa_verdict(review_b, review_c)

    if verdict == "NICE":
        log_santa_result(output, iteration, "passed")
        return ship(output)

    # Fix all critical issues (suggestions are optional)
    output = fix_agent.execute(
        output=output,
        issues=issues,
        instruction="Fix ONLY the flagged issues. Do not refactor or add unrequested changes."
    )

    # Re-run BOTH reviewers on fixed output (fresh agents, no memory of previous round)
    review_b = Agent(prompt=REVIEWER_PROMPT.format(output=output, ...))
    review_c = Agent(prompt=REVIEWER_PROMPT.format(output=output, ...))

# Exhausted iterations — escalate
log_santa_result(output, MAX_ITERATIONS, "escalated")
escalate_to_human(output, issues)
```

重要：各レビューラウンドでは**新しいエージェント**を使用します。レビュアーは前のラウンドの記憶を持ってはなりません。前のコンテキストがアンカリングバイアスを生むためです。

## 実装パターン

### パターン A: Claude Codeサブエージェント（推奨）

サブエージェントは真のコンテキスト分離を提供します。各レビュアーは共有状態のない独立したプロセスです。

```bash
# In a Claude Code session, use the Agent tool to spawn reviewers
# Both agents run in parallel for speed
```

```python
# Pseudocode for Agent tool invocation
reviewer_b = Agent(
    description="Santa Review B",
    prompt=f"Review this output for quality...\n\nRUBRIC:\n{rubric}\n\nOUTPUT:\n{output}"
)
reviewer_c = Agent(
    description="Santa Review C",
    prompt=f"Review this output for quality...\n\nRUBRIC:\n{rubric}\n\nOUTPUT:\n{output}"
)
```

### パターン B: シーケンシャルインライン（フォールバック）

サブエージェントが利用できない場合、明示的なコンテキストリセットで分離をシミュレートします：

1. 出力を生成
2. 新しいコンテキスト：「あなたはレビュアー1です。この評価基準のみに対して評価してください。問題を見つけてください。」
3. 結果をそのまま記録
4. コンテキストを完全にクリア
5. 新しいコンテキスト：「あなたはレビュアー2です。この評価基準のみに対して評価してください。問題を見つけてください。」
6. 両方のレビューを比較、修正、繰り返し

サブエージェントパターンが厳密に優れています — インラインシミュレーションはレビュアー間でコンテキストが漏れるリスクがあります。

### パターン C: バッチサンプリング

大規模バッチ（100件以上）の場合、全アイテムに完全なSantaを適用するとコストが高くなります。層化サンプリングを使用します：

1. ランダムサンプルでSantaを実行（バッチの10-15%、最低5件）
2. 失敗をタイプ別に分類（ハルシネーション、コンプライアンス、完全性など）
3. 体系的パターンが見つかった場合、バッチ全体に対象を絞った修正を適用
4. 修正されたバッチを再サンプリングして再検証
5. クリーンなサンプルがパスするまで継続

```python
import random

def santa_batch(items, rubric, sample_rate=0.15):
    sample = random.sample(items, max(5, int(len(items) * sample_rate)))

    for item in sample:
        result = santa_full(item, rubric)
        if result.verdict == "NAUGHTY":
            pattern = classify_failure(result.issues)
            items = batch_fix(items, pattern)  # Fix all items matching pattern
            return santa_batch(items, rubric)   # Re-sample

    return items  # Clean sample → ship batch
```

## 障害モードと対策

| 障害モード | 症状 | 対策 |
|-------------|---------|------------|
| 無限ループ | 修正後もレビュアーが新しい問題を見つけ続ける | 最大反復回数制限（3回）。エスカレーション。 |
| ラバースタンピング | 両方のレビュアーがすべてパスさせる | 対抗的プロンプト：「あなたの仕事は問題を見つけることであり、承認することではない。」 |
| 主観的ドリフト | レビュアーがエラーではなくスタイルの好みにフラグを立てる | 客観的な合否条件のみの厳密な評価基準 |
| 修正リグレッション | 問題Aの修正が問題Bを導入 | 各ラウンドの新しいレビュアーがリグレッションをキャッチ |
| レビュアー合意バイアス | 両方のレビュアーが同じことを見逃す | 独立性で緩和されるが排除はされない。重要な出力には第3のレビュアーまたは人間のスポットチェックを追加。 |
| コスト爆発 | 大きな出力に対して反復が多すぎる | バッチサンプリングパターン。検証サイクルごとの予算上限。 |

## 他のスキルとの統合

| スキル | 関係 |
|-------|-------------|
| Verification Loop | 決定論的チェック（ビルド、lint、テスト）に使用。Santaはセマンティックチェック（正確性、ハルシネーション）に使用。verification-loopを先に実行し、Santaを後に実行。 |
| Eval Harness | Santa Methodの結果が評価メトリクスにフィードされる。Santa実行全体のpass@kを追跡してジェネレーター品質を経時的に測定。 |
| Continuous Learning v2 | Santaの結果がインスティンクトになる。同じ基準での繰り返しの失敗→パターンを回避する学習済み行動。 |
| Strategic Compact | コンパクション前にSantaを実行。検証途中でレビューコンテキストを失わない。 |

## メトリクス

Santa Methodの有効性を測定するために以下を追跡します：

- **初回パス率**: ラウンド1でSantaをパスした出力の割合（目標：>70%）
- **平均収束反復回数**: NICEまでの平均ラウンド数（目標：<1.5）
- **問題分類**: 失敗タイプの分布（ハルシネーション vs 完全性 vs コンプライアンス）
- **レビュアー合意率**: 両方のレビュアーがフラグした問題 vs 片方だけの割合（低い合意＝評価基準の厳密化が必要）
- **エスケープ率**: リリース後にSantaがキャッチすべきだった問題（目標：0）

## コスト分析

Santa Methodは検証サイクルあたり生成のみのトークンコストの約2-3倍かかります。ほとんどのハイステークス出力では、これは割安です：

```
Cost of Santa = (generation tokens) + 2×(review tokens per round) × (avg rounds)
Cost of NOT Santa = (reputation damage) + (correction effort) + (trust erosion)
```

バッチ操作では、サンプリングパターンにより、体系的な問題の90%以上をキャッチしながらコストを完全検証の約15-20%に削減できます。
