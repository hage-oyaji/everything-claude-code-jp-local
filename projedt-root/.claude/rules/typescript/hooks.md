---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
---
# TypeScript/JavaScript フック

> このファイルは [common/hooks.md](../common/hooks.md) を TypeScript/JavaScript 固有の内容で拡張します。

## PostToolUse フック

`~/.claude/settings.json` で設定：

- **Prettier**: 編集後に JS/TS ファイルを自動フォーマット
- **TypeScript チェック**: `.ts`/`.tsx` ファイル編集後に `tsc` を実行
- **console.log 警告**: 編集されたファイル内の `console.log` について警告

## Stop フック

- **console.log 監査**: セッション終了前にすべての変更ファイルで `console.log` をチェック
