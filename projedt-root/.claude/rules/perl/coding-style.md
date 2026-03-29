---
paths:
  - "**/*.pl"
  - "**/*.pm"
  - "**/*.t"
  - "**/*.psgi"
  - "**/*.cgi"
---
# Perl コーディングスタイル

> このファイルは [common/coding-style.md](../common/coding-style.md) を Perl 固有の内容で拡張します。

## 標準

- 常に `use v5.36`（`strict`、`warnings`、`say`、サブルーチンシグネチャを有効化）
- サブルーチンシグネチャを使用 — `@_` を手動でアンパックしない
- 明示的な改行を伴う `print` よりも `say` を優先

## イミュータビリティ

- すべての属性に **Moo** の `is => 'ro'` と `Types::Standard` を使用
- blessed hashref を直接使わない — 常に Moo/Moose アクセサを使用
- **OO オーバーライドに関する注記**: `builder` または `default` を持つ Moo `has` 属性は、算出された読み取り専用値として許容

## フォーマット

以下の設定で **perltidy** を使用：

```
-i=4    # 4-space indent
-l=100  # 100 char line length
-ce     # cuddled else
-bar    # opening brace always right
```

## リンティング

**perlcritic** をシビリティ 3 で、テーマ `core`、`pbp`、`security` を使用：

```bash
perlcritic --severity 3 --theme 'core || pbp || security' lib/
```

## 参考

スキル `perl-patterns` にモダン Perl のイディオムとベストプラクティスの詳細があります。
