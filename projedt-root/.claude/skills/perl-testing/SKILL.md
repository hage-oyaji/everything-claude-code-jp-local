---
name: perl-testing
description: Test2::V0、Test::More、proveランナー、モッキング、Devel::Coverによるカバレッジ、TDD方法論を使用したPerlテストパターン。
origin: ECC
---

# Perlテストパターン

Test2::V0、Test::More、prove、TDD方法論を使用したPerlアプリケーションの包括的なテスト戦略。

## 有効化するタイミング

- 新しいPerlコードを書く場合（TDDに従う: レッド、グリーン、リファクタリング）
- Perlモジュールやアプリケーションのテストスイートを設計する場合
- Perlテストカバレッジをレビューする場合
- Perlテスト基盤をセットアップする場合
- テストをTest::MoreからTest2::V0に移行する場合
- 失敗するPerlテストをデバッグする場合

## TDDワークフロー

常にレッド-グリーン-リファクタリングサイクルに従う。

```perl
# Step 1: RED — Write a failing test
# t/unit/calculator.t
use v5.36;
use Test2::V0;

use lib 'lib';
use Calculator;

subtest 'addition' => sub {
    my $calc = Calculator->new;
    is($calc->add(2, 3), 5, 'adds two numbers');
    is($calc->add(-1, 1), 0, 'handles negatives');
};

done_testing;

# Step 2: GREEN — Write minimal implementation
# lib/Calculator.pm
package Calculator;
use v5.36;
use Moo;

sub add($self, $a, $b) {
    return $a + $b;
}

1;

# Step 3: REFACTOR — Improve while tests stay green
# Run: prove -lv t/unit/calculator.t
```

## Test::Moreの基礎

標準的なPerlテストモジュール — 広く使用され、コアに同梱。

### 基本アサーション

```perl
use v5.36;
use Test::More;

# Plan upfront or use done_testing
# plan tests => 5;  # Fixed plan (optional)

# Equality
is($result, 42, 'returns correct value');
isnt($result, 0, 'not zero');

# Boolean
ok($user->is_active, 'user is active');
ok(!$user->is_banned, 'user is not banned');

# Deep comparison
is_deeply(
    $got,
    { name => 'Alice', roles => ['admin'] },
    'returns expected structure'
);

# Pattern matching
like($error, qr/not found/i, 'error mentions not found');
unlike($output, qr/password/, 'output hides password');

# Type check
isa_ok($obj, 'MyApp::User');
can_ok($obj, 'save', 'delete');

done_testing;
```

### SKIPとTODO

```perl
use v5.36;
use Test::More;

# Skip tests conditionally
SKIP: {
    skip 'No database configured', 2 unless $ENV{TEST_DB};

    my $db = connect_db();
    ok($db->ping, 'database is reachable');
    is($db->version, '15', 'correct PostgreSQL version');
}

# Mark expected failures
TODO: {
    local $TODO = 'Caching not yet implemented';
    is($cache->get('key'), 'value', 'cache returns value');
}

done_testing;
```

## Test2::V0モダンフレームワーク

Test2::V0はTest::Moreのモダンな置き換え — より豊富なアサーション、より良い診断、拡張可能。

### なぜTest2なのか

- ハッシュ/配列ビルダーによる優れた深い比較
- 失敗時のより良い診断出力
- よりクリーンなスコープを持つサブテスト
- Test2::Tools::*プラグインで拡張可能
- Test::Moreテストとの後方互換性

### ビルダーによる深い比較

```perl
use v5.36;
use Test2::V0;

# Hash builder — check partial structure
is(
    $user->to_hash,
    hash {
        field name  => 'Alice';
        field email => match(qr/\@example\.com$/);
        field age   => validator(sub { $_ >= 18 });
        # Ignore other fields
        etc();
    },
    'user has expected fields'
);

# Array builder
is(
    $result,
    array {
        item 'first';
        item match(qr/^second/);
        item DNE();  # Does Not Exist — verify no extra items
    },
    'result matches expected list'
);

# Bag — order-independent comparison
is(
    $tags,
    bag {
        item 'perl';
        item 'testing';
        item 'tdd';
    },
    'has all required tags regardless of order'
);
```

### サブテスト

