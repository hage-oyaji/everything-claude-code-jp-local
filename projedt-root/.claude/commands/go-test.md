---
description: GoのTDDワークフローを実施する。先にテーブル駆動テストを書き、その後実装する。go test -coverで80%以上のカバレッジを検証する。
---

# Go TDD コマンド

このコマンドは慣用的なGoテストパターンを使用した、Goコードのテスト駆動開発手法を実施します。

## このコマンドの動作

1. **型/インターフェースを定義**: まず関数のシグネチャをスキャフォールドする
2. **テーブル駆動テストを書く**: 包括的なテストケースを作成する（RED）
3. **テストを実行**: テストが正しい理由で失敗することを確認する
4. **コードを実装**: テストを通過する最小限のコードを書く（GREEN）
5. **リファクタリング**: テストをグリーンに保ちながら改善する
6. **カバレッジを確認**: 80%以上のカバレッジを確保する

## 使用場面

`/go-test` を使用する場面:
- 新しいGo関数を実装する時
- 既存コードにテストカバレッジを追加する時
- バグ修正時（先にテストを失敗させてから修正）
- 重要なビジネスロジックを構築する時
- GoでのTDDワークフローを学習する時

## TDDサイクル

```
RED     → 失敗するテーブル駆動テストを書く
GREEN   → テストを通過する最小限のコードを実装
REFACTOR → コードを改善、テストはグリーンを維持
REPEAT  → 次のテストケースへ
```

## セッション例

````
User: /go-test I need a function to validate email addresses

Agent:
# TDDセッション: メールバリデーター

## ステップ1: インターフェースの定義

```go
// validator/email.go
package validator

// ValidateEmail checks if the given string is a valid email address.
// Returns nil if valid, error describing the issue otherwise.
func ValidateEmail(email string) error {
    panic("not implemented")
}
```

## ステップ2: テーブル駆動テストを書く（RED）

```go
// validator/email_test.go
package validator

import (
    "testing"
)

func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantErr bool
    }{
        // 有効なメール
        {"simple email", "user@example.com", false},
        {"with subdomain", "user@mail.example.com", false},
        {"with plus", "user+tag@example.com", false},
        {"with dots", "first.last@example.com", false},

        // 無効なメール
        {"empty string", "", true},
        {"no at sign", "userexample.com", true},
        {"no domain", "user@", true},
        {"no local part", "@example.com", true},
        {"double at", "user@@example.com", true},
        {"spaces", "user @example.com", true},
        {"no tld", "user@example", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateEmail(tt.email)
            if tt.wantErr && err == nil {
                t.Errorf("ValidateEmail(%q) = nil; want error", tt.email)
            }
            if !tt.wantErr && err != nil {
                t.Errorf("ValidateEmail(%q) = %v; want nil", tt.email, err)
            }
        })
    }
}
```

## ステップ3: テスト実行 - 失敗を確認

```bash
$ go test ./validator/...

--- FAIL: TestValidateEmail (0.00s)
    --- FAIL: TestValidateEmail/simple_email (0.00s)
        panic: not implemented

FAIL
```

✓ テストが期待通りに失敗（panic）。

## ステップ4: 最小限のコードを実装（GREEN）

```go
// validator/email.go
package validator

import (
    "errors"
    "regexp"
)

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

var (
    ErrEmailEmpty   = errors.New("email cannot be empty")
    ErrEmailInvalid = errors.New("email format is invalid")
)

func ValidateEmail(email string) error {
    if email == "" {
        return ErrEmailEmpty
    }
    if !emailRegex.MatchString(email) {
        return ErrEmailInvalid
    }
    return nil
}
```

## ステップ5: テスト実行 - 合格を確認

```bash
$ go test ./validator/...

PASS
ok      project/validator    0.003s
```

✓ すべてのテスト合格！

## ステップ6: カバレッジ確認

```bash
$ go test -cover ./validator/...

PASS
coverage: 100.0% of statements
ok      project/validator    0.003s
```

✓ カバレッジ: 100%

## TDD完了！
````

## テストパターン

### テーブル駆動テスト
```go
tests := []struct {
    name     string
    input    InputType
    want     OutputType
    wantErr  bool
}{
    {"case 1", input1, want1, false},
    {"case 2", input2, want2, true},
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := Function(tt.input)
        // アサーション
    })
}
```

### パラレルテスト
```go
for _, tt := range tests {
    tt := tt // キャプチャ
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // テスト本体
    })
}
```

### テストヘルパー
```go
func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()
    db := createDB()
    t.Cleanup(func() { db.Close() })
    return db
}
```

## カバレッジコマンド

```bash
# 基本カバレッジ
go test -cover ./...

# カバレッジプロファイル
go test -coverprofile=coverage.out ./...

# ブラウザで表示
go tool cover -html=coverage.out

# 関数ごとのカバレッジ
go tool cover -func=coverage.out

# レース検出付き
go test -race -cover ./...
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
- 包括的なカバレッジのためにテーブル駆動テストを使用する
- 実装の詳細ではなく、振る舞いをテストする
- エッジケースを含める（空、nil、最大値）

**非推奨:**
- テストの前に実装を書くこと
- REDフェーズをスキップすること
- プライベート関数を直接テストすること
- テストで `time.Sleep` を使用すること
- フレーキーテストを無視すること

## 関連コマンド

- `/go-build` - ビルドエラーの修正
- `/go-review` - 実装後のコードレビュー
- `/verify` - 完全な検証ループの実行

## 関連

- スキル: `skills/golang-testing/`
- スキル: `skills/tdd-workflow/`
