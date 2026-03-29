---
name: cpp-testing
description: C++テストの作成・更新・修正、GoogleTest/CTestの設定、失敗やフレーキーテストの診断、カバレッジ/サニタイザーの追加時にのみ使用。
origin: ECC
---

# C++テスト（エージェントスキル）

GoogleTest/GoogleMockとCMake/CTestを使用した、モダンC++（C++17/20）のエージェント重視テストワークフロー。

## 使用するとき

- 新しいC++テストを書くとき、または既存のテストを修正するとき
- C++コンポーネントのユニット/統合テストカバレッジを設計するとき
- テストカバレッジ、CIゲーティング、リグレッション保護を追加するとき
- 一貫した実行のためのCMake/CTestワークフローを設定するとき
- テスト失敗やフレーキーな挙動を調査するとき
- メモリ/レース診断のためのサニタイザーを有効化するとき

### 使用しないとき

- テスト変更を伴わない新しいプロダクト機能の実装
- テストカバレッジや失敗と無関係な大規模リファクタリング
- 検証すべきテストリグレッションなしのパフォーマンスチューニング
- C++以外のプロジェクトまたはテスト以外のタスク

## コアコンセプト

- **TDDループ**: red → green → refactor（テストファースト、最小限の修正、クリーンアップ）。
- **分離**: グローバルステートよりも依存性注入とフェイクを優先。
- **テストレイアウト**: `tests/unit`、`tests/integration`、`tests/testdata`。
- **モック vs フェイク**: インタラクションにはモック、ステートフルな振る舞いにはフェイク。
- **CTestディスカバリー**: 安定したテストディスカバリーには`gtest_discover_tests()`を使用。
- **CIシグナル**: まずサブセットを実行し、次にフルスイートを`--output-on-failure`で実行。

## TDDワークフロー

RED → GREEN → REFACTORループに従う:

1. **RED**: 新しい振る舞いをキャプチャする失敗テストを書く
2. **GREEN**: パスするための最小限の変更を実装する
3. **REFACTOR**: テストがグリーンのままクリーンアップする

```cpp
// tests/add_test.cpp
#include <gtest/gtest.h>

int Add(int a, int b); // Provided by production code.

TEST(AddTest, AddsTwoNumbers) { // RED
  EXPECT_EQ(Add(2, 3), 5);
}

// src/add.cpp
int Add(int a, int b) { // GREEN
  return a + b;
}

// REFACTOR: simplify/rename once tests pass
```

## コード例

### 基本ユニットテスト（gtest）

```cpp
// tests/calculator_test.cpp
#include <gtest/gtest.h>

int Add(int a, int b); // Provided by production code.

TEST(CalculatorTest, AddsTwoNumbers) {
    EXPECT_EQ(Add(2, 3), 5);
}
```

### フィクスチャ（gtest）

```cpp
// tests/user_store_test.cpp
// Pseudocode stub: replace UserStore/User with project types.
#include <gtest/gtest.h>
#include <memory>
#include <optional>
#include <string>

struct User { std::string name; };
class UserStore {
public:
    explicit UserStore(std::string /*path*/) {}
    void Seed(std::initializer_list<User> /*users*/) {}
    std::optional<User> Find(const std::string &/*name*/) { return User{"alice"}; }
};

class UserStoreTest : public ::testing::Test {
protected:
    void SetUp() override {
        store = std::make_unique<UserStore>(":memory:");
        store->Seed({{"alice"}, {"bob"}});
    }

    std::unique_ptr<UserStore> store;
};

TEST_F(UserStoreTest, FindsExistingUser) {
    auto user = store->Find("alice");
    ASSERT_TRUE(user.has_value());
    EXPECT_EQ(user->name, "alice");
}
```

### モック（gmock）

```cpp
// tests/notifier_test.cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>
#include <string>

class Notifier {
public:
    virtual ~Notifier() = default;
    virtual void Send(const std::string &message) = 0;
};

class MockNotifier : public Notifier {
public:
    MOCK_METHOD(void, Send, (const std::string &message), (override));
};

class Service {
public:
    explicit Service(Notifier &notifier) : notifier_(notifier) {}
    void Publish(const std::string &message) { notifier_.Send(message); }

private:
    Notifier &notifier_;
};

TEST(ServiceTest, SendsNotifications) {
    MockNotifier notifier;
    Service service(notifier);

    EXPECT_CALL(notifier, Send("hello")).Times(1);
    service.Publish("hello");
}
```

### CMake/CTestクイックスタート

