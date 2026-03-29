---
description: 所有権、ライフタイム、エラーハンドリング、unsafe使用、イディオマティックなパターンに関する包括的なRustコードレビュー。rust-reviewerエージェントを呼び出します。
---

# Rustコードレビュー

このコマンドは、Rust固有の包括的なコードレビューのために**rust-reviewer**エージェントを呼び出します。

## このコマンドの動作

1. **自動チェックの検証**: `cargo check`、`cargo clippy -- -D warnings`、`cargo fmt --check`、`cargo test`を実行 — いずれかが失敗したら停止
2. **Rust変更の特定**: `git diff HEAD~1`（PRの場合は`git diff main...HEAD`）で変更された`.rs`ファイルを検出
3. **セキュリティ監査の実行**: 利用可能な場合`cargo audit`を実行
4. **セキュリティスキャン**: unsafe使用、コマンドインジェクション、ハードコードされた秘密情報をチェック
5. **所有権レビュー**: 不要なclone、ライフタイムの問題、借用パターンを分析
6. **レポート生成**: 重要度別に問題を分類

## 使用するタイミング

`/rust-review`は以下の場合に使用してください：
- Rustコードを作成または変更した後
- Rustの変更をコミットする前
- Rustコードを含むプルリクエストのレビュー時
- 新しいRustコードベースへのオンボーディング時
- イディオマティックなRustパターンの学習時

## レビューカテゴリ

### CRITICAL（修正必須）
- 本番コードパスでの未チェックの`unwrap()`/`expect()`
- 不変条件を文書化する`// SAFETY:`コメントなしの`unsafe`
- クエリ内の文字列補間によるSQLインジェクション
- `std::process::Command`での未検証入力によるコマンドインジェクション
- ハードコードされた認証情報
- 生ポインタによるuse-after-free

### HIGH（修正推奨）
- 借用チェッカーを満たすための不要な`.clone()`
- `&str`または`impl AsRef<str>`で十分な場合の`String`パラメータ
- 非同期コンテキストでのブロッキング（`std::thread::sleep`、`std::fs`）
- 共有型での`Send`/`Sync`バウンドの欠落
- ビジネスクリティカルなenumでのワイルドカード`_ =>`マッチ
- 大きな関数（50行超）

### MEDIUM（検討事項）
- ホットパスでの不要な割り当て
- サイズが分かっている場合の`with_capacity`の欠落
- 正当な理由なしのclippy警告抑制
- `///`ドキュメントなしのパブリックAPI
- 値を無視することがバグの可能性がある非`must_use`戻り値型に`#[must_use]`を検討

## 自動チェックの実行

```bash
# Build gate (must pass before review)
cargo check

# Lints and suggestions
cargo clippy -- -D warnings

# Formatting
cargo fmt --check

# Tests
cargo test

# Security audit (if available)
if command -v cargo-audit >/dev/null; then cargo audit; else echo "cargo-audit not installed"; fi
```

## 使用例

````text
User: /rust-review

Agent:
# Rust Code Review Report

## Files Reviewed
- src/service/user.rs (modified)
- src/handler/api.rs (modified)

## Static Analysis Results
- Build: Successful
- Clippy: No warnings
- Formatting: Passed
- Tests: All passing

## Issues Found

[CRITICAL] Unchecked unwrap in Production Path
File: src/service/user.rs:28
Issue: Using `.unwrap()` on database query result
```rust
let user = db.find_by_id(id).unwrap();  // Panics on missing user
```
Fix: Propagate error with context
```rust
let user = db.find_by_id(id)
    .context("failed to fetch user")?;
```

[HIGH] Unnecessary Clone
File: src/handler/api.rs:45
Issue: Cloning String to satisfy borrow checker
```rust
let name = user.name.clone();
process(&user, &name);
```
Fix: Restructure to avoid clone
```rust
let result = process_name(&user.name);
use_user(&user, result);
```

## Summary
- CRITICAL: 1
- HIGH: 1
- MEDIUM: 0

Recommendation: Block merge until CRITICAL issue is fixed
````

## 承認基準

| ステータス | 条件 |
|--------|-----------|
| 承認 | CRITICALまたはHIGHの問題がない |
| 警告 | MEDIUMの問題のみ（注意してマージ） |
| ブロック | CRITICALまたはHIGHの問題が検出された |

## 他のコマンドとの連携

- まず`/rust-test`を使用してテストが通ることを確認
- ビルドエラーが発生した場合は`/rust-build`を使用
- コミット前に`/rust-review`を使用
- Rust固有でない問題には`/code-review`を使用

## 関連

- エージェント：`agents/rust-reviewer.md`
- スキル：`skills/rust-patterns/`、`skills/rust-testing/`
