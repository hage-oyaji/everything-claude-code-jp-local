---
name: perl-patterns
description: モダンPerl 5.36+のイディオム、ベストプラクティス、堅牢で保守性の高いPerlアプリケーション構築のための規約。
origin: ECC
---

# モダンPerl開発パターン

堅牢で保守性の高いアプリケーションを構築するための、慣用的なPerl 5.36+パターンとベストプラクティス。

## 有効化するタイミング

- 新しいPerlコードやモジュールを書く場合
- Perlコードのイディオム準拠をレビューする場合
- レガシーPerlをモダンな標準にリファクタリングする場合
- Perlモジュールアーキテクチャを設計する場合
- 5.36以前のコードをモダンPerlに移行する場合

## 仕組み

これらのパターンをモダンPerl 5.36+のデフォルトへのバイアスとして適用する: シグネチャ、明示的モジュール、焦点を絞ったエラーハンドリング、テスト可能な境界。以下の例は出発点としてコピーし、実際のアプリケーション、依存関係スタック、デプロイモデルに合わせて調整することを意図している。

## コア原則

### 1. `v5.36` プラグマの使用

単一の `use v5.36` で旧来のボイラープレートを置き換え、strict、warnings、サブルーチンシグネチャを有効にする。

```perl
# Good: Modern preamble
use v5.36;

sub greet($name) {
    say "Hello, $name!";
}

# Bad: Legacy boilerplate
use strict;
use warnings;
use feature 'say', 'signatures';
no warnings 'experimental::signatures';

sub greet {
    my ($name) = @_;
    say "Hello, $name!";
}
```

### 2. サブルーチンシグネチャ

明確さと自動的なアリティチェックのためにシグネチャを使用する。

```perl
use v5.36;

# Good: Signatures with defaults
sub connect_db($host, $port = 5432, $timeout = 30) {
    # $host is required, others have defaults
    return DBI->connect("dbi:Pg:host=$host;port=$port", undef, undef, {
        RaiseError => 1,
        PrintError => 0,
    });
}

# Good: Slurpy parameter for variable args
sub log_message($level, @details) {
    say "[$level] " . join(' ', @details);
}

# Bad: Manual argument unpacking
sub connect_db {
    my ($host, $port, $timeout) = @_;
    $port    //= 5432;
    $timeout //= 30;
    # ...
}
```

### 3. コンテキスト感度

スカラーコンテキストとリストコンテキスト — Perlのコアコンセプト。

```perl
use v5.36;

my @items = (1, 2, 3, 4, 5);

my @copy  = @items;            # List context: all elements
my $count = @items;            # Scalar context: count (5)
say "Items: " . scalar @items; # Force scalar context
```

### 4. ポストフィックスデリファレンス

ネスト構造の可読性のためにポストフィックスデリファレンス構文を使用する。

```perl
use v5.36;

my $data = {
    users => [
        { name => 'Alice', roles => ['admin', 'user'] },
        { name => 'Bob',   roles => ['user'] },
    ],
};

# Good: Postfix dereferencing
my @users = $data->{users}->@*;
my @roles = $data->{users}[0]{roles}->@*;
my %first = $data->{users}[0]->%*;

# Bad: Circumfix dereferencing (harder to read in chains)
my @users = @{ $data->{users} };
my @roles = @{ $data->{users}[0]{roles} };
```

### 5. `isa` 演算子（5.32+）

中置型チェック — `blessed($o) && $o->isa('X')` を置き換える。

```perl
use v5.36;
if ($obj isa 'My::Class') { $obj->do_something }
```

## エラーハンドリング

### eval/dieパターン

```perl
use v5.36;

sub parse_config($path) {
    my $content = eval { path($path)->slurp_utf8 };
    die "Config error: $@" if $@;
    return decode_json($content);
}
```

### Try::Tiny（信頼性の高い例外処理）

```perl
use v5.36;
use Try::Tiny;

sub fetch_user($id) {
    my $user = try {
        $db->resultset('User')->find($id)
            // die "User $id not found\n";
    }
    catch {
        warn "Failed to fetch user $id: $_";
        undef;
    };
    return $user;
}
```

### ネイティブtry/catch（5.40+）

```perl
use v5.40;

sub divide($x, $y) {
    try {
        die "Division by zero" if $y == 0;
        return $x / $y;
    }
    catch ($e) {
        warn "Error: $e";
        return;
    }
}
```