```perl
use v5.36;
use Test2::V0;

subtest 'User creation' => sub {
    my $user = User->new(name => 'Alice', email => 'alice@example.com');
    ok($user, 'user object created');
    is($user->name, 'Alice', 'name is set');
    is($user->email, 'alice@example.com', 'email is set');
};

subtest 'User validation' => sub {
    my $warnings = warns {
        User->new(name => '', email => 'bad');
    };
    ok($warnings, 'warns on invalid data');
};

done_testing;
```

### Test2による例外テスト

```perl
use v5.36;
use Test2::V0;

# Test that code dies
like(
    dies { divide(10, 0) },
    qr/Division by zero/,
    'dies on division by zero'
);

# Test that code lives
ok(lives { divide(10, 2) }, 'division succeeds') or note($@);

# Combined pattern
subtest 'error handling' => sub {
    ok(lives { parse_config('valid.json') }, 'valid config parses');
    like(
        dies { parse_config('missing.json') },
        qr/Cannot open/,
        'missing file dies with message'
    );
};

done_testing;
```

## テスト構成とprove

### ディレクトリ構造

```text
t/
├── 00-load.t              # Verify modules compile
├── 01-basic.t             # Core functionality
├── unit/
│   ├── config.t           # Unit tests by module
│   ├── user.t
│   └── util.t
├── integration/
│   ├── database.t
│   └── api.t
├── lib/
│   └── TestHelper.pm      # Shared test utilities
└── fixtures/
    ├── config.json        # Test data files
    └── users.csv
```

### proveコマンド

```bash
# Run all tests
prove -l t/

# Verbose output
prove -lv t/

# Run specific test
prove -lv t/unit/user.t

# Recursive search
prove -lr t/

# Parallel execution (8 jobs)
prove -lr -j8 t/

# Run only failing tests from last run
prove -l --state=failed t/

# Colored output with timer
prove -l --color --timer t/

# TAP output for CI
prove -l --formatter TAP::Formatter::JUnit t/ > results.xml
```

### .proverc設定

```text
-l
--color
--timer
-r
-j4
--state=save
```

## フィクスチャとセットアップ/ティアダウン

### サブテストの分離

```perl
use v5.36;
use Test2::V0;
use File::Temp qw(tempdir);
use Path::Tiny;

subtest 'file processing' => sub {
    # Setup
    my $dir = tempdir(CLEANUP => 1);
    my $file = path($dir, 'input.txt');
    $file->spew_utf8("line1\nline2\nline3\n");

    # Test
    my $result = process_file("$file");
    is($result->{line_count}, 3, 'counts lines');

    # Teardown happens automatically (CLEANUP => 1)
};
```

### 共有テストヘルパー

再利用可能なヘルパーを `t/lib/TestHelper.pm` に配置し、`use lib 't/lib'` でロードする。`create_test_db()`、`create_temp_dir()`、`fixture_path()` などのファクトリー関数を `Exporter` でエクスポートする。

## モッキング

### Test::MockModule

```perl
use v5.36;
use Test2::V0;
use Test::MockModule;

subtest 'mock external API' => sub {
    my $mock = Test::MockModule->new('MyApp::API');

    # Good: Mock returns controlled data
    $mock->mock(fetch_user => sub ($self, $id) {
        return { id => $id, name => 'Mock User', email => 'mock@test.com' };
    });

    my $api = MyApp::API->new;
    my $user = $api->fetch_user(42);
    is($user->{name}, 'Mock User', 'returns mocked user');

    # Verify call count
    my $call_count = 0;
    $mock->mock(fetch_user => sub { $call_count++; return {} });
    $api->fetch_user(1);
    $api->fetch_user(2);
    is($call_count, 2, 'fetch_user called twice');

    # Mock is automatically restored when $mock goes out of scope
};

# Bad: Monkey-patching without restoration
# *MyApp::API::fetch_user = sub { ... };  # NEVER — leaks across tests
```

軽量なモックオブジェクトには、`Test::MockObject` を使用して `->mock()` で注入可能なテストダブルを作成し、`->called_ok()` で呼び出しを検証する。

## Devel::Coverによるカバレッジ

### カバレッジの実行

```bash
# Basic coverage report
cover -test

# Or step by step
perl -MDevel::Cover -Ilib t/unit/user.t
cover

# HTML report
cover -report html
open cover_db/coverage.html

# Specific thresholds
cover -test -report text | grep 'Total'

# CI-friendly: fail under threshold
cover -test && cover -report text -select '^lib/' \
  | perl -ne 'if (/Total.*?(\d+\.\d+)/) { exit 1 if $1 < 80 }'
```

