---
name: eval-harness
description: 評価駆動開発（EDD）原則を実装するClaude Codeセッションのための正式な評価フレームワーク
origin: ECC
tools: Read, Write, Edit, Bash, Grep, Glob
---

# 評価ハーネススキル

Claude Codeセッションのための正式な評価フレームワークで、評価駆動開発（EDD）原則を実装します。

## アクティベーション条件

- AI支援ワークフロー向けの評価駆動開発（EDD）のセットアップ
- Claude Codeタスク完了の合格/不合格基準の定義
- pass@kメトリクスによるエージェント信頼性の測定
- プロンプトまたはエージェント変更に対するリグレッションテストスイートの作成
- モデルバージョン間のエージェントパフォーマンスベンチマーク

## 哲学

評価駆動開発は評価を「AI開発のユニットテスト」として扱います：
- 実装前に期待される動作を定義
- 開発中に継続的に評価を実行
- 各変更でリグレッションを追跡
- 信頼性測定にpass@kメトリクスを使用

## 評価タイプ

### ケイパビリティ評価
Claudeが以前できなかったことができるかをテスト：
```markdown
[CAPABILITY EVAL: feature-name]
Task: Description of what Claude should accomplish
Success Criteria:
  - [ ] Criterion 1
  - [ ] Criterion 2
  - [ ] Criterion 3
Expected Output: Description of expected result
```

### リグレッション評価
変更が既存機能を壊さないことを確認：
```markdown
[REGRESSION EVAL: feature-name]
Baseline: SHA or checkpoint name
Tests:
  - existing-test-1: PASS/FAIL
  - existing-test-2: PASS/FAIL
  - existing-test-3: PASS/FAIL
Result: X/Y passed (previously Y/Y)
```

## グレーダータイプ

### 1. コードベースグレーダー
コードによる決定論的チェック：
```bash
# Check if file contains expected pattern
grep -q "export function handleAuth" src/auth.ts && echo "PASS" || echo "FAIL"

# Check if tests pass
npm test -- --testPathPattern="auth" && echo "PASS" || echo "FAIL"

# Check if build succeeds
npm run build && echo "PASS" || echo "FAIL"
```

### 2. モデルベースグレーダー
Claudeを使用してオープンエンドな出力を評価：
```markdown
[MODEL GRADER PROMPT]
Evaluate the following code change:
1. Does it solve the stated problem?
2. Is it well-structured?
3. Are edge cases handled?
4. Is error handling appropriate?

Score: 1-5 (1=poor, 5=excellent)
Reasoning: [explanation]
```

### 3. ヒューマングレーダー
手動レビュー用にフラグ：
```markdown
[HUMAN REVIEW REQUIRED]
Change: Description of what changed
Reason: Why human review is needed
Risk Level: LOW/MEDIUM/HIGH
```

## メトリクス

### pass@k
「k回の試行で少なくとも1回成功」
- pass@1：初回試行の成功率
- pass@3：3回以内の成功率
- 一般的な目標：pass@3 > 90%

### pass^k
「k回すべての試行が成功」
- 信頼性のより高い基準
- pass^3：3回連続で成功
- クリティカルパスに使用

## 評価ワークフロー

### 1. 定義（コーディング前）
```markdown
## EVAL DEFINITION: feature-xyz

### Capability Evals
1. Can create new user account
2. Can validate email format
3. Can hash password securely

### Regression Evals
1. Existing login still works
2. Session management unchanged
3. Logout flow intact

### Success Metrics
- pass@3 > 90% for capability evals
- pass^3 = 100% for regression evals
```

### 2. 実装
定義した評価をパスするコードを作成。

### 3. 評価
```bash
# Run capability evals
[Run each capability eval, record PASS/FAIL]

# Run regression evals
npm test -- --testPathPattern="existing"

# Generate report
```

### 4. レポート
```markdown
EVAL REPORT: feature-xyz
========================

Capability Evals:
  create-user:     PASS (pass@1)
  validate-email:  PASS (pass@2)
  hash-password:   PASS (pass@1)
  Overall:         3/3 passed

Regression Evals:
  login-flow:      PASS
  session-mgmt:    PASS
  logout-flow:     PASS
  Overall:         3/3 passed

Metrics:
  pass@1: 67% (2/3)
  pass@3: 100% (3/3)

Status: READY FOR REVIEW
```

## 統合パターン

### 実装前
```
/eval define feature-name
```
`.claude/evals/feature-name.md`に評価定義ファイルを作成

### 実装中
```
/eval check feature-name
```
現在の評価を実行しステータスを報告

### 実装後
```
/eval report feature-name
```
完全な評価レポートを生成

## 評価の保存

プロジェクトに評価を保存：
```
.claude/
  evals/
    feature-xyz.md      # Eval definition
    feature-xyz.log     # Eval run history
    baseline.json       # Regression baselines
```

## ベストプラクティス

1. **コーディング前に評価を定義** — 成功基準の明確な思考を強制
2. **評価を頻繁に実行** — リグレッションを早期に検出
3. **pass@kを経時的に追跡** — 信頼性トレンドを監視
4. **可能な限りコードグレーダーを使用** — 決定論的 > 確率的
5. **セキュリティにはヒューマンレビュー** — セキュリティチェックを完全に自動化しない
6. **評価を高速に保つ** — 遅い評価は実行されない
7. **評価をコードとバージョン管理** — 評価はファーストクラスのアーティファクト

## 例：認証の追加

```markdown
## EVAL: add-authentication

### Phase 1: Define (10 min)
Capability Evals:
- [ ] User can register with email/password
- [ ] User can login with valid credentials
- [ ] Invalid credentials rejected with proper error
- [ ] Sessions persist across page reloads
- [ ] Logout clears session

Regression Evals:
- [ ] Public routes still accessible
- [ ] API responses unchanged
- [ ] Database schema compatible

### Phase 2: Implement (varies)
[Write code]

### Phase 3: Evaluate
Run: /eval check add-authentication

### Phase 4: Report
EVAL REPORT: add-authentication
==============================
Capability: 5/5 passed (pass@3: 100%)
Regression: 3/3 passed (pass^3: 100%)
Status: SHIP IT
```

## プロダクト評価 (v1.8)

ユニットテストだけでは動作品質をキャプチャできない場合にプロダクト評価を使用します。

### グレーダータイプ

1. コードグレーダー（決定論的アサーション）
2. ルールグレーダー（正規表現/スキーマ制約）
3. モデルグレーダー（LLM-as-judgeルーブリック）
4. ヒューマングレーダー（曖昧な出力に対する手動判定）

### pass@kガイダンス

- `pass@1`：直接的な信頼性
- `pass@3`：制御されたリトライ下での実用的信頼性
- `pass^3`：安定性テスト（3回すべてパスする必要あり）

推奨閾値：
- ケイパビリティ評価：pass@3 >= 0.90
- リグレッション評価：リリースクリティカルパスでpass^3 = 1.00

### 評価アンチパターン

- 既知の評価例にプロンプトをオーバーフィッティング
- ハッピーパスの出力のみを測定
- パス率を追求しながらコストとレイテンシのドリフトを無視
- リリースゲートでフレーキーなグレーダーを許可

### 最小評価アーティファクトレイアウト

- `.claude/evals/<feature>.md` 定義
- `.claude/evals/<feature>.log` 実行履歴
- `docs/releases/<version>/eval-summary.md` リリーススナップショット
