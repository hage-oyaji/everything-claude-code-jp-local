---
paths:
  - "**/*.pl"
  - "**/*.pm"
  - "**/*.t"
  - "**/*.psgi"
  - "**/*.cgi"
---
# Perl パターン

> このファイルは [common/patterns.md](../common/patterns.md) を Perl 固有の内容で拡張します。

## リポジトリパターン

インターフェースの背後に **DBI** または **DBIx::Class** を使用：

```perl
package MyApp::Repo::User;
use Moo;

has dbh => (is => 'ro', required => 1);

sub find_by_id ($self, $id) {
    my $sth = $self->dbh->prepare('SELECT * FROM users WHERE id = ?');
    $sth->execute($id);
    return $sth->fetchrow_hashref;
}
```

## DTO / バリューオブジェクト

**Types::Standard** を使った **Moo** クラス（Python の dataclass に相当）：

```perl
package MyApp::DTO::User;
use Moo;
use Types::Standard qw(Str Int);

has name  => (is => 'ro', isa => Str, required => 1);
has email => (is => 'ro', isa => Str, required => 1);
has age   => (is => 'ro', isa => Int);
```

## リソース管理

- `autodie` を伴う **3引数 open** を常に使用
- ファイル操作には **Path::Tiny** を使用

```perl
use autodie;
use Path::Tiny;

my $content = path('config.json')->slurp_utf8;
```

## モジュールインターフェース

`Exporter 'import'` と `@EXPORT_OK` を使用 — `@EXPORT` は使わない：

```perl
use Exporter 'import';
our @EXPORT_OK = qw(parse_config validate_input);
```

## 依存関係管理

再現可能なインストールのために **cpanfile** + **carton** を使用：

```bash
carton install
carton exec prove -lr t/
```

## 参考

スキル `perl-patterns` にモダン Perl パターンとイディオムの詳細があります。
