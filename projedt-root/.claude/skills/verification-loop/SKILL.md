---
name: verification-loop
description: "Claude Codeセッションのための包括的な検証システム。"
origin: ECC
---

# 検証ループスキル

Claude Codeセッションのための包括的な検証システム。

## 使用タイミング

このスキルを呼び出すタイミング:
- 機能または重要なコード変更の完了後
- PR作成前
- 品質ゲートが通過していることを確認したいとき
- リファクタリング後

## 検証フェーズ

### フェーズ1: ビルド検証
```bash
# Check if project builds
npm run build 2>&1 | tail -20
# OR
pnpm build 2>&1 | tail -20
```

ビルドが失敗した場合、続行前に停止して修正。

### フェーズ2: 型チェック
```bash
# TypeScript projects
npx tsc --noEmit 2>&1 | head -30

# Python projects
pyright . 2>&1 | head -30
```

すべての型エラーを報告。クリティカルなものは続行前に修正。

### フェーズ3: リントチェック
```bash
# JavaScript/TypeScript
npm run lint 2>&1 | head -30

# Python
ruff check . 2>&1 | head -30
```

### フェーズ4: テストスイート
```bash
# Run tests with coverage
npm run test -- --coverage 2>&1 | tail -50

# Check coverage threshold
# Target: 80% minimum
```

レポート:
- テスト総数: X
- 通過: X
- 失敗: X
- カバレッジ: X%

### フェーズ5: セキュリティスキャン
```bash
# Check for secrets
grep -rn "sk-" --include="*.ts" --include="*.js" . 2>/dev/null | head -10
grep -rn "api_key" --include="*.ts" --include="*.js" . 2>/dev/null | head -10

# Check for console.log
grep -rn "console.log" --include="*.ts" --include="*.tsx" src/ 2>/dev/null | head -10
```

### フェーズ6: 差分レビュー
```bash
# Show what changed
git diff --stat
git diff HEAD~1 --name-only
```

変更された各ファイルを以下の観点でレビュー:
- 意図しない変更
- 欠落しているエラーハンドリング
- 潜在的なエッジケース

## 出力フォーマット

すべてのフェーズ実行後、検証レポートを作成:

```
検証レポート
==================

ビルド:     [PASS/FAIL]
型:         [PASS/FAIL] (Xエラー)
リント:     [PASS/FAIL] (X警告)
テスト:     [PASS/FAIL] (X/Y通過, Z%カバレッジ)
セキュリティ: [PASS/FAIL] (X問題)
差分:       [Xファイル変更]

総合:       PR[準備完了/未準備]

修正すべき問題:
1. ...
2. ...
```

## 継続モード

長時間セッションでは、15分ごとまたは大きな変更後に検証を実行:

```markdown
メンタルチェックポイントを設定:
- 各関数の完了後
- コンポーネントの完了後
- 次のタスクに移る前

実行: /verify
```

## フックとの統合

このスキルはPostToolUseフックを補完しますが、より深い検証を提供します。
フックは問題を即座に捕捉し、このスキルは包括的なレビューを提供します。