## MooによるモダンOO

軽量でモダンなOOにはMooを優先する。メタプロトコルが必要な場合のみMooseを使用する。

```perl
# Good: Moo class
package User;
use Moo;
use Types::Standard qw(Str Int ArrayRef);
use namespace::autoclean;

has name  => (is => 'ro', isa => Str, required => 1);
has email => (is => 'ro', isa => Str, required => 1);
has age   => (is => 'ro', isa => Int, default  => sub { 0 });
has roles => (is => 'ro', isa => ArrayRef[Str], default => sub { [] });

sub is_admin($self) {
    return grep { $_ eq 'admin' } $self->roles->@*;
}

sub greet($self) {
    return "Hello, I'm " . $self->name;
}

1;

# Usage
my $user = User->new(
    name  => 'Alice',
    email => 'alice@example.com',
    roles => ['admin', 'user'],
);

# Bad: Blessed hashref (no validation, no accessors)
package User;
sub new {
    my ($class, %args) = @_;
    return bless \%args, $class;
}
sub name { return $_[0]->{name} }
1;
```

### Mooロール

```perl
package Role::Serializable;
use Moo::Role;
use JSON::MaybeXS qw(encode_json);
requires 'TO_HASH';
sub to_json($self) { encode_json($self->TO_HASH) }
1;

package User;
use Moo;
with 'Role::Serializable';
has name  => (is => 'ro', required => 1);
has email => (is => 'ro', required => 1);
sub TO_HASH($self) { { name => $self->name, email => $self->email } }
1;
```

### ネイティブ `class` キーワード（5.38+、Corinna）

```perl
use v5.38;
use feature 'class';
no warnings 'experimental::class';

class Point {
    field $x :param;
    field $y :param;
    method magnitude() { sqrt($x**2 + $y**2) }
}

my $p = Point->new(x => 3, y => 4);
say $p->magnitude;  # 5
```

## 正規表現

### 名前付きキャプチャと `/x` フラグ

```perl
use v5.36;

# Good: Named captures with /x for readability
my $log_re = qr{
    ^ (?<timestamp> \d{4}-\d{2}-\d{2} \s \d{2}:\d{2}:\d{2} )
    \s+ \[ (?<level> \w+ ) \]
    \s+ (?<message> .+ ) $
}x;

if ($line =~ $log_re) {
    say "Time: $+{timestamp}, Level: $+{level}";
    say "Message: $+{message}";
}

# Bad: Positional captures (hard to maintain)
if ($line =~ /^(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+\[(\w+)\]\s+(.+)$/) {
    say "Time: $1, Level: $2";
}
```

### コンパイル済みパターン

```perl
use v5.36;

# Good: Compile once, use many
my $email_re = qr/^[A-Za-z0-9._%+-]+\@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$/;

sub validate_emails(@emails) {
    return grep { $_ =~ $email_re } @emails;
}
```

## データ構造

### リファレンスと安全な深いアクセス

```perl
use v5.36;

# Hash and array references
my $config = {
    database => {
        host => 'localhost',
        port => 5432,
        options => ['utf8', 'sslmode=require'],
    },
};

# Safe deep access (returns undef if any level missing)
my $port = $config->{database}{port};           # 5432
my $missing = $config->{cache}{host};           # undef, no error

# Hash slices
my %subset;
@subset{qw(host port)} = @{$config->{database}}{qw(host port)};

# Array slices
my @first_two = $config->{database}{options}->@[0, 1];

# Multi-variable for loop (experimental in 5.36, stable in 5.40)
use feature 'for_list';
no warnings 'experimental::for_list';
for my ($key, $val) (%$config) {
    say "$key => $val";
}
```

## ファイルI/O

### 3引数open

```perl
use v5.36;

# Good: Three-arg open with autodie (core module, eliminates 'or die')
use autodie;

sub read_file($path) {
    open my $fh, '<:encoding(UTF-8)', $path;
    local $/;
    my $content = <$fh>;
    close $fh;
    return $content;
}

# Bad: Two-arg open (shell injection risk, see perl-security)
open FH, $path;            # NEVER do this
open FH, "< $path";        # Still bad — user data in mode string
```

### Path::Tinyによるファイル操作

```perl
use v5.36;
use Path::Tiny;

my $file = path('config', 'app.json');
my $content = $file->slurp_utf8;
$file->spew_utf8($new_content);

# Iterate directory
for my $child (path('src')->children(qr/\.pl$/)) {
    say $child->basename;
}
```

