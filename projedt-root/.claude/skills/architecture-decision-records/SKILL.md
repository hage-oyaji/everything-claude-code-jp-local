---
name: architecture-decision-records
description: Claude Codeセッション中に行われたアーキテクチャ上の意思決定を構造化されたADRとして記録する。意思決定の瞬間を自動検出し、コンテキスト、検討された代替案、根拠を記録する。将来の開発者がコードベースの形状の理由を理解できるようADRログを維持する。
origin: ECC
---

# アーキテクチャデシジョンレコード

コーディングセッション中にアーキテクチャ上の意思決定を発生時に記録します。決定がSlackスレッド、PRコメント、個人の記憶だけに存在するのではなく、このスキルはコードと並んで存在する構造化されたADRドキュメントを生成します。

## 起動条件

- ユーザーが明示的に「この決定を記録しよう」や「ADRにして」と言った場合
- ユーザーが重要な選択肢（フレームワーク、ライブラリ、パターン、データベース、API設計）の間で選択した場合
- ユーザーが「〜することに決めた」や「YではなくXを使う理由は…」と言った場合
- ユーザーが「なぜXを選んだのか？」と質問した場合（既存ADRの参照）
- アーキテクチャ上のトレードオフが議論される計画フェーズ中

## ADR形式

Michael Nygardが提唱した軽量ADR形式を、AI支援開発向けに適応したものを使用します：

```markdown
# ADR-NNNN: [Decision Title]

**Date**: YYYY-MM-DD
**Status**: proposed | accepted | deprecated | superseded by ADR-NNNN
**Deciders**: [who was involved]

## Context

What is the issue that we're seeing that is motivating this decision or change?

[2-5 sentences describing the situation, constraints, and forces at play]

## Decision

What is the change that we're proposing and/or doing?

[1-3 sentences stating the decision clearly]

## Alternatives Considered

### Alternative 1: [Name]
- **Pros**: [benefits]
- **Cons**: [drawbacks]
- **Why not**: [specific reason this was rejected]

### Alternative 2: [Name]
- **Pros**: [benefits]
- **Cons**: [drawbacks]
- **Why not**: [specific reason this was rejected]

## Consequences

What becomes easier or more difficult to do because of this change?

### Positive
- [benefit 1]
- [benefit 2]

### Negative
- [trade-off 1]
- [trade-off 2]

### Risks
- [risk and mitigation]
```

## ワークフロー

### 新しいADRの記録

意思決定の瞬間が検出された場合：

1. **初期化（初回のみ）** — `docs/adr/`が存在しない場合、ディレクトリの作成、インデックステーブルヘッダーをシードした`README.md`（下記のADRインデックス形式を参照）、および手動使用用の空の`template.md`の作成について、ユーザーの確認を求めてから実行する。明示的な同意なしにファイルを作成しないこと。
2. **意思決定の特定** — 行われているコアとなるアーキテクチャ上の選択を抽出する
3. **コンテキストの収集** — どのような問題がきっかけか？どのような制約があるか？
4. **代替案の文書化** — 他にどのような選択肢が検討されたか？なぜ却下されたか？
5. **結果の記述** — トレードオフは何か？何が容易になり/困難になるか？
6. **番号の割り当て** — `docs/adr/`内の既存ADRをスキャンしてインクリメントする
7. **確認と書き込み** — ADRの下書きをユーザーに提示してレビューしてもらう。明示的な承認の後にのみ`docs/adr/NNNN-decision-title.md`に書き込む。ユーザーが拒否した場合、ファイルを書き込まずに下書きを破棄する。
8. **インデックスの更新** — `docs/adr/README.md`に追記する

### 既存ADRの参照

ユーザーが「なぜXを選んだのか？」と質問した場合：

1. `docs/adr/`が存在するか確認する — 存在しない場合：「このプロジェクトにADRが見つかりません。アーキテクチャ上の意思決定の記録を始めますか？」と回答する
2. 存在する場合、`docs/adr/README.md`のインデックスで関連するエントリをスキャンする
3. 一致するADRファイルを読み、ContextとDecisionセクションを提示する
4. 一致するものがない場合：「その決定に関するADRが見つかりません。今すぐ記録しますか？」と回答する

