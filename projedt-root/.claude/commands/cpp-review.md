---
description: メモリ安全性、モダンC++のイディオム、並行性、セキュリティに関する包括的なC++コードレビュー。cpp-reviewerエージェントを呼び出す。
---

# C++ コードレビュー

このコマンドは**cpp-reviewer**エージェントを呼び出し、C++固有の包括的なコードレビューを実施します。

## このコマンドの動作

1. **C++の変更を特定**: `git diff` で変更された `.cpp`、`.hpp`、`.cc`、`.h` ファイルを検出
2. **静的解析を実行**: `clang-tidy` と `cppcheck` を実行
3. **メモリ安全性スキャン**: 生のnew/delete、バッファオーバーフロー、use-after-freeを確認
4. **並行性レビュー**: スレッドセーフ性、mutex使用、データ競合を分析
5. **モダンC++チェック**: コードがC++17/20の規約とベストプラクティスに従っているか検証
6. **レポート生成**: 問題を重大度別に分類

## 使用場面

`/cpp-review` を使用する場面:
- C++コードを書いたり変更した後
- C++の変更をコミットする前
- C++コードを含むプルリクエストのレビュー時
- 新しいC++コードベースのオンボーディング時
- メモリ安全性の問題を確認する時

## レビューカテゴリ

### CRITICAL（必ず修正）
- RAIIなしの生の `new`/`delete`
- バッファオーバーフローとuse-after-free
- 同期なしのデータ競合
- `system()` を介したコマンドインジェクション
- 未初期化変数の読み取り
- ヌルポインタの参照外し

### HIGH（修正すべき）
- Rule of Five違反
- `std::lock_guard` / `std::scoped_lock` の欠如
- 適切なライフタイム管理なしのデタッチされたスレッド
- `static_cast`/`dynamic_cast` の代わりにCスタイルキャスト
- `const` 正確性の欠如

### MEDIUM（検討）
- 不必要なコピー（`const&` ではなく値渡し）
- サイズが既知のコンテナでの `reserve()` の欠如
- ヘッダーでの `using namespace std;`
- 重要な戻り値での `[[nodiscard]]` の欠如
- 過度に複雑なテンプレートメタプログラミング

## 実行される自動チェック

```bash
# 静的解析
clang-tidy --checks='*,-llvmlibc-*' src/*.cpp -- -std=c++17

# 追加解析
cppcheck --enable=all --suppress=missingIncludeSystem src/

# 警告付きビルド
cmake --build build -- -Wall -Wextra -Wpedantic
```

## 使用例

```text
User: /cpp-review

Agent:
# C++ コードレビューレポート

## レビュー対象ファイル
- src/handler/user.cpp (変更)
- src/service/auth.cpp (変更)

## 静的解析結果
✓ clang-tidy: 警告2件
✓ cppcheck: 問題なし

## 発見された問題

[CRITICAL] メモリリーク
ファイル: src/service/auth.cpp:45
問題: 対応する `delete` のない生の `new`
```cpp
auto* session = new Session(userId);  // メモリリーク！
cache[userId] = session;
```
修正: `std::unique_ptr` を使用
```cpp
auto session = std::make_unique<Session>(userId);
cache[userId] = std::move(session);
```

[HIGH] const参照の欠如
ファイル: src/handler/user.cpp:28
問題: 大きなオブジェクトが値渡しされている
```cpp
void processUser(User user) {  // 不必要なコピー
```
修正: const参照で渡す
```cpp
void processUser(const User& user) {
```

## サマリー
- CRITICAL: 1
- HIGH: 1
- MEDIUM: 0

推奨: ❌ CRITICALの問題が修正されるまでマージをブロック
```

## 承認基準

| ステータス | 条件 |
|--------|-----------|
| ✅ 承認 | CRITICALまたはHIGHの問題なし |
| ⚠️ 警告 | MEDIUMの問題のみ（注意してマージ） |
| ❌ ブロック | CRITICALまたはHIGHの問題あり |

## 他のコマンドとの連携

- まず `/cpp-test` を使用してテストが通ることを確認
- ビルドエラーが発生した場合は `/cpp-build` を使用
- コミット前に `/cpp-review` を使用
- C++以外の一般的な懸念には `/code-review` を使用

## 関連

- エージェント: `agents/cpp-reviewer.md`
- スキル: `skills/cpp-coding-standards/`、`skills/cpp-testing/`
