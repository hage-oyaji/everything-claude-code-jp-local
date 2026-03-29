# CLAUDE.md

このファイルは、このリポジトリのコードを扱う際のClaude Code（claude.ai/code）向けガイダンスを提供します。

## プロジェクト概要

これは**Claude Codeプラグイン**です。本番運用可能なエージェント、スキル、フック、コマンド、ルール、MCP設定のコレクションです。このプロジェクトは、Claude Codeを使用したソフトウェア開発のための実戦検証済みワークフローを提供します。

## テストの実行

```bash
# Run all tests
node tests/run-all.js

# Run individual test files
node tests/lib/utils.test.js
node tests/lib/package-manager.test.js
node tests/hooks/hooks.test.js
```

## アーキテクチャ

プロジェクトは複数のコアコンポーネントで構成されています：

- **agents/** - 委譲用の特化型サブエージェント（planner、code-reviewer、tdd-guideなど）
- **skills/** - ワークフロー定義とドメイン知識（コーディング標準、パターン、テスト）
- **commands/** - ユーザーが呼び出すスラッシュコマンド（/tdd、/plan、/e2eなど）
- **hooks/** - トリガーベースの自動化（セッション永続化、ツール実行前後のフック）
- **rules/** - 常時適用ガイドライン（セキュリティ、コーディングスタイル、テスト要件）
- **mcp-configs/** - 外部連携用のMCPサーバー設定
- **scripts/** - フックとセットアップ用のクロスプラットフォームNode.jsユーティリティ
- **tests/** - スクリプトとユーティリティのテストスイート

## 主要コマンド

- `/tdd` - テスト駆動開発ワークフロー
- `/plan` - 実装計画
- `/e2e` - E2Eテストの生成と実行
- `/code-review` - 品質レビュー
- `/build-fix` - ビルドエラーの修正
- `/learn` - セッションからパターンを抽出
- `/skill-create` - gitの履歴からスキルを生成

## 開発メモ

- パッケージマネージャー検出：npm、pnpm、yarn、bun（`CLAUDE_PACKAGE_MANAGER`環境変数またはプロジェクト設定で構成可能）
- クロスプラットフォーム：Node.jsスクリプトによるWindows、macOS、Linuxサポート
- エージェント形式：YAMLフロントマター付きMarkdown（name、description、tools、model）
- スキル形式：明確なセクション構成のMarkdown（使用場面、動作方法、例）
- スキル配置：skills/にキュレーション済み、~/.claude/skills/に生成/インポート済み。docs/SKILL-PLACEMENT-POLICY.mdを参照
- フック形式：マッチャー条件とコマンド/通知フック付きJSON

## コントリビューション

CONTRIBUTING.mdのフォーマットに従ってください：
- エージェント：フロントマター付きMarkdown（name、description、tools、model）
- スキル：明確なセクション構成（When to Use、How It Works、Examples）
- コマンド：descriptionフロントマター付きMarkdown
- フック：matcherとhooks配列を含むJSON

ファイル命名規則：小文字のハイフン区切り（例：`python-reviewer.md`、`tdd-workflow.md`）
