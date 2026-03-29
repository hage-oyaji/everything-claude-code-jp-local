---
description: C++ビルドエラー、CMakeの問題、リンカーの問題を段階的に修正する。最小限の外科的修正のためにcpp-build-resolverエージェントを呼び出す。
---

# C++ ビルドと修正

このコマンドは**cpp-build-resolver**エージェントを呼び出し、最小限の変更でC++ビルドエラーを段階的に修正します。

## このコマンドの動作

1. **診断を実行**: `cmake --build`、`clang-tidy`、`cppcheck` を実行
2. **エラーを解析**: ファイルごとにグループ化し、重大度順にソート
3. **段階的に修正**: 一度に1つのエラーを修正
4. **各修正を検証**: 変更ごとにビルドを再実行
5. **サマリーを報告**: 修正した内容と残存する問題を表示

## 使用場面

`/cpp-build` を使用する場面:
- `cmake --build build` がエラーで失敗する場合
- リンカーエラー（未定義の参照、多重定義）
- テンプレートのインスタンス化失敗
- インクルード/依存関係の問題
- ビルドを壊す変更をプルした後

## 実行される診断コマンド

```bash
# CMake設定
cmake -B build -S .

# ビルド
cmake --build build 2>&1 | head -100

# 静的解析（利用可能な場合）
clang-tidy src/*.cpp -- -std=c++17
cppcheck --enable=all src/
```

## セッション例

```text
User: /cpp-build

Agent:
# C++ ビルド解決

## 初期診断

```bash
$ cmake --build build
src/service/user.cpp:25:15: error: use of undeclared identifier 'UserRepository'
src/handler/api.cpp:42:9: error: no matching function for call to 'process'
src/handler/api.cpp:58:1: error: non-void function does not return a value
```

エラー数: 3

## 修正1: 未宣言の識別子

ファイル: src/service/user.cpp:25
エラー: use of undeclared identifier 'UserRepository'
原因: インクルード不足

```cpp
// インクルードを追加
#include "repository/user_repository.hpp"
```

```bash
$ cmake --build build
# 残りエラー: 2
```

## 修正2: 一致する関数がない

ファイル: src/handler/api.cpp:42
エラー: no matching function for call to 'process'

```cpp
// 変更前
process(params.get("count"));
// 変更後
process(std::stoi(params.get("count")));
```

```bash
$ cmake --build build
# 残りエラー: 1
```

## 修正3: 戻り値の欠落

ファイル: src/handler/api.cpp:58
エラー: non-void function does not return a value

```cpp
std::optional<User> getUser(const std::string& id) {
    if (id.empty()) {
        return std::nullopt;
    }
    auto user = findUser(id);
    // 欠落していたreturnを追加
    return user;
}
```

```bash
$ cmake --build build
# ビルド成功！
```

## 最終検証

```bash
$ ctest --test-dir build --output-on-failure
Test project build
    1/5 Test #1: unit_tests ........   Passed    0.02 sec
    2/5 Test #2: integration_tests    Passed    0.15 sec
All tests passed.
```

## サマリー

| 指標 | 数 |
|--------|-------|
| 修正したビルドエラー | 3 |
| 修正したリンカーエラー | 0 |
| 変更したファイル | 2 |
| 残存する問題 | 0 |

ビルドステータス: ✅ 成功
```

## よくあるエラーの修正

| エラー | 典型的な修正 |
|-------|-------------|
| `undeclared identifier` | `#include` を追加またはタイポを修正 |
| `no matching function` | 引数の型を修正またはオーバーロードを追加 |
| `undefined reference` | ライブラリをリンクまたは実装を追加 |
| `multiple definition` | `inline` を使用または.cppに移動 |
| `incomplete type` | 前方宣言を `#include` に置き換え |
| `no member named X` | メンバー名を修正またはインクルード |
| `cannot convert X to Y` | 適切なキャストを追加 |
| `CMake Error` | CMakeLists.txtの設定を修正 |

## 修正戦略

1. **コンパイルエラーを最優先** - コードがコンパイルできること
2. **リンカーエラーを次に** - 未定義の参照を解決
3. **警告を3番目に** - `-Wall -Wextra` で修正
4. **一度に1つの修正** - 各変更を検証
5. **最小限の変更** - リファクタリングせず、修正のみ

## 停止条件

以下の場合、エージェントは停止して報告する:
- 同じエラーが3回試行しても解消しない
- 修正がより多くのエラーを引き起こす
- アーキテクチャの変更が必要
- 外部依存関係が不足

## 関連コマンド

- `/cpp-test` - ビルド成功後にテストを実行
- `/cpp-review` - コード品質のレビュー
- `/verify` - 完全な検証ループ

## 関連

- エージェント: `agents/cpp-build-resolver.md`
- スキル: `skills/cpp-coding-standards/`
