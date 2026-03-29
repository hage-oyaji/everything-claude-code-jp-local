---
description: RustのTDDワークフローを強制します。テストを先に書き、その後実装します。cargo-llvm-covで80%以上のカバレッジを検証します。
---

# Rust TDDコマンド

このコマンドは、`#[test]`、rstest、proptest、mockallを使用したRustコードのテスト駆動開発手法を強制します。

## このコマンドの動作

1. **型/トレイトの定義**: `todo!()`で関数シグネチャをスキャフォールド
2. **テストの作成**: 包括的なテストモジュールを作成（RED）
3. **テストの実行**: テストが正しい理由で失敗することを確認
4. **コードの実装**: テストを通過する最小限のコードを記述（GREEN）
5. **リファクタリング**: テストをグリーンに保ちながら改善
6. **カバレッジの確認**: cargo-llvm-covで80%以上のカバレッジを確保

## 使用するタイミング

`/rust-test`は以下の場合に使用してください：
- 新しいRustの関数、メソッド、トレイトの実装
- 既存のRustコードへのテストカバレッジの追加
- バグ修正（最初に失敗するテストを記述）
- 重要なビジネスロジックの構築
- RustでのTDDワークフローの学習

## TDDサイクル

```
RED     -> Write failing test first
GREEN   -> Implement minimal code to pass
REFACTOR -> Improve code, tests stay green
REPEAT  -> Next test case
```

## セッション例

````text
User: /rust-test I need a function to validate user registration

Agent:
# TDD Session: Registration Validator

## Step 1: Define Interface

```rust
// src/validator.rs
pub struct RegistrationRequest {
    pub name: String,
    pub email: String,
    pub password: String,
}

pub enum ValidationResult {
    Valid,
    Invalid(Vec<String>),
}

pub fn validate_registration(request: &RegistrationRequest) -> ValidationResult {
    todo!()
}
```

## Step 2: Write Tests (RED)

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_registration_returns_valid() {
        let request = RegistrationRequest {
            name: "Alice".into(),
            email: "alice@example.com".into(),
            password: "SecureP@ss1".into(),
        };
        assert!(matches!(validate_registration(&request), ValidationResult::Valid));
    }

    #[test]
    fn blank_name_returns_invalid() {
        let request = RegistrationRequest {
            name: "".into(),
            email: "alice@example.com".into(),
            password: "SecureP@ss1".into(),
        };
        match validate_registration(&request) {
            ValidationResult::Invalid(errors) => {
                assert!(errors.contains(&"Name is required".to_string()));
            }
            ValidationResult::Valid => panic!("expected Invalid"),
        }
    }

    #[test]
    fn invalid_email_returns_invalid() {
        let request = RegistrationRequest {
            name: "Alice".into(),
            email: "not-an-email".into(),
            password: "SecureP@ss1".into(),
        };
        assert!(matches!(
            validate_registration(&request),
            ValidationResult::Invalid(_)
        ));
    }

    #[test]
    fn short_password_returns_invalid() {
        let request = RegistrationRequest {
            name: "Alice".into(),
            email: "alice@example.com".into(),
            password: "short".into(),
        };
        assert!(matches!(
            validate_registration(&request),
            ValidationResult::Invalid(_)
        ));
    }
}
```

## Step 3: Run Tests — Verify FAIL

```bash
$ cargo test
running 4 tests
test tests::valid_registration_returns_valid ... FAILED (not yet implemented)
test tests::blank_name_returns_invalid ... FAILED
test tests::invalid_email_returns_invalid ... FAILED
test tests::short_password_returns_invalid ... FAILED
```

Tests fail as expected (todo! panic).

## Step 4: Implement Minimal Code (GREEN)

```rust
pub fn validate_registration(request: &RegistrationRequest) -> ValidationResult {
    let mut errors = Vec::new();

    if request.name.trim().is_empty() {
        errors.push("Name is required".into());
    }
    if !request.email.contains('@') {
        errors.push("Invalid email format".into());
    }
    if request.password.len() < 8 {
        errors.push("Password must be at least 8 characters".into());
    }

    if errors.is_empty() {
        ValidationResult::Valid
    } else {
        ValidationResult::Invalid(errors)
    }
}
```

## Step 5: Run Tests — Verify PASS

```bash
$ cargo test
running 4 tests
test tests::valid_registration_returns_valid ... ok
test tests::blank_name_returns_invalid ... ok
test tests::invalid_email_returns_invalid ... ok
test tests::short_password_returns_invalid ... ok

test result: ok. 4 passed; 0 failed
```

All tests passing!

## Step 6: Check Coverage

```bash
$ cargo llvm-cov
Coverage: 100.0% of lines
```

Coverage: 100%

## TDD Complete!
````

## テストパターン

### ユニットテスト

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_two_numbers() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn handles_error() -> Result<(), Box<dyn std::error::Error>> {
        let result = parse_config(r#"port = 8080"#)?;
        assert_eq!(result.port, 8080);
        Ok(())
    }
}
```

### rstestによるパラメータ化テスト

```rust
use rstest::{rstest, fixture};

#[rstest]
#[case("hello", 5)]
#[case("", 0)]
#[case("rust", 4)]
fn test_string_length(#[case] input: &str, #[case] expected: usize) {
    assert_eq!(input.len(), expected);
}
```

### 非同期テスト

```rust
#[tokio::test]
async fn fetches_data_successfully() {
    let client = TestClient::new().await;
    let result = client.get("/data").await;
    assert!(result.is_ok());
}
```

### プロパティベーステスト

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn encode_decode_roundtrip(input in ".*") {
        let encoded = encode(&input);
        let decoded = decode(&encoded).unwrap();
        assert_eq!(input, decoded);
    }
}
```

## カバレッジコマンド

```bash
# Summary report
cargo llvm-cov

# HTML report
cargo llvm-cov --html

# Fail if below threshold
cargo llvm-cov --fail-under-lines 80

# Run specific test
cargo test test_name

# Run with output
cargo test -- --nocapture

# Run without stopping on first failure
cargo test --no-fail-fast
```

## カバレッジ目標

| コードタイプ | 目標 |
|-----------|--------|
| 重要なビジネスロジック | 100% |
| パブリックAPI | 90%以上 |
| 一般的なコード | 80%以上 |
| 生成コード / FFIバインディング | 除外 |

## TDDのベストプラクティス

**推奨：**
- 実装の前にまずテストを書く
- 変更ごとにテストを実行する
- より良いエラーメッセージのために`assert!`より`assert_eq!`を使用
- `Result`を返すテストでは`?`を使用してクリーンな出力を得る
- 実装ではなく振る舞いをテストする
- エッジケースを含める（空、境界値、エラーパス）

**非推奨：**
- テスト前に実装を書く
- REDフェーズをスキップする
- `Result::is_err()`で動作する場合に`#[should_panic]`を使用する
- テストで`sleep()`を使用する — チャネルまたは`tokio::time::pause()`を使用
- すべてをモックする — 実行可能な場合は統合テストを優先

## 関連コマンド

- `/rust-build` - ビルドエラーの修正
- `/rust-review` - 実装後のコードレビュー
- `/verify` - 完全な検証ループの実行

## 関連

- スキル：`skills/rust-testing/`
- スキル：`skills/rust-patterns/`
