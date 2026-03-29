# カナリーウォッチ — デプロイ後モニタリング

## 使用タイミング

- 本番環境またはステージングにデプロイした後
- リスクのあるPRをマージした後
- 修正が実際に機能したことを確認したい場合
- リリースウィンドウ中の継続的モニタリング
- 依存関係のアップグレード後

## 動作の仕組み

デプロイされたURLのリグレッションをモニタリングします。停止されるか、ウォッチウィンドウが期限切れになるまでループで実行されます。

### 監視項目

```
1. HTTP Status — is the page returning 200?
2. Console Errors — new errors that weren't there before?
3. Network Failures — failed API calls, 5xx responses?
4. Performance — LCP/CLS/INP regression vs baseline?
5. Content — did key elements disappear? (h1, nav, footer, CTA)
6. API Health — are critical endpoints responding within SLA?
```

### ウォッチモード

**クイックチェック**（デフォルト）: 1回のパス、結果をレポート
```
/canary-watch https://myapp.com
```

**持続ウォッチ**: N分ごとにM時間チェック
```
/canary-watch https://myapp.com --interval 5m --duration 2h
```

**差分モード**: ステージング vs 本番を比較
```
/canary-watch --compare https://staging.myapp.com https://myapp.com
```

### アラート閾値

```yaml
critical:  # immediate alert
  - HTTP status != 200
  - Console error count > 5 (new errors only)
  - LCP > 4s
  - API endpoint returns 5xx

warning:   # flag in report
  - LCP increased > 500ms from baseline
  - CLS > 0.1
  - New console warnings
  - Response time > 2x baseline

info:      # log only
  - Minor performance variance
  - New network requests (third-party scripts added?)
```

### 通知

クリティカル閾値を超えた場合：
- デスクトップ通知（macOS/Linux）
- オプション: Slack/Discord Webhook
- `~/.claude/canary-watch.log`にログ出力

## 出力

```markdown
## Canary Report — myapp.com — 2026-03-23 03:15 PST

### Status: HEALTHY ✓

| Check | Result | Baseline | Delta |
|-------|--------|----------|-------|
| HTTP | 200 ✓ | 200 | — |
| Console errors | 0 ✓ | 0 | — |
| LCP | 1.8s ✓ | 1.6s | +200ms |
| CLS | 0.01 ✓ | 0.01 | — |
| API /health | 145ms ✓ | 120ms | +25ms |

### No regressions detected. Deploy is clean.
```

## 連携

組み合わせ先：
- `/browser-qa` — デプロイ前の検証
- フック: `git push`のPostToolUseフックとして追加し、デプロイ後に自動チェック
- CI: デプロイステップ後にGitHub Actionsで実行
