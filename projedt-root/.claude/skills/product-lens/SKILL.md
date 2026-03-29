# Product Lens — 構築前に考える

## 使用するタイミング

- 機能開発前 — 「なぜ」をバリデーションする
- 週次プロダクトレビュー — 正しいものを構築しているか？
- 機能の選択で迷った場合
- ローンチ前 — ユーザージャーニーのサニティチェック
- 曖昧なアイデアを仕様に変換する場合

## 仕組み

### モード1: プロダクト診断

YCオフィスアワーの自動化版。厳しい質問をする:

```
1. Who is this for? (specific person, not "developers")
2. What's the pain? (quantify: how often, how bad, what do they do today?)
3. Why now? (what changed that makes this possible/necessary?)
4. What's the 10-star version? (if money/time were unlimited)
5. What's the MVP? (smallest thing that proves the thesis)
6. What's the anti-goal? (what are you explicitly NOT building?)
7. How do you know it's working? (metric, not vibes)
```

出力: 回答、リスク、ゴー/ノーゴーの推奨事項を含む `PRODUCT-BRIEF.md`。

### モード2: ファウンダーレビュー

現在のプロジェクトをファウンダーの視点でレビューする:

```
1. Read README, CLAUDE.md, package.json, recent commits
2. Infer: what is this trying to be?
3. Score: product-market fit signals (0-10)
   - Usage growth trajectory
   - Retention indicators (repeat contributors, return users)
   - Revenue signals (pricing page, billing code, Stripe integration)
   - Competitive moat (what's hard to copy?)
4. Identify: the one thing that would 10x this
5. Flag: things you're building that don't matter
```

### モード3: ユーザージャーニー監査

実際のユーザー体験をマッピングする:

```
1. Clone/install the product as a new user
2. Document every friction point (confusing steps, errors, missing docs)
3. Time each step
4. Compare to competitor onboarding
5. Score: time-to-value (how long until the user gets their first win?)
6. Recommend: top 3 fixes for onboarding
```

### モード4: 機能優先順位付け

10個のアイデアがあって2つ選ぶ必要がある場合:

```
1. List all candidate features
2. Score each on: impact (1-5) × confidence (1-5) ÷ effort (1-5)
3. Rank by ICE score
4. Apply constraints: runway, team size, dependencies
5. Output: prioritized roadmap with rationale
```

## 出力

すべてのモードはエッセイではなく、アクショナブルなドキュメントを出力する。すべての推奨事項には具体的な次のステップがある。

## 統合

併用推奨:
- `/browser-qa` でユーザージャーニー監査の結果を検証
- `/design-system audit` でビジュアルポリッシュの評価
- `/canary-watch` でローンチ後のモニタリング