## モジュール構成

### 標準プロジェクトレイアウト

```text
MyApp/
├── lib/
│   └── MyApp/
│       ├── App.pm           # Main module
│       ├── Config.pm        # Configuration
│       ├── DB.pm            # Database layer
│       └── Util.pm          # Utilities
├── bin/
│   └── myapp                # Entry-point script
├── t/
│   ├── 00-load.t            # Compilation tests
│   ├── unit/                # Unit tests
│   └── integration/         # Integration tests
├── cpanfile                 # Dependencies
├── Makefile.PL              # Build system
└── .perlcriticrc            # Linting config
```

### Exporterパターン

```perl
package MyApp::Util;
use v5.36;
use Exporter 'import';

our @EXPORT_OK   = qw(trim);
our %EXPORT_TAGS = (all => \@EXPORT_OK);

sub trim($str) { $str =~ s/^\s+|\s+$//gr }

1;
```

## ツール

### perltidy設定（.perltidyrc）

```text
-i=4        # 4-space indent
-l=100      # 100-char line length
-ci=4       # continuation indent
-ce         # cuddled else
-bar        # opening brace on same line
-nolq       # don't outdent long quoted strings
```

### perlcritic設定（.perlcriticrc）

```ini
severity = 3
theme = core + pbp + security

[InputOutput::RequireCheckedSyscalls]
functions = :builtins
exclude_functions = say print

[Subroutines::ProhibitExplicitReturnUndef]
severity = 4

[ValuesAndExpressions::ProhibitMagicNumbers]
allowed_values = 0 1 2 -1
```

### 依存関係管理（cpanfile + carton）

```bash
cpanm App::cpanminus Carton   # Install tools
carton install                 # Install deps from cpanfile
carton exec -- perl bin/myapp  # Run with local deps
```

```perl
# cpanfile
requires 'Moo', '>= 2.005';
requires 'Path::Tiny';
requires 'JSON::MaybeXS';
requires 'Try::Tiny';

on test => sub {
    requires 'Test2::V0';
    requires 'Test::MockModule';
};
```

## クイックリファレンス: モダンPerlイディオム

| レガシーパターン | モダンな置き換え |
|---|---|
| `use strict; use warnings;` | `use v5.36;` |
| `my ($x, $y) = @_;` | `sub foo($x, $y) { ... }` |
| `@{ $ref }` | `$ref->@*` |
| `%{ $ref }` | `$ref->%*` |
| `open FH, "< $file"` | `open my $fh, '<:encoding(UTF-8)', $file` |
| `blessed hashref` | `Moo` クラスと型 |
| `$1, $2, $3` | `$+{name}`（名前付きキャプチャ） |
| `eval { }; if ($@)` | `Try::Tiny` またはネイティブ `try/catch`（5.40+） |
| `BEGIN { require Exporter; }` | `use Exporter 'import';` |
| 手動ファイル操作 | `Path::Tiny` |
| `blessed($o) && $o->isa('X')` | `$o isa 'X'`（5.32+） |
| `builtin::true / false` | `use builtin 'true', 'false';`（5.36+、実験的） |

## アンチパターン

```perl
# 1. Two-arg open (security risk)
open FH, $filename;                     # NEVER

# 2. Indirect object syntax (ambiguous parsing)
my $obj = new Foo(bar => 1);            # Bad
my $obj = Foo->new(bar => 1);           # Good

# 3. Excessive reliance on $_
map { process($_) } grep { validate($_) } @items;  # Hard to follow
my @valid = grep { validate($_) } @items;           # Better: break it up
my @results = map { process($_) } @valid;

# 4. Disabling strict refs
no strict 'refs';                        # Almost always wrong
${"My::Package::$var"} = $value;         # Use a hash instead

# 5. Global variables as configuration
our $TIMEOUT = 30;                       # Bad: mutable global
use constant TIMEOUT => 30;              # Better: constant
# Best: Moo attribute with default

# 6. String eval for module loading
eval "require $module";                  # Bad: code injection risk
eval "use $module";                      # Bad
use Module::Runtime 'require_module';    # Good: safe module loading
require_module($module);
```

**覚えておくこと**: モダンPerlはクリーンで、読みやすく、安全。`use v5.36` にボイラープレートを任せ、オブジェクトにはMooを使い、手作りソリューションよりCPANの実績あるモジュールを優先する。
