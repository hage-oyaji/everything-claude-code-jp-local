---
description: Rustのビルドエラー、借用チェッカーの問題、依存関係の問題をインクリメンタルに修正します。最小限の外科的修正のためにrust-build-resolverエージェントを呼び出します。
---

# Rustビルド＆修正

このコマンドは、最小限の変更でRustのビルドエラーをインクリメンタルに修正するために**rust-build-resolver**エージェントを呼び出します。

## このコマンドの動作

1. **診断の実行**: `cargo check`、`cargo clippy`、`cargo fmt --check`を実行
2. **エラーの解析**: エラーコードと影響を受けるファイルを特定
3. **インクリメンタルに修正**: 一度に1つのエラー
4. **各修正の検証**: 変更ごとに`cargo check`を再実行
5. **サマリーの報告**: 修正された内容と残りの問題を表示

## 使用するタイミング

`/rust-build`は以下の場合に使用してください：
- `cargo build`または`cargo check`がエラーで失敗する
- `cargo clippy`が警告を報告する
- 借用チェッカーやライフタイムのエラーがコンパイルをブロックする
- Cargoの依存関係解決が失敗する
- ビルドを壊す変更をプルした後

## 実行される診断コマンド

```bash
# Primary build check
cargo check 2>&1

# Lints and suggestions
cargo clippy -- -D warnings 2>&1

# Formatting check
cargo fmt --check 2>&1

# Dependency issues
cargo tree --duplicates

# Security audit (if available)
if command -v cargo-audit >/dev/null; then cargo audit; else echo "cargo-audit not installed"; fi
```

## セッション例

````text
User: /rust-build

Agent:
# Rust Build Resolution

## Initial Diagnostics

```bash
$ cargo check
error[E0502]: cannot borrow `map` as mutable because it is also borrowed as immutable
  --> src/service/cache.rs:25:5
error[E0308]: mismatched types
  --> src/handler/api.rs:42:12
error[E0425]: cannot find value `db` in this scope
  --> src/handler/api.rs:58:9
```

Errors found: 3

## Fix 1: Borrow Checker Error

File: src/service/cache.rs:25
Error: E0502 — cannot borrow `map` as mutable because also borrowed as immutable
Cause: Immutable borrow still active during mutable insert

```rust
// Changed: restructured to end immutable borrow before mutable access
if !map.contains_key("key") {
    map.insert("key".into(), default);
}
```

```bash
$ cargo check
# 2 errors remaining
```

## Fix 2: Type Mismatch

File: src/handler/api.rs:42
Error: E0308 — mismatched types: expected `usize`, found `&str`

```rust
// Changed
let count = params.get("count");
// To
let count: usize = params.get("count")
    .and_then(|s| s.parse().ok())
    .unwrap_or(0);
```

```bash
$ cargo check
# 1 error remaining
```

## Fix 3: Unresolved Name

File: src/handler/api.rs:58
Error: E0425 — cannot find value `db` in this scope
Cause: Missing import

```rust
// Added
use crate::db;
```

```bash
$ cargo check
# Build successful!
```

## Final Verification

```bash
$ cargo clippy -- -D warnings
# No warnings

$ cargo test
running 6 tests
test service::cache::tests::test_insert ... ok
test service::cache::tests::test_get ... ok
test handler::api::tests::test_parse_count ... ok
test handler::api::tests::test_missing_count ... ok
test handler::api::tests::test_db_import ... ok
test handler::api::tests::test_response ... ok

test result: ok. 6 passed; 0 failed; 0 ignored
```

## Summary

| Metric | Count |
|--------|-------|
| Build errors fixed | 3 |
| Clippy warnings fixed | 0 |
| Files modified | 2 |
| Remaining issues | 0 |

Build Status: SUCCESS
````

## よくあるエラーの修正

| エラー | 一般的な修正 |
|-------|-------------|
| `cannot borrow as mutable` | 不変借用を先に終了するように構造変更。正当な場合のみclone |
| `does not live long enough` | 所有型を使用するかライフタイム注釈を追加 |
| `cannot move out of` | 所有権を取得するように構造変更。cloneは最終手段 |
| `mismatched types` | `.into()`、`as`、または明示的な変換を追加 |
| `trait X not implemented` | `#[derive(Trait)]`を追加または手動で実装 |
| `unresolved import` | Cargo.tomlに追加または`use`パスを修正 |
| `cannot find value` | importを追加またはパスを修正 |

## 修正戦略

1. **ビルドエラーを最優先** - コードがコンパイルできること
2. **次にClippy警告** - 疑わしい構成を修正
3. **フォーマットは最後** - `cargo fmt`準拠
4. **一度に1つの修正** - 各変更を検証
5. **最小限の変更** - リファクタリングせず、修正のみ

## 停止条件

エージェントは以下の場合に停止して報告します：
- 3回の試行後も同じエラーが続く
- 修正がさらに多くのエラーを生む
- アーキテクチャの変更が必要
- 借用チェッカーのエラーがデータ所有権の再設計を必要とする

## 関連コマンド

- `/rust-test` - ビルド成功後にテストを実行
- `/rust-review` - コード品質のレビュー
- `/verify` - 完全な検証ループ

## 関連

- エージェント：`agents/rust-build-resolver.md`
- スキル：`skills/rust-patterns/`