### インテグレーションテスト

データベーステストにはインメモリSQLite、APIテストにはHTTP::Tinyのモックを使用する。

```perl
use v5.36;
use Test2::V0;
use DBI;

subtest 'database integration' => sub {
    my $dbh = DBI->connect('dbi:SQLite:dbname=:memory:', '', '', {
        RaiseError => 1,
    });
    $dbh->do('CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)');

    $dbh->prepare('INSERT INTO users (name) VALUES (?)')->execute('Alice');
    my $row = $dbh->selectrow_hashref('SELECT * FROM users WHERE name = ?', undef, 'Alice');
    is($row->{name}, 'Alice', 'inserted and retrieved user');
};

done_testing;
```

## ベストプラクティス

### 推奨

- **TDDに従う**: 実装前にテストを書く（レッド-グリーン-リファクタリング）
- **Test2::V0を使用する**: モダンなアサーション、より良い診断
- **サブテストを使用する**: 関連するアサーションをグループ化し、状態を分離
- **外部依存関係をモックする**: ネットワーク、データベース、ファイルシステム
- **`prove -l` を使用する**: 常にlib/を `@INC` に含める
- **テストに明確な名前を付ける**: `'user login with invalid password fails'`
- **エッジケースをテストする**: 空文字列、undef、ゼロ、境界値
- **80%以上のカバレッジを目指す**: ビジネスロジックパスに焦点
- **テストを高速に保つ**: I/Oをモック、インメモリデータベースを使用

### 非推奨

- **実装をテストしない**: 内部ではなく動作と出力をテスト
- **サブテスト間で状態を共有しない**: 各サブテストは独立
- **`done_testing` を省略しない**: すべての計画されたテストが実行されたことを確認
- **過度にモックしない**: 境界のみモック、テスト対象コードはモックしない
- **新規プロジェクトに `Test::More` を使用しない**: Test2::V0を優先
- **テスト失敗を無視しない**: マージ前にすべてのテストがパスする必要がある
- **CPANモジュールをテストしない**: ライブラリが正しく動作することを信頼する
- **脆いテストを書かない**: 過度に具体的な文字列マッチングを避ける

## クイックリファレンス

| タスク | コマンド / パターン |
|---|---|
| 全テスト実行 | `prove -lr t/` |
| 1つのテストを詳細実行 | `prove -lv t/unit/user.t` |
| 並列テスト実行 | `prove -lr -j8 t/` |
| カバレッジレポート | `cover -test && cover -report html` |
| 等値テスト | `is($got, $expected, 'label')` |
| 深い比較 | `is($got, hash { field k => 'v'; etc() }, 'label')` |
| 例外テスト | `like(dies { ... }, qr/msg/, 'label')` |
| 例外なしテスト | `ok(lives { ... }, 'label')` |
| メソッドのモック | `Test::MockModule->new('Pkg')->mock(m => sub { ... })` |
| テストスキップ | `SKIP: { skip 'reason', $count unless $cond; ... }` |
| TODOテスト | `TODO: { local $TODO = 'reason'; ... }` |

## よくある落とし穴

### `done_testing` の忘れ

```perl
# Bad: Test file runs but doesn't verify all tests executed
use Test2::V0;
is(1, 1, 'works');
# Missing done_testing — silent bugs if test code is skipped

# Good: Always end with done_testing
use Test2::V0;
is(1, 1, 'works');
done_testing;
```

### `-l` フラグの不足

```bash
# Bad: Modules in lib/ not found
prove t/unit/user.t
# Can't locate MyApp/User.pm in @INC

# Good: Include lib/ in @INC
prove -l t/unit/user.t
```

### 過度なモッキング

*依存関係*をモックし、テスト対象のコードをモックしない。テストがモックに設定した値が返されることだけを検証しているなら、何もテストしていない。

### テスト汚染

サブテスト内で `my` 変数を使用する — テスト間の状態リークを防ぐために `our` は絶対に使わない。

**覚えておくこと**: テストはセーフティネット。高速で、焦点を絞り、独立に保つ。新規プロジェクトにはTest2::V0、実行にはprove、説明責任にはDevel::Coverを使用する。
