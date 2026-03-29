---
name: agent-eval
description: コーディングエージェント（Claude Code、Aider、Codexなど）のカスタムタスクによる直接比較。合格率、コスト、時間、一貫性メトリクスを測定
origin: ECC
tools: Read, Write, Edit, Bash, Grep, Glob
---

# エージェント評価スキル

コーディングエージェントを再現可能なタスクで直接比較するための軽量CLIツール。「どのコーディングエージェントが最適か？」という比較はすべて感覚に頼りがちですが、このツールはそれを体系化します。

## 起動条件

- コーディングエージェント（Claude Code、Aider、Codexなど）を自分のコードベースで比較する場合
- 新しいツールやモデルを導入する前にエージェントのパフォーマンスを測定する場合
- エージェントのモデルやツールが更新された際のリグレッションチェックを実行する場合
- チームのためにデータに基づいたエージェント選定を行う場合

## インストール

> **注意:** agent-evalはソースを確認した上で、リポジトリからインストールしてください。

## コアコンセプト

### YAMLタスク定義

タスクを宣言的に定義します。各タスクでは、何を行うか、どのファイルに触れるか、どのように成功を判定するかを指定します：

```yaml
name: add-retry-logic
description: Add exponential backoff retry to the HTTP client
repo: ./my-project
files:
  - src/http_client.py
prompt: |
  Add retry logic with exponential backoff to all HTTP requests.
  Max 3 retries. Initial delay 1s, max delay 30s.
judge:
  - type: pytest
    command: pytest tests/test_http_client.py -v
  - type: grep
    pattern: "exponential_backoff|retry"
    files: src/http_client.py
commit: "abc1234"  # pin to specific commit for reproducibility
```

### Gitワークツリー分離

各エージェント実行は独自のgitワークツリーで実行されます — Dockerは不要です。これにより再現性の分離が保証され、エージェント同士が干渉したりベースリポジトリを破損したりすることがありません。

### 収集メトリクス

| メトリクス | 測定内容 |
|--------|-----------------|
| 合格率 | エージェントが生成したコードはジャッジに合格したか？ |
| コスト | タスクごとのAPI費用（取得可能な場合） |
| 時間 | 完了までの実時間（秒） |
| 一貫性 | 繰り返し実行での合格率（例：3/3 = 100%） |

## ワークフロー

### 1. タスクの定義

`tasks/`ディレクトリにYAMLファイルを作成します（タスクごとに1ファイル）：

```bash
mkdir tasks
# Write task definitions (see template above)
```

### 2. エージェントの実行

タスクに対してエージェントを実行します：

```bash
agent-eval run --task tasks/add-retry-logic.yaml --agent claude-code --agent aider --runs 3
```

各実行の流れ：
1. 指定されたコミットから新しいgitワークツリーを作成
2. プロンプトをエージェントに渡す
3. ジャッジ基準を実行
4. 合格/不合格、コスト、時間を記録

### 3. 結果の比較

比較レポートを生成します：

```bash
agent-eval report --format table
```

```
Task: add-retry-logic (3 runs each)
┌──────────────┬───────────┬────────┬────────┬─────────────┐
│ Agent        │ Pass Rate │ Cost   │ Time   │ Consistency │
├──────────────┼───────────┼────────┼────────┼─────────────┤
│ claude-code  │ 3/3       │ $0.12  │ 45s    │ 100%        │
│ aider        │ 2/3       │ $0.08  │ 38s    │  67%        │
└──────────────┴───────────┴────────┴────────┴─────────────┘
```

## ジャッジタイプ

### コードベース（決定論的）

```yaml
judge:
  - type: pytest
    command: pytest tests/ -v
  - type: command
    command: npm run build
```

### パターンベース

```yaml
judge:
  - type: grep
    pattern: "class.*Retry"
    files: src/**/*.py
```

### モデルベース（LLMによるジャッジ）

```yaml
judge:
  - type: llm
    prompt: |
      Does this implementation correctly handle exponential backoff?
      Check for: max retries, increasing delays, jitter.
```

## ベストプラクティス

- **実際のワークロードを代表する3〜5個のタスク**から始める（トイサンプルではなく）
- エージェントごとに**少なくとも3回の試行**を実施してばらつきを把握する — エージェントは非決定的
- タスクYAMLで**コミットを固定**し、日や週をまたいで結果を再現可能にする
- タスクごとに**少なくとも1つの決定論的ジャッジ**（テスト、ビルド）を含める — LLMジャッジはノイズを加える
- **合格率とともにコストを追跡**する — 10倍のコストで95%のエージェントが最適とは限らない
- **タスク定義をバージョン管理**する — テストフィクスチャと同様にコードとして扱う

## リンク

- リポジトリ: [github.com/joaquinhuigomez/agent-eval](https://github.com/joaquinhuigomez/agent-eval)
