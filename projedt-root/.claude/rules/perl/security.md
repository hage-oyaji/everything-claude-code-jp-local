---
paths:
  - "**/*.pl"
  - "**/*.pm"
  - "**/*.t"
  - "**/*.psgi"
  - "**/*.cgi"
---
# Perl セキュリティ

> このファイルは [common/security.md](../common/security.md) を Perl 固有の内容で拡張します。

## テイントモード

- すべての CGI/Web 向けスクリプトで `-T` フラグを使用
- 外部コマンド実行前に `%ENV`（`$ENV{PATH}`、`$ENV{CDPATH}` など）をサニタイズ

## 入力バリデーション

- アンテイントにはホワイトリスト正規表現を使用 — `/(.*)/s` は絶対に使わない
- すべてのユーザー入力を明示的なパターンでバリデーション：

```perl
if ($input =~ /\A([a-zA-Z0-9_-]+)\z/) {
    my $clean = $1;
}
```

## ファイル I/O

- **3引数 open のみ** — 2引数 open は使わない
- `Cwd::realpath` でパストラバーサルを防止：

```perl
use Cwd 'realpath';
my $safe_path = realpath($user_path);
die "Path traversal" unless $safe_path =~ m{\A/allowed/directory/};
```

## プロセス実行

- **リスト形式の `system()`** を使用 — 単一文字列形式は使わない
- 出力キャプチャには **IPC::Run3** を使用
- 変数展開を伴うバッククォートは使わない

```perl
system('grep', '-r', $pattern, $directory);  # safe
```

## SQL インジェクション防止

常に DBI プレースホルダを使用 — SQL に直接変数を展開しない：

```perl
my $sth = $dbh->prepare('SELECT * FROM users WHERE email = ?');
$sth->execute($email);
```

## セキュリティスキャン

**perlcritic** をシビリティ 4 以上でセキュリティテーマを使って実行：

```bash
perlcritic --severity 4 --theme security lib/
```

## 参考

スキル `perl-security` に Perl セキュリティパターン、テイントモード、安全な I/O の詳細があります。
