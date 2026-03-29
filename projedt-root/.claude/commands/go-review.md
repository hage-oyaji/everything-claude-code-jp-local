---
description: 慣用的なGoパターン、並行性の安全性、エラーハンドリング、セキュリティに関する包括的なGoコードレビュー。go-reviewerエージェントを呼び出す。
---

# Go コードレビュー

このコマンドは**go-reviewer**エージェントを呼び出し、Go固有の包括的なコードレビューを実施します。

## このコマンドの動作

1. **Goの変更を特定**: `git diff` で変更された `.go` ファイルを検出
2. **静的解析を実行**: `go vet`、`staticcheck`、`golangci-lint` を実行
3. **セキュリティスキャン**: SQLインジェクション、コマンドインジェクション、競合状態を確認
4. **並行性レビュー**: goroutineの安全性、チャネルの使用、mutexパターンを分析
5. **慣用的Goチェック**: コードがGoの規約とベストプラクティスに従っているか検証
6. **レポート生成**: 問題を重大度別に分類

## 使用場面

`/go-review` を使用する場面:
- Goコードを書いたり変更した後
- Goの変更をコミットする前
- Goコードを含むプルリクエストのレビュー時
- 新しいGoコードベースのオンボーディング時
- 慣用的なGoパターンを学習する時

## レビューカテゴリ

### CRITICAL（必ず修正）
- SQL/コマンドインジェクションの脆弱性
- 同期なしの競合状態
- goroutineリーク
- ハードコードされた認証情報
- 安全でないポインタの使用
- クリティカルパスでのエラー無視

### HIGH（修正すべき）
- コンテキスト付きのエラーラッピングの欠如
- エラーを返す代わりにpanic
- コンテキストの伝播漏れ
- デッドロックを引き起こすバッファなしチャネル
- インターフェース未実装エラー
- mutex保護の欠如

### MEDIUM（検討）
- 慣用的でないコードパターン
- エクスポートに対するgodocコメントの欠如
- 非効率な文字列連結
- スライスの事前割り当てなし
- テーブル駆動テストの未使用

## 実行される自動チェック

```bash
# 静的解析
go vet ./...

# 高度なチェック（インストール済みの場合）
staticcheck ./...
golangci-lint run

# レース検出
go build -race ./...

# セキュリティ脆弱性
govulncheck ./...
```

## 使用例

```text
User: /go-review

Agent:
# Go コードレビューレポート

## レビュー対象ファイル
- internal/handler/user.go (変更)
- internal/service/auth.go (変更)

## 静的解析結果
✓ go vet: 問題なし
✓ staticcheck: 問題なし

## 発見された問題

[CRITICAL] 競合状態
ファイル: internal/service/auth.go:45
問題: 共有マップが同期なしでアクセスされている
```go
var cache = map[string]*Session{}  // 並行アクセス！

func GetSession(id string) *Session {
    return cache[id]  // 競合状態
}
```
修正: sync.RWMutexまたはsync.Mapを使用
```go
var (
    cache   = map[string]*Session{}
    cacheMu sync.RWMutex
)

func GetSession(id string) *Session {
    cacheMu.RLock()
    defer cacheMu.RUnlock()
    return cache[id]
}
```

[HIGH] エラーコンテキストの欠如
ファイル: internal/handler/user.go:28
問題: コンテキストなしでエラーが返されている
```go
return err  // コンテキストなし
```
修正: コンテキスト付きでラップする
```go
return fmt.Errorf("get user %s: %w", userID, err)
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

- まず `/go-test` を使用してテストが通ることを確認
- ビルドエラーが発生した場合は `/go-build` を使用
- コミット前に `/go-review` を使用
- Go以外の一般的な懸念には `/code-review` を使用

## 関連

- エージェント: `agents/go-reviewer.md`
- スキル: `skills/golang-patterns/`、`skills/golang-testing/`
