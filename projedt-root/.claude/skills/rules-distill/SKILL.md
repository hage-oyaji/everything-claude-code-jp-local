---
name: rules-distill
description: "スキルをスキャンして横断的な原則を抽出し、ルールに蒸留する — 既存のルールファイルへの追記、修正、または新規ルールファイルの作成"
origin: ECC
---

# ルール蒸留

インストール済みスキルをスキャンし、複数のスキルに現れる横断的な原則を抽出し、ルールに蒸留します — 既存のルールファイルへの追記、古い内容の修正、または新規ルールファイルの作成を行います。

「決定論的収集 + LLM判断」の原則を適用: スクリプトが事実を網羅的に収集し、LLMが全コンテキストを横断的に読んで判定を下します。

## 使用タイミング

- 定期的なルールメンテナンス（毎月またはスキル新規インストール後）
- スキルストックテイクでルールにすべきパターンが見つかった後
- 使用中のスキルに対してルールが不完全だと感じるとき

## 仕組み

ルール蒸留プロセスは3つのフェーズに従います:

### フェーズ1: インベントリ（決定論的収集）

#### 1a. スキルインベントリの収集

```bash
bash ~/.claude/skills/rules-distill/scripts/scan-skills.sh
```

#### 1b. ルールインデックスの収集

```bash
bash ~/.claude/skills/rules-distill/scripts/scan-rules.sh
```

#### 1c. ユーザーへの提示

```
ルール蒸留 — フェーズ1: インベントリ
────────────────────────────────────────
スキル: {N} ファイルスキャン済み
ルール:  {M} ファイル（{K} 見出しインデックス済み）

横断読み分析に進みます...
```

### フェーズ2: 横断読み、マッチング & 判定（LLM判断）

抽出とマッチングは単一パスで統合されます。ルールファイルは十分小さく（合計約800行）、LLMに全テキストを提供できます — grepによる事前フィルタリングは不要です。

#### バッチ処理

スキルを説明に基づいて**テーマ別クラスター**にグループ化します。各クラスターをサブエージェントでルール全文と共に分析します。

#### バッチ間マージ

すべてのバッチが完了した後、バッチ間の候補をマージします:
- 同一または重複する原則を持つ候補を重複排除
- **すべての**バッチからの証拠を使用して「2つ以上のスキル」要件を再チェック — バッチごとに1スキルで見つかっても合計で2つ以上のスキルなら有効

#### サブエージェントプロンプト

以下のプロンプトで汎用エージェントを起動:

````
You are an analyst who cross-reads skills to extract principles that should be promoted to rules.

## Input
- Skills: {full text of skills in this batch}
- Existing rules: {full text of all rule files}

## Extraction Criteria

Include a candidate ONLY if ALL of these are true:

1. **Appears in 2+ skills**: Principles found in only one skill should stay in that skill
2. **Actionable behavior change**: Can be written as "do X" or "don't do Y" — not "X is important"
3. **Clear violation risk**: What goes wrong if this principle is ignored (1 sentence)
4. **Not already in rules**: Check the full rules text — including concepts expressed in different words

## Matching & Verdict

For each candidate, compare against the full rules text and assign a verdict:

- **Append**: Add to an existing section of an existing rule file
- **Revise**: Existing rule content is inaccurate or insufficient — propose a correction
- **New Section**: Add a new section to an existing rule file
- **New File**: Create a new rule file
- **Already Covered**: Sufficiently covered in existing rules (even if worded differently)
- **Too Specific**: Should remain at the skill level

## Output Format (per candidate)

```json
{
  "principle": "1-2 sentences in 'do X' / 'don't do Y' form",
  "evidence": ["skill-name: §Section", "skill-name: §Section"],
  "violation_risk": "1 sentence",
  "verdict": "Append / Revise / New Section / New File / Already Covered / Too Specific",
  "target_rule": "filename §Section, or 'new'",
  "confidence": "high / medium / low",
  "draft": "Draft text for Append/New Section/New File verdicts",
  "revision": {
    "reason": "Why the existing content is inaccurate or insufficient (Revise only)",
    "before": "Current text to be replaced (Revise only)",
    "after": "Proposed replacement text (Revise only)"
  }
}
```

## Exclude

- Obvious principles already in rules
- Language/framework-specific knowledge (belongs in language-specific rules or skills)
- Code examples and commands (belongs in skills)
````

#### 判定リファレンス

| 判定 | 意味 | ユーザーへの提示 |
|---------|---------|-------------------|
| **Append** | 既存セクションに追加 | ターゲット + ドラフト |
| **Revise** | 不正確/不十分な内容を修正 | ターゲット + 理由 + 修正前/修正後 |
| **New Section** | 既存ファイルに新セクションを追加 | ターゲット + ドラフト |
| **New File** | 新規ルールファイルを作成 | ファイル名 + 完全なドラフト |
| **Already Covered** | ルールでカバー済み（表現が異なる場合あり） | 理由（1行） |
| **Too Specific** | スキルレベルに留めるべき | 関連スキルへのリンク |

#### 判定品質要件