### ADRディレクトリ構成

```
docs/
└── adr/
    ├── README.md              ← index of all ADRs
    ├── 0001-use-nextjs.md
    ├── 0002-postgres-over-mongo.md
    ├── 0003-rest-over-graphql.md
    └── template.md            ← blank template for manual use
```

### ADRインデックス形式

```markdown
# Architecture Decision Records

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0001](0001-use-nextjs.md) | Use Next.js as frontend framework | accepted | 2026-01-15 |
| [0002](0002-postgres-over-mongo.md) | PostgreSQL over MongoDB for primary datastore | accepted | 2026-01-20 |
| [0003](0003-rest-over-graphql.md) | REST API over GraphQL | accepted | 2026-02-01 |
```

## 意思決定検出シグナル

会話中にアーキテクチャ上の意思決定を示す以下のパターンに注目してください：

**明示的シグナル**
- 「Xにしよう」
- 「YではなくXを使うべき」
- 「このトレードオフは〜のため価値がある」
- 「これをADRに記録して」

**暗黙的シグナル**（ADRの記録を提案する — ユーザーの確認なしに自動作成しないこと）
- 2つのフレームワークやライブラリを比較して結論に達した場合
- 根拠を示したデータベーススキーマ設計の選択をした場合
- アーキテクチャパターン間の選択（モノリス vs マイクロサービス、REST vs GraphQL）
- 認証/認可戦略の決定
- 代替案を評価した後のデプロイインフラストラクチャの選択

## 良いADRの条件

### やるべきこと
- **具体的に** — 「ORMを使う」ではなく「Prisma ORMを使う」
- **理由を記録する** — 根拠は「何を」よりも重要
- **却下された代替案を含める** — 将来の開発者は何が検討されたかを知る必要がある
- **結果を正直に記述する** — すべての決定にはトレードオフがある
- **簡潔に保つ** — ADRは2分で読めるべき
- **現在形を使う** — 「Xを使用する予定」ではなく「Xを使用する」

### やってはいけないこと
- 些細な決定を記録する — 変数命名やフォーマットの選択にADRは不要
- エッセイを書く — コンテキストセクションが10行を超える場合は長すぎる
- 代替案を省略する — 「なんとなく選んだ」は有効な根拠ではない
- マークなしにバックフィルする — 過去の決定を記録する場合は元の日付を記載する
- ADRを陳腐化させる — 置き換えられた決定はその代替を参照すべき

## ADRライフサイクル

```
proposed → accepted → [deprecated | superseded by ADR-NNNN]
```

- **proposed**: 意思決定が議論中であり、まだ確定していない
- **accepted**: 意思決定が有効であり、遵守されている
- **deprecated**: 意思決定がもはや関連しない（例：機能が削除された）
- **superseded**: より新しいADRがこれを置き換えた（常に代替をリンクする）

## 記録に値する決定のカテゴリ

| カテゴリ | 例 |
|----------|---------|
| **技術選定** | フレームワーク、言語、データベース、クラウドプロバイダー |
| **アーキテクチャパターン** | モノリス vs マイクロサービス、イベント駆動、CQRS |
| **API設計** | REST vs GraphQL、バージョニング戦略、認証メカニズム |
| **データモデリング** | スキーマ設計、正規化の決定、キャッシュ戦略 |
| **インフラストラクチャ** | デプロイモデル、CI/CDパイプライン、モニタリングスタック |
| **セキュリティ** | 認証戦略、暗号化アプローチ、シークレット管理 |
| **テスト** | テストフレームワーク、カバレッジ目標、E2E vs インテグレーションのバランス |
| **プロセス** | ブランチ戦略、レビュープロセス、リリースケイデンス |

## 他のスキルとの連携

- **Plannerエージェント**: プランナーがアーキテクチャ変更を提案した際にADRの作成を提案する
- **Code reviewerエージェント**: 対応するADRなしにアーキテクチャ変更を導入するPRにフラグを立てる
