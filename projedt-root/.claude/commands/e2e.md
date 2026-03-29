---
description: Playwrightでエンドツーエンドテストを生成・実行する。テストジャーニーの作成、テスト実行、スクリーンショット/動画/トレースのキャプチャ、アーティファクトのアップロードを行う。
---

# E2E コマンド

このコマンドは**e2e-runner**エージェントを呼び出し、Playwrightを使用したエンドツーエンドテストの生成、保守、実行を行います。

## このコマンドの動作

1. **テストジャーニーの生成** - ユーザーフローのPlaywrightテストを作成
2. **E2Eテストの実行** - 複数ブラウザでテストを実行
3. **アーティファクトのキャプチャ** - 失敗時のスクリーンショット、動画、トレース
4. **結果のアップロード** - HTMLレポートとJUnit XML
5. **フレーキーテストの特定** - 不安定なテストを隔離

## 使用場面

`/e2e` を使用する場面:
- クリティカルなユーザージャーニーのテスト（ログイン、取引、決済）
- マルチステップフローがエンドツーエンドで動作することの確認
- UIインタラクションとナビゲーションのテスト
- フロントエンドとバックエンドの統合検証
- 本番デプロイの準備

## 動作の仕組み

e2e-runnerエージェントは以下を実行します:

1. **ユーザーフローを分析**し、テストシナリオを特定する
2. Page Object Modelパターンを使用して**Playwrightテストを生成**する
3. 複数ブラウザ（Chrome、Firefox、Safari）で**テストを実行**する
4. スクリーンショット、動画、トレースで**失敗をキャプチャ**する
5. 結果とアーティファクトを含む**レポートを生成**する
6. **フレーキーテストを特定**し、修正を推奨する

## 使用例

```
User: /e2e Test the market search and view flow

Agent (e2e-runner):
# E2Eテスト生成: マーケット検索と表示フロー

## 特定されたテストシナリオ

**ユーザージャーニー:** マーケット検索 → 結果表示 → マーケットクリック → 詳細表示

**テストカバレッジ:**
1. マーケットページへの遷移
2. セマンティック検索の実行
3. 検索結果の確認
4. 最初の結果をクリック
5. マーケット詳細ページの読み込み確認
6. チャートのレンダリング確認

## 生成されたテストコード

```typescript
// tests/e2e/markets/search-and-view.spec.ts
import { test, expect } from '@playwright/test'
import { MarketsPage } from '../../pages/MarketsPage'
import { MarketDetailsPage } from '../../pages/MarketDetailsPage'

