# ベンチマーク — パフォーマンスベースラインとリグレッション検出

## 使用タイミング

- PRの前後でパフォーマンスへの影響を測定する場合
- プロジェクトのパフォーマンスベースラインを設定する場合
- ユーザーから「遅く感じる」と報告された場合
- リリース前 — パフォーマンス目標を達成していることを確認する場合
- 自分のスタックを他の選択肢と比較する場合

## 動作の仕組み

### モード1：ページパフォーマンス

ブラウザMCPを介して実際のブラウザメトリクスを測定します：

```
1. Navigate to each target URL
2. Measure Core Web Vitals:
   - LCP (Largest Contentful Paint) — target < 2.5s
   - CLS (Cumulative Layout Shift) — target < 0.1
   - INP (Interaction to Next Paint) — target < 200ms
   - FCP (First Contentful Paint) — target < 1.8s
   - TTFB (Time to First Byte) — target < 800ms
3. Measure resource sizes:
   - Total page weight (target < 1MB)
   - JS bundle size (target < 200KB gzipped)
   - CSS size
   - Image weight
   - Third-party script weight
4. Count network requests
5. Check for render-blocking resources
```

### モード2：APIパフォーマンス

APIエンドポイントをベンチマークします：

```
1. Hit each endpoint 100 times
2. Measure: p50, p95, p99 latency
3. Track: response size, status codes
4. Test under load: 10 concurrent requests
5. Compare against SLA targets
```

### モード3：ビルドパフォーマンス

開発フィードバックループを測定します：

```
1. Cold build time
2. Hot reload time (HMR)
3. Test suite duration
4. TypeScript check time
5. Lint time
6. Docker build time
```

### モード4：変更前後の比較

変更の前後で実行して影響を測定します：

```
/benchmark baseline    # saves current metrics
# ... make changes ...
/benchmark compare     # compares against baseline
```

出力：
```
| Metric | Before | After | Delta | Verdict |
|--------|--------|-------|-------|---------|
| LCP | 1.2s | 1.4s | +200ms | ⚠ WARN |
| Bundle | 180KB | 175KB | -5KB | ✓ BETTER |
| Build | 12s | 14s | +2s | ⚠ WARN |
```

## 出力

ベースラインは`.ecc/benchmarks/`にJSONとして保存されます。Gitで追跡されるため、チームでベースラインを共有できます。

## 連携

- CI: すべてのPRで`/benchmark compare`を実行
- `/canary-watch`と組み合わせてデプロイ後のモニタリング
- `/browser-qa`と組み合わせて完全なリリース前チェックリスト
