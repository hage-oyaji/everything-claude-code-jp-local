---
paths:
  - "**/*.rs"
---
# Rust コーディングスタイル

> このファイルは [common/coding-style.md](../common/coding-style.md) を Rust 固有の内容で拡張します。

## フォーマット

- **rustfmt** で適用 — コミット前に必ず `cargo fmt` を実行
- **clippy** でリント — `cargo clippy -- -D warnings`（警告をエラーとして扱う）
- 4 スペースインデント（rustfmt デフォルト）
- 最大行幅: 100 文字（rustfmt デフォルト）

## イミュータビリティ

Rust の変数はデフォルトでイミュータブル — これを活かす：

- デフォルトで `let` を使用。ミューテーションが必要な場合のみ `let mut` を使用
- インプレース変更よりも新しい値を返すことを優先
- 関数がアロケーションする場合としない場合に `Cow<'_, T>` を使用

```rust
use std::borrow::Cow;

// GOOD — immutable by default, new value returned
fn normalize(input: &str) -> Cow<'_, str> {
    if input.contains(' ') {
        Cow::Owned(input.replace(' ', "_"))
    } else {
        Cow::Borrowed(input)
    }
}

// BAD — unnecessary mutation
fn normalize_bad(input: &mut String) {
    *input = input.replace(' ', "_");
}
```

## 命名規則

標準的な Rust の規約に従う：
- 関数、メソッド、変数、モジュール、クレートには `snake_case`
- 型、トレイト、列挙型、型パラメータには `PascalCase`（UpperCamelCase）
- 定数とスタティクスには `SCREAMING_SNAKE_CASE`
- ライフタイム: 短い小文字（`'a`、`'de`）— 複雑なケースには説明的な名前（`'input`）

## 所有権と借用

- デフォルトで借用（`&T`）。保存または消費する必要がある場合のみ所有権を取得
- 根本原因を理解せずに借用チェッカーを満たすためだけにクローンしない
- 関数パラメータでは `String` よりも `&str`、`Vec<T>` よりも `&[T]` を受け取る
- `String` の所有が必要なコンストラクタには `impl Into<String>` を使用

```rust
// GOOD — borrows when ownership isn't needed
fn word_count(text: &str) -> usize {
    text.split_whitespace().count()
}

// GOOD — takes ownership in constructor via Into
fn new(name: impl Into<String>) -> Self {
    Self { name: name.into() }
}

// BAD — takes String when &str suffices
fn word_count_bad(text: String) -> usize {
    text.split_whitespace().count()
}
```

## エラーハンドリング

- `Result<T, E>` と `?` を伝播に使用 — プロダクションコードで `unwrap()` は使わない
- **ライブラリ**: `thiserror` で型付きエラーを定義
- **アプリケーション**: 柔軟なエラーコンテキストに `anyhow` を使用
- `.with_context(|| format!("failed to ..."))?` でコンテキストを追加
- `unwrap()` / `expect()` はテストと本当に到達不能な状態に限定

```rust
// GOOD — library error with thiserror
#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    #[error("failed to read config: {0}")]
    Io(#[from] std::io::Error),
    #[error("invalid config format: {0}")]
    Parse(String),
}

// GOOD — application error with anyhow
use anyhow::Context;

fn load_config(path: &str) -> anyhow::Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read {path}"))?;
    toml::from_str(&content)
        .with_context(|| format!("failed to parse {path}"))
}
```

## ループよりもイテレータ

変換にはイテレータチェーンを優先。複雑な制御フローにはループを使用：

```rust
// GOOD — declarative and composable
let active_emails: Vec<&str> = users.iter()
    .filter(|u| u.is_active)
    .map(|u| u.email.as_str())
    .collect();

// GOOD — loop for complex logic with early returns
for user in &users {
    if let Some(verified) = verify_email(&user.email)? {
        send_welcome(&verified)?;
    }
}
```

## モジュール構成

型ではなくドメインで整理：

```text
src/
├── main.rs
├── lib.rs
├── auth/           # Domain module
│   ├── mod.rs
│   ├── token.rs
│   └── middleware.rs
├── orders/         # Domain module
│   ├── mod.rs
│   ├── model.rs
│   └── service.rs
└── db/             # Infrastructure
    ├── mod.rs
    └── pool.rs
```

## 可視性

- デフォルトはプライベート。内部共有には `pub(crate)` を使用
- クレートのパブリック API の一部のみ `pub` とする
- パブリック API は `lib.rs` から再エクスポート

## 参考

スキル `rust-patterns` に Rust のイディオムとパターンの詳細があります。