```cmake
# CMakeLists.txt (excerpt)
cmake_minimum_required(VERSION 3.20)
project(example LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)
# Prefer project-locked versions. If using a tag, use a pinned version per project policy.
set(GTEST_VERSION v1.17.0) # Adjust to project policy.
FetchContent_Declare(
  googletest
  # Google Test framework (official repository)
  URL https://github.com/google/googletest/archive/refs/tags/${GTEST_VERSION}.zip
)
FetchContent_MakeAvailable(googletest)

add_executable(example_tests
  tests/calculator_test.cpp
  src/calculator.cpp
)
target_link_libraries(example_tests GTest::gtest GTest::gmock GTest::gtest_main)

enable_testing()
include(GoogleTest)
gtest_discover_tests(example_tests)
```

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j
ctest --test-dir build --output-on-failure
```

## テストの実行

```bash
ctest --test-dir build --output-on-failure
ctest --test-dir build -R ClampTest
ctest --test-dir build -R "UserStoreTest.*" --output-on-failure
```

```bash
./build/example_tests --gtest_filter=ClampTest.*
./build/example_tests --gtest_filter=UserStoreTest.FindsExistingUser
```

## 失敗のデバッグ

1. gtestフィルターで単一の失敗テストを再実行する。
2. 失敗したアサーション周辺にスコープ付きログを追加する。
3. サニタイザーを有効にして再実行する。
4. 根本原因が修正されたらフルスイートに拡張する。

## カバレッジ

グローバルフラグの代わりにターゲットレベルの設定を優先する。

```cmake
option(ENABLE_COVERAGE "Enable coverage flags" OFF)

if(ENABLE_COVERAGE)
  if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    target_compile_options(example_tests PRIVATE --coverage)
    target_link_options(example_tests PRIVATE --coverage)
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    target_compile_options(example_tests PRIVATE -fprofile-instr-generate -fcoverage-mapping)
    target_link_options(example_tests PRIVATE -fprofile-instr-generate)
  endif()
endif()
```

GCC + gcov + lcov:

```bash
cmake -S . -B build-cov -DENABLE_COVERAGE=ON
cmake --build build-cov -j
ctest --test-dir build-cov
lcov --capture --directory build-cov --output-file coverage.info
lcov --remove coverage.info '/usr/*' --output-file coverage.info
genhtml coverage.info --output-directory coverage
```

Clang + llvm-cov:

```bash
cmake -S . -B build-llvm -DENABLE_COVERAGE=ON -DCMAKE_CXX_COMPILER=clang++
cmake --build build-llvm -j
LLVM_PROFILE_FILE="build-llvm/default.profraw" ctest --test-dir build-llvm
llvm-profdata merge -sparse build-llvm/default.profraw -o build-llvm/default.profdata
llvm-cov report build-llvm/example_tests -instr-profile=build-llvm/default.profdata
```

## サニタイザー

```cmake
option(ENABLE_ASAN "Enable AddressSanitizer" OFF)
option(ENABLE_UBSAN "Enable UndefinedBehaviorSanitizer" OFF)
option(ENABLE_TSAN "Enable ThreadSanitizer" OFF)

if(ENABLE_ASAN)
  add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
  add_link_options(-fsanitize=address)
endif()
if(ENABLE_UBSAN)
  add_compile_options(-fsanitize=undefined -fno-omit-frame-pointer)
  add_link_options(-fsanitize=undefined)
endif()
if(ENABLE_TSAN)
  add_compile_options(-fsanitize=thread)
  add_link_options(-fsanitize=thread)
endif()
```

## フレーキーテストのガードレール

- 同期に`sleep`を使用しない; 条件変数やラッチを使用する。
- 一時ディレクトリをテストごとにユニークにし、常にクリーンアップする。
- ユニットテストではリアルタイム、ネットワーク、ファイルシステムの依存を避ける。
- ランダム化された入力には決定的なシードを使用する。

## ベストプラクティス

### DO

- テストを決定的かつ分離して保つ
- グローバルよりも依存性注入を優先する
- 前提条件には`ASSERT_*`、複数チェックには`EXPECT_*`を使用
- CTestラベルまたはディレクトリでユニットテストと統合テストを分離する
- メモリとレース検出のためにCIでサニタイザーを実行する

### DON'T

- ユニットテストでリアルタイムやネットワークに依存しない
- 条件変数が使える場面でsleepを同期として使用しない
- シンプルな値オブジェクトを過度にモックしない
- 重要でないログに脆いストリングマッチングを使用しない

### よくある落とし穴

- **固定の一時パスの使用** → テストごとにユニークな一時ディレクトリを生成しクリーンアップする。
- **ウォールクロック時間への依存** → クロックを注入するかフェイクタイムソースを使用する。
- **フレーキーな並行テスト** → 条件変数/ラッチと上限付き待機を使用する。
- **隠れたグローバルステート** → フィクスチャでグローバルステートをリセットするか、グローバルを排除する。
- **過度なモック** → ステートフルな振る舞いにはフェイクを優先し、インタラクションのみモックする。
- **サニタイザー実行の欠落** → CIにASan/UBSan/TSanビルドを追加する。
- **デバッグ専用ビルドのカバレッジ** → カバレッジターゲットが一貫したフラグを使用していることを確認する。

## オプション付録: ファジング / プロパティテスト

プロジェクトが既にLLVM/libFuzzerまたはプロパティテストライブラリをサポートしている場合のみ使用する。

- **libFuzzer**: 最小限のI/Oを持つ純粋関数に最適。
- **RapidCheck**: 不変条件を検証するプロパティベーステスト。

最小限のlibFuzzerハーネス（疑似コード: ParseConfigを置き換える）:

```cpp
#include <cstddef>
#include <cstdint>
#include <string>

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    std::string input(reinterpret_cast<const char *>(data), size);
    // ParseConfig(input); // project function
    return 0;
}
```

## GoogleTestの代替

- **Catch2**: ヘッダーオンリー、表現力豊かなマッチャー
- **doctest**: 軽量、最小限のコンパイルオーバーヘッド
