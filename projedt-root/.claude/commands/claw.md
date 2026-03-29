---
description: NanoClaw v2を起動 — ECCの永続的でゼロ依存のREPL。モデルルーティング、スキルのホットロード、ブランチ、コンパクション、エクスポート、メトリクスに対応。
---

# Claw コマンド

永続的なMarkdown履歴と運用制御を備えたインタラクティブなAIエージェントセッションを開始します。

## 使い方

```bash
node scripts/claw.js
```

またはnpm経由:

```bash
npm run claw
```

## 環境変数

| 変数 | デフォルト | 説明 |
|----------|---------|-------------|
| `CLAW_SESSION` | `default` | セッション名（英数字 + ハイフン） |
| `CLAW_SKILLS` | *(空)* | 起動時に読み込むスキル（カンマ区切り） |
| `CLAW_MODEL` | `sonnet` | セッションのデフォルトモデル |

## REPLコマンド

```text
/help                          ヘルプを表示
/clear                         現在のセッション履歴をクリア
/history                       会話履歴全体を表示
/sessions                      保存済みセッション一覧
/model [name]                  モデルの表示/設定
/load <skill-name>             スキルをコンテキストにホットロード
/branch <session-name>         現在のセッションをブランチ
/search <query>                セッション横断で検索
/compact                       古いターンをコンパクトにし、最近のコンテキストを保持
/export <md|json|txt> [path]   セッションをエクスポート
/metrics                       セッションメトリクスを表示
exit                           終了
```

## 注意事項

- NanoClawはゼロ依存を維持しています。
- セッションは `~/.claude/claw/<session>.md` に保存されます。
- コンパクションは最新のターンを保持し、コンパクションヘッダーを書き込みます。
- エクスポートはMarkdown、JSONターン、プレーンテキストに対応しています。
