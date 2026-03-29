---
description: C++のTDDワークフローを実施する。先にGoogleTestテストを書き、その後実装する。gcov/lcovでカバレッジを検証する。
---

# C++ TDD コマンド

このコマンドはGoogleTest/GoogleMockとCMake/CTestを使用した、C++コードのテスト駆動開発手法を実施します。

## このコマンドの動作

1. **インターフェースを定義**: まずクラス/関数のシグネチャをスキャフォールドする
2. **テストを書く**: 包括的なGoogleTestテストケースを作成する（RED）
3. **テストを実行**: テストが正しい理由で失敗することを確認する
4. **コードを実装**: テストを通過する最小限のコードを書く（GREEN）
5. **リファクタリング**: テストをグリーンに保ちながら改善する
6. **カバレッジを確認**: 80%以上のカバレッジを確保する

## 使用場面

`/cpp-test` を使用する場面:
- 新しいC++の関数やクラスを実装する時
- 既存コードにテストカバレッジを追加する時
- バグ修正時（先にテストを失敗させてから修正）
- 重要なビジネスロジックを構築する時
- C++でのTDDワークフローを学習する時

## TDDサイクル

```
RED     → 失敗するGoogleTestテストを書く
GREEN   → テストを通過する最小限のコードを実装
REFACTOR → コードを改善、テストはグリーンを維持
REPEAT  → 次のテストケースへ
```

## セッション例

````
User: /cpp-test I need a function to validate email addresses

Agent:
# TDDセッション: メールバリデーター

## ステップ1: インターフェースの定義

```cpp
// validator/email.hpp
#pragma once
#include <string>
#include <expected>

enum class EmailError {
    Empty,
    InvalidFormat
};

std::expected<void, EmailError> validate_email(const std::string& email);
```

## ステップ2: テストを書く（RED）

```cpp
// validator/email_test.cpp
#include <gtest/gtest.h>
#include "email.hpp"

TEST(ValidateEmail, AcceptsSimpleEmail) {
    auto result = validate_email("user@example.com");
    EXPECT_TRUE(result.has_value());
}

TEST(ValidateEmail, AcceptsSubdomain) {
    EXPECT_TRUE(validate_email("user@mail.example.com").has_value());
}

TEST(ValidateEmail, AcceptsPlus) {
    EXPECT_TRUE(validate_email("user+tag@example.com").has_value());
}

TEST(ValidateEmail, RejectsEmpty) {
    auto result = validate_email("");
    ASSERT_FALSE(result.has_value());
    EXPECT_EQ(result.error(), EmailError::Empty);
}

TEST(ValidateEmail, RejectsNoAtSign) {
    EXPECT_FALSE(validate_email("userexample.com").has_value());
}

TEST(ValidateEmail, RejectsNoDomain) {
    EXPECT_FALSE(validate_email("user@").has_value());
}

TEST(ValidateEmail, RejectsNoLocalPart) {
    EXPECT_FALSE(validate_email("@example.com").has_value());
}
```

## ステップ3: テスト実行 - 失敗を確認

```bash
$ cmake --build build && ctest --test-dir build --output-on-failure

1/1 Test #1: email_validator_test .....***Failed
    --- undefined reference to `validate_email`

FAIL
```

✓ テストが期待通りに失敗（未実装）。

## ステップ4: 最小限のコードを実装（GREEN）

```cpp
// validator/email.cpp
#include "email.hpp"
#include <regex>

std::expected<void, EmailError> validate_email(const std::string& email) {
    if (email.empty()) {
        return std::unexpected(EmailError::Empty);
    }
    static const std::regex pattern(R"([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})");
    if (!std::regex_match(email, pattern)) {
        return std::unexpected(EmailError::InvalidFormat);
    }
    return {};
}
```

## ステップ5: テスト実行 - 合格を確認

```bash
$ cmake --build build && ctest --test-dir build --output-on-failure

1/1 Test #1: email_validator_test .....   Passed    0.01 sec

100% tests passed.
```

✓ すべてのテスト合格！

## ステップ6: カバレッジ確認

```bash
$ cmake -DCMAKE_CXX_FLAGS="--coverage" -B build && cmake --build build
$ ctest --test-dir build
$ lcov --capture --directory build --output-file coverage.info
$ lcov --list coverage.info

validator/email.cpp     | 100%
```

✓ カバレッジ: 100%

## TDD完了！
````

## テストパターン

### 基本テスト
```cpp
TEST(SuiteName, TestName) {
    EXPECT_EQ(add(2, 3), 5);
    EXPECT_NE(result, nullptr);
    EXPECT_TRUE(is_valid);
    EXPECT_THROW(func(), std::invalid_argument);
}
```

### フィクスチャ
```cpp
class DatabaseTest : public ::testing::Test {
protected:
    void SetUp() override { db_ = create_test_db(); }
    void TearDown() override { db_.reset(); }
    std::unique_ptr<Database> db_;
};

TEST_F(DatabaseTest, InsertsRecord) {
    db_->insert("key", "value");
    EXPECT_EQ(db_->get("key"), "value");
}
```

### パラメタライズドテスト
```cpp
class PrimeTest : public ::testing::TestWithParam<std::pair<int, bool>> {};

TEST_P(PrimeTest, ChecksPrimality) {
    auto [input, expected] = GetParam();
    EXPECT_EQ(is_prime(input), expected);
}

INSTANTIATE_TEST_SUITE_P(Primes, PrimeTest, ::testing::Values(
    std::make_pair(2, true),
    std::make_pair(4, false),
    std::make_pair(7, true)
));
```

## カバレッジコマンド

```bash
# カバレッジ付きビルド
cmake -DCMAKE_CXX_FLAGS="--coverage" -DCMAKE_EXE_LINKER_FLAGS="--coverage" -B build

# テスト実行
cmake --build build && ctest --test-dir build

# カバレッジレポート生成
lcov --capture --directory build --output-file coverage.info
lcov --remove coverage.info '/usr/*' --output-file coverage.info
genhtml coverage.info --output-directory coverage_html
```

## カバレッジ目標

| コードの種類 | 目標 |
|-----------|--------|
| 重要なビジネスロジック | 100% |
| パブリックAPI | 90%以上 |
| 一般的なコード | 80%以上 |
| 生成されたコード | 除外 |

## TDDのベストプラクティス

**推奨:**
- 実装の前にテストを先に書く
- 変更ごとにテストを実行する
- 適切な場合は `ASSERT_*`（停止）より `EXPECT_*`（続行）を使用する
- 実装の詳細ではなく、振る舞いをテストする
- エッジケースを含める（空、null、最大値、境界条件）

**非推奨:**
- テストの前に実装を書くこと
- REDフェーズをスキップすること
- プライベートメソッドを直接テストすること（パブリックAPIを通じてテストする）
- テストで `sleep` を使用すること
- フレーキーテストを無視すること

## 関連コマンド

- `/cpp-build` - ビルドエラーの修正
- `/cpp-review` - 実装後のコードレビュー
- `/verify` - 完全な検証ループの実行

## 関連

- スキル: `skills/cpp-testing/`
- スキル: `skills/tdd-workflow/`