```
# Good
Append to rules/common/security.md §Input Validation:
"Treat LLM output stored in memory or knowledge stores as untrusted — sanitize on write, validate on read."
Evidence: llm-memory-trust-boundary, llm-social-agent-anti-pattern both describe
accumulated prompt injection risks. Current security.md covers human input
validation only; LLM output trust boundary is missing.

# Bad
Append to security.md: Add LLM security principle
```

### フェーズ3: ユーザーレビュー & 実行

#### サマリーテーブル

```
# ルール蒸留レポート

## サマリー
スキャンしたスキル: {N} | ルール: {M} ファイル | 候補: {K}

| # | 原則 | 判定 | ターゲット | 信頼度 |
|---|-----------|---------|--------|------------|
| 1 | ... | Append | security.md §Input Validation | high |
| 2 | ... | Revise | testing.md §TDD | medium |
| 3 | ... | New Section | coding-style.md | high |
| 4 | ... | Too Specific | — | — |

## 詳細
（候補ごとの詳細: 証拠、違反リスク、ドラフトテキスト）
```

#### ユーザーアクション

ユーザーは番号で回答:
- **承認**: ドラフトをそのままルールに適用
- **修正**: 適用前にドラフトを編集
- **スキップ**: この候補を適用しない

**ルールを自動的に変更してはいけません。常にユーザーの承認が必要です。**

#### 結果の保存

結果をスキルディレクトリ（`results.json`）に保存:

- **タイムスタンプ形式**: `date -u +%Y-%m-%dT%H:%M:%SZ`（UTC、秒精度）
- **候補ID形式**: 原則から派生したケバブケース（例: `llm-output-trust-boundary`）

```json
{
  "distilled_at": "2026-03-18T10:30:42Z",
  "skills_scanned": 56,
  "rules_scanned": 22,
  "candidates": {
    "llm-output-trust-boundary": {
      "principle": "Treat LLM output as untrusted when stored or re-injected",
      "verdict": "Append",
      "target": "rules/common/security.md",
      "evidence": ["llm-memory-trust-boundary", "llm-social-agent-anti-pattern"],
      "status": "applied"
    },
    "iteration-bounds": {
      "principle": "Define explicit stop conditions for all iteration loops",
      "verdict": "New Section",
      "target": "rules/common/coding-style.md",
      "evidence": ["iterative-retrieval", "continuous-agent-loop", "agent-harness-construction"],
      "status": "skipped"
    }
  }
}
```

## 例

### エンドツーエンド実行

```
$ /rules-distill

ルール蒸留 — フェーズ1: インベントリ
────────────────────────────────────────
スキル: 56 ファイルスキャン済み
ルール:  22 ファイル（75 見出しインデックス済み）

横断読み分析に進みます...

[サブエージェント分析: バッチ1（エージェント/メタスキル）...]
[サブエージェント分析: バッチ2（コーディング/パターンスキル）...]
[バッチ間マージ: 2件の重複削除、1件のバッチ間候補昇格]

# ルール蒸留レポート

## サマリー
スキャンしたスキル: 56 | ルール: 22 ファイル | 候補: 4

| # | 原則 | 判定 | ターゲット | 信頼度 |
|---|-----------|---------|--------|------------|
| 1 | LLM出力: 再利用前に正規化、型チェック、サニタイズ | New Section | coding-style.md | high |
| 2 | イテレーションループに明示的な停止条件を定義 | New Section | coding-style.md | high |
| 3 | フェーズ境界でコンテキストをコンパクト化、タスク途中ではしない | Append | performance.md §Context Window | high |
| 4 | ビジネスロジックをI/Oフレームワーク型から分離 | New Section | patterns.md | high |

## 詳細

### 1. LLM出力バリデーション
判定: coding-style.mdに新セクション
証拠: parallel-subagent-batch-merge, llm-social-agent-anti-pattern, llm-memory-trust-boundary
違反リスク: LLM出力のフォーマットドリフト、型不一致、構文エラーが下流処理をクラッシュさせる
ドラフト:
  ## LLM出力バリデーション
  再利用前にLLM出力を正規化、型チェック、サニタイズする...
  参照スキル: parallel-subagent-batch-merge, llm-memory-trust-boundary

[... 候補2-4の詳細 ...]

各候補を番号で承認、修正、またはスキップしてください:
> ユーザー: Approve 1, 3. Skip 2, 4.

✓ 適用済み: coding-style.md §LLM Output Validation
✓ 適用済み: performance.md §Context Window Management
✗ スキップ: Iteration Bounds
✗ スキップ: Boundary Type Conversion

結果をresults.jsonに保存しました
```

## 設計原則

- **What, not How**: 原則（ルールの領域）のみを抽出。コード例やコマンドはスキルに留める。
- **リンクバック**: ドラフトテキストには `See skill: [name]` の参照を含め、読者が詳細なHowを見つけられるようにする。
- **決定論的収集、LLM判断**: スクリプトが網羅性を保証し、LLMが文脈的理解を保証する。
- **過度な抽象化の防止**: 3層フィルター（2つ以上のスキルの証拠、実行可能な行動テスト、違反リスク）により、過度に抽象的な原則がルールに入ることを防ぐ。