test.describe('Market Search and View Flow', () => {
  test('user can search markets and view details', async ({ page }) => {
    // 1. Navigate to markets page
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()

    // Verify page loaded
    await expect(page).toHaveTitle(/Markets/)
    await expect(page.locator('h1')).toContainText('Markets')

    // 2. Perform semantic search
    await marketsPage.searchMarkets('election')

    // Wait for API response
    await page.waitForResponse(resp =>
      resp.url().includes('/api/markets/search') && resp.status() === 200
    )

    // 3. Verify search results
    const marketCards = marketsPage.marketCards
    await expect(marketCards.first()).toBeVisible()
    const resultCount = await marketCards.count()
    expect(resultCount).toBeGreaterThan(0)

    // Take screenshot of search results
    await page.screenshot({ path: 'artifacts/search-results.png' })

    // 4. Click on first result
    const firstMarketTitle = await marketCards.first().textContent()
    await marketCards.first().click()

    // 5. Verify market details page loads
    await expect(page).toHaveURL(/\/markets\/[a-z0-9-]+/)

    const detailsPage = new MarketDetailsPage(page)
    await expect(detailsPage.marketName).toBeVisible()
    await expect(detailsPage.marketDescription).toBeVisible()

    // 6. Verify chart renders
    await expect(detailsPage.priceChart).toBeVisible()

    // Verify market name matches
    const detailsTitle = await detailsPage.marketName.textContent()
    expect(detailsTitle?.toLowerCase()).toContain(
      firstMarketTitle?.toLowerCase().substring(0, 20) || ''
    )

    // Take screenshot of market details
    await page.screenshot({ path: 'artifacts/market-details.png' })
  })

  test('search with no results shows empty state', async ({ page }) => {
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()

    // Search for non-existent market
    await marketsPage.searchMarkets('xyznonexistentmarket123456')

    // Verify empty state
    await expect(page.locator('[data-testid="no-results"]')).toBeVisible()
    await expect(page.locator('[data-testid="no-results"]')).toContainText(
      /no.*results|no.*markets/i
    )

    const marketCount = await marketsPage.marketCards.count()
    expect(marketCount).toBe(0)
  })

  test('can clear search and see all markets again', async ({ page }) => {
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()

    // Initial market count
    const initialCount = await marketsPage.marketCards.count()

    // Perform search
    await marketsPage.searchMarkets('trump')
    await page.waitForLoadState('networkidle')

    // Verify filtered results
    const filteredCount = await marketsPage.marketCards.count()
    expect(filteredCount).toBeLessThan(initialCount)

    // Clear search
    await marketsPage.searchInput.clear()
    await page.waitForLoadState('networkidle')

    // Verify all markets shown again
    const finalCount = await marketsPage.marketCards.count()
    expect(finalCount).toBe(initialCount)
  })
})
```

## テストの実行

```bash
# 生成されたテストを実行
npx playwright test tests/e2e/markets/search-and-view.spec.ts

Running 3 tests using 3 workers

  ✓  [chromium] › search-and-view.spec.ts:5:3 › user can search markets and view details (4.2s)
  ✓  [chromium] › search-and-view.spec.ts:52:3 › search with no results shows empty state (1.8s)
  ✓  [chromium] › search-and-view.spec.ts:67:3 › can clear search and see all markets again (2.9s)

  3 passed (9.1s)

Artifacts generated:
- artifacts/search-results.png
- artifacts/market-details.png
- playwright-report/index.html
```

## テストレポート

```
╔══════════════════════════════════════════════════════════════╗
║                    E2E テスト結果                             ║
╠══════════════════════════════════════════════════════════════╣
║ ステータス:  ✅ 全テスト合格                                    ║
║ 合計:       3テスト                                           ║
║ 合格:       3 (100%)                                         ║
║ 失敗:       0                                                ║
║ フレーキー:  0                                                ║
║ 所要時間:   9.1秒                                             ║
╚══════════════════════════════════════════════════════════════╝

アーティファクト:
📸 スクリーンショット: 2ファイル
📹 動画: 0ファイル（失敗時のみ）
🔍 トレース: 0ファイル（失敗時のみ）
📊 HTMLレポート: playwright-report/index.html

レポートを表示: npx playwright show-report
```

✅ E2Eテストスイート、CI/CD統合の準備完了！
```

## テストアーティファクト

テスト実行時に以下のアーティファクトがキャプチャされます:

**全テスト共通:**
- タイムラインと結果を含むHTMLレポート
- CI統合用のJUnit XML

**失敗時のみ:**
- 失敗状態のスクリーンショット
- テストの動画記録
- デバッグ用トレースファイル（ステップバイステップのリプレイ）
- ネットワークログ
- コンソールログ

## アーティファクトの表示

```bash
# ブラウザでHTMLレポートを表示
npx playwright show-report

# 特定のトレースファイルを表示
npx playwright show-trace artifacts/trace-abc123.zip

# スクリーンショットはartifacts/ディレクトリに保存される
open artifacts/search-results.png
```

## フレーキーテストの検出

テストが断続的に失敗する場合:

