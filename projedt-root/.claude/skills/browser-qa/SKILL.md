# ブラウザQA — 自動ビジュアルテストとインタラクション

## 使用タイミング

- ステージング/プレビューに機能をデプロイした後
- 複数ページにまたがるUI動作を検証する必要がある場合
- リリース前 — レイアウト、フォーム、インタラクションが実際に動作することを確認する場合
- フロントエンドコードに変更を加えるPRをレビューする場合
- アクセシビリティ監査とレスポンシブテスト

## 動作の仕組み

ブラウザ自動化MCP（claude-in-chrome、Playwright、またはPuppeteer）を使用して、実際のユーザーのようにライブページを操作します。

### フェーズ1：スモークテスト
```
1. Navigate to target URL
2. Check for console errors (filter noise: analytics, third-party)
3. Verify no 4xx/5xx in network requests
4. Screenshot above-the-fold on desktop + mobile viewport
5. Check Core Web Vitals: LCP < 2.5s, CLS < 0.1, INP < 200ms
```

### フェーズ2：インタラクションテスト
```
1. Click every nav link — verify no dead links
2. Submit forms with valid data — verify success state
3. Submit forms with invalid data — verify error state
4. Test auth flow: login → protected page → logout
5. Test critical user journeys (checkout, onboarding, search)
```

### フェーズ3：ビジュアルリグレッション
```
1. Screenshot key pages at 3 breakpoints (375px, 768px, 1440px)
2. Compare against baseline screenshots (if stored)
3. Flag layout shifts > 5px, missing elements, overflow
4. Check dark mode if applicable
```

### フェーズ4：アクセシビリティ
```
1. Run axe-core or equivalent on each page
2. Flag WCAG AA violations (contrast, labels, focus order)
3. Verify keyboard navigation works end-to-end
4. Check screen reader landmarks
```

## 出力形式

```markdown
## QA Report — [URL] — [timestamp]

### Smoke Test
- Console errors: 0 critical, 2 warnings (analytics noise)
- Network: all 200/304, no failures
- Core Web Vitals: LCP 1.2s ✓, CLS 0.02 ✓, INP 89ms ✓

### Interactions
- [✓] Nav links: 12/12 working
- [✗] Contact form: missing error state for invalid email
- [✓] Auth flow: login/logout working

### Visual
- [✗] Hero section overflows on 375px viewport
- [✓] Dark mode: all pages consistent

### Accessibility
- 2 AA violations: missing alt text on hero image, low contrast on footer links

### Verdict: SHIP WITH FIXES (2 issues, 0 blockers)
```

## 連携

任意のブラウザMCPで動作します：
- `mChild__claude-in-chrome__*` ツール（推奨 — 実際のChromeを使用）
- `mcp__browserbase__*` 経由のPlaywright
- Puppeteerスクリプトの直接実行

デプロイ後のモニタリングには`/canary-watch`と組み合わせてください。
