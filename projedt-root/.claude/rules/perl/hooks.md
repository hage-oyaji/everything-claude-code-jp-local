---
paths:
  - "**/*.pl"
  - "**/*.pm"
  - "**/*.t"
  - "**/*.psgi"
  - "**/*.cgi"
---
# Perl フック

> このファイルは [common/hooks.md](../common/hooks.md) を Perl 固有の内容で拡張します。

## PostToolUse フック

`~/.claude/settings.json` で設定：

- **perltidy**: 編集後に `.pl` および `.pm` ファイルを自動フォーマット
- **perlcritic**: `.pm` ファイル編集後にリントチェックを実行

## 警告

- スクリプトでない `.pm` ファイルでの `print` について警告 — `say` またはロギングモジュール（例: `Log::Any`）を使用