```
⚠️  フレーキーテスト検出: tests/e2e/markets/trade.spec.ts

テストは10回中7回合格（合格率70%）

一般的な失敗:
"Timeout waiting for element '[data-testid="confirm-btn"]'"

推奨される修正:
1. 明示的な待機を追加: await page.waitForSelector('[data-testid="confirm-btn"]')
2. タイムアウトを増加: { timeout: 10000 }
3. コンポーネントの競合状態を確認
4. アニメーションで要素が隠れていないか確認

隔離の推奨: 修正まで test.fixme() としてマーク
```

## ブラウザ設定

テストはデフォルトで複数のブラウザで実行されます:
- ✅ Chromium（デスクトップChrome）
- ✅ Firefox（デスクトップ）
- ✅ WebKit（デスクトップSafari）
- ✅ モバイルChrome（オプション）

ブラウザの調整は `playwright.config.ts` で設定します。

## CI/CD統合

CIパイプラインに追加:

```yaml
# .github/workflows/e2e.yml
- name: Install Playwright
  run: npx playwright install --with-deps

- name: Run E2E tests
  run: npx playwright test

- name: Upload artifacts
  if: always()
  uses: actions/upload-artifact@v3
  with:
    name: playwright-report
    path: playwright-report/
```

## PMX固有のクリティカルフロー

PMXでは、以下のE2Eテストを優先する:

**🔴 CRITICAL（常に合格必須）:**
1. ユーザーがウォレットを接続できる
2. ユーザーがマーケットを閲覧できる
3. ユーザーがマーケットを検索できる（セマンティック検索）
4. ユーザーがマーケット詳細を表示できる
5. ユーザーが取引を実行できる（テスト資金）
6. マーケットが正しく解決される
7. ユーザーが資金を引き出せる

**🟡 IMPORTANT:**
1. マーケット作成フロー
2. ユーザープロフィールの更新
3. リアルタイム価格更新
4. チャートのレンダリング
5. マーケットのフィルターとソート
6. モバイルレスポンシブレイアウト

## ベストプラクティス

**推奨:**
- ✅ 保守性のためにPage Object Modelを使用する
- ✅ セレクターにdata-testid属性を使用する
- ✅ 任意のタイムアウトではなくAPIレスポンスを待つ
- ✅ クリティカルなユーザージャーニーをエンドツーエンドでテストする
- ✅ mainへのマージ前にテストを実行する
- ✅ テスト失敗時にアーティファクトをレビューする

**非推奨:**
- ❌ 脆弱なセレクター（CSSクラスは変更される可能性がある）
- ❌ 実装の詳細をテスト
- ❌ 本番環境に対してテストを実行
- ❌ フレーキーテストを無視
- ❌ 失敗時のアーティファクトレビューをスキップ
- ❌ すべてのエッジケースをE2Eでテスト（ユニットテストを使用する）

## 重要な注意事項

**PMXにおいて重要:**
- 実際のお金に関わるE2Eテストは必ずテストネット/ステージング環境のみで実行
- 本番環境に対して取引テストを実行しない
- 金融テストには `test.skip(process.env.NODE_ENV === 'production')` を設定
- 少額のテスト資金のみを持つテストウォレットを使用

## 他のコマンドとの連携

- `/plan` でテストすべきクリティカルジャーニーを特定
- `/tdd` でユニットテスト（より高速で詳細）
- `/e2e` で統合テストとユーザージャーニーテスト
- `/code-review` でテスト品質の確認

## 関連エージェント

このコマンドはECCが提供する `e2e-runner` エージェントを呼び出します。

手動インストールの場合、ソースファイルは以下にあります:
`agents/e2e-runner.md`

## クイックコマンド

```bash
# 全E2Eテストを実行
npx playwright test

# 特定のテストファイルを実行
npx playwright test tests/e2e/markets/search.spec.ts

# ヘッドモードで実行（ブラウザを表示）
npx playwright test --headed

# テストをデバッグ
npx playwright test --debug

# テストコードを生成
npx playwright codegen http://localhost:3000

# レポートを表示
npx playwright show-report
```
