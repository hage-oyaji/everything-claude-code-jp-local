# エージェントオーケストレーション

## 利用可能なエージェント

`~/.claude/agents/`に配置：

| エージェント | 用途 | 使用タイミング |
|-------|---------|-------------|
| planner | 実装計画 | 複雑な機能、リファクタリング |
| architect | システム設計 | アーキテクチャ上の意思決定 |
| tdd-guide | テスト駆動開発 | 新機能、バグ修正 |
| code-reviewer | コードレビュー | コード作成後 |
| security-reviewer | セキュリティ分析 | コミット前 |
| build-error-resolver | ビルドエラー修正 | ビルド失敗時 |
| e2e-runner | E2Eテスト | クリティカルなユーザーフロー |
| refactor-cleaner | デッドコード削除 | コードメンテナンス |
| doc-updater | ドキュメント | ドキュメント更新 |
| rust-reviewer | Rustコードレビュー | Rustプロジェクト |

## エージェントの即時使用

ユーザーのプロンプトは不要：
1. 複雑な機能リクエスト - **planner**エージェントを使用
2. コードの作成/変更直後 - **code-reviewer**エージェントを使用
3. バグ修正または新機能 - **tdd-guide**エージェントを使用
4. アーキテクチャ上の意思決定 - **architect**エージェントを使用

## 並列タスク実行

独立した操作には常に並列タスク実行を使用してください：

```markdown
# GOOD: Parallel execution
Launch 3 agents in parallel:
1. Agent 1: Security analysis of auth module
2. Agent 2: Performance review of cache system
3. Agent 3: Type checking of utilities

# BAD: Sequential when unnecessary
First agent 1, then agent 2, then agent 3
```

## マルチパースペクティブ分析

複雑な問題には、役割を分割したサブエージェントを使用してください：
- 事実確認レビュアー
- シニアエンジニア
- セキュリティエキスパート
- 一貫性レビュアー
- 冗長性チェッカー
