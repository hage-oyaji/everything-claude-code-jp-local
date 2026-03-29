---
paths:
  - "**/*.pl"
  - "**/*.pm"
  - "**/*.t"
  - "**/*.psgi"
  - "**/*.cgi"
---
# Perl テスト

> このファイルは [common/testing.md](../common/testing.md) を Perl 固有の内容で拡張します。

## フレームワーク

新規プロジェクトには **Test2::V0** を使用（Test::More ではなく）：

```perl
use Test2::V0;

is($result, 42, 'answer is correct');

done_testing;
```

## ランナー

```bash
prove -l t/              # adds lib/ to @INC
prove -lr -j8 t/         # recursive, 8 parallel jobs
```

`lib/` が `@INC` に含まれるよう、常に `-l` を使用。

## カバレッジ

**Devel::Cover** を使用 — 80% 以上を目標：

```bash
cover -test
```

## モッキング

- **Test::MockModule** — 既存モジュールのメソッドをモック
- **Test::MockObject** — テストダブルをゼロから作成

## 注意点

- テストファイルは必ず `done_testing` で終了
- `prove` で `-l` フラグを忘れない

## 参考

スキル `perl-testing` に Test2::V0、prove、Devel::Cover を使った Perl TDD パターンの詳細があります。
