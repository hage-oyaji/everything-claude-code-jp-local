---
name: iterative-retrieval
description: サブエージェントのコンテキスト問題を解決するための段階的コンテキスト取得パターン
origin: ECC
---

# イテレーティブリトリーバルパターン

マルチエージェントワークフローにおける「コンテキスト問題」を解決します。サブエージェントは作業を開始するまで必要なコンテキストがわかりません。

## アクティベーション条件

- 事前に予測できないコードベースコンテキストを必要とするサブエージェントの生成
- コンテキストが段階的に精緻化されるマルチエージェントワークフローの構築
- エージェントタスクで「コンテキストが大きすぎる」または「コンテキストが不足」の失敗が発生
- コード探索のためのRAG的な取得パイプラインの設計
- エージェントオーケストレーションにおけるトークン使用の最適化

## 問題

サブエージェントは限られたコンテキストで生成されます。以下がわかりません:
- どのファイルに関連コードが含まれているか
- コードベースにどのようなパターンが存在するか
- プロジェクトがどのような用語を使用しているか

標準的なアプローチは失敗します:
- **すべてを送信**: コンテキスト制限を超過
- **何も送信しない**: エージェントが重要な情報を欠く
- **必要なものを推測**: しばしば誤り

## ソリューション: イテレーティブリトリーバル

コンテキストを段階的に精緻化する4フェーズのループ:

```
┌─────────────────────────────────────────────┐
│                                             │
│   ┌──────────┐      ┌──────────┐            │
│   │ DISPATCH │─────▶│ EVALUATE │            │
│   └──────────┘      └──────────┘            │
│        ▲                  │                 │
│        │                  ▼                 │
│   ┌──────────┐      ┌──────────┐            │
│   │   LOOP   │◀─────│  REFINE  │            │
│   └──────────┘      └──────────┘            │
│                                             │
│        Max 3 cycles, then proceed           │
└─────────────────────────────────────────────┘
```

### フェーズ1: DISPATCH

候補ファイルを収集するための初期の広範なクエリ:

```javascript
// Start with high-level intent
const initialQuery = {
  patterns: ['src/**/*.ts', 'lib/**/*.ts'],
  keywords: ['authentication', 'user', 'session'],
  excludes: ['*.test.ts', '*.spec.ts']
};

// Dispatch to retrieval agent
const candidates = await retrieveFiles(initialQuery);
```

### フェーズ2: EVALUATE

取得したコンテンツの関連性を評価:

```javascript
function evaluateRelevance(files, task) {
  return files.map(file => ({
    path: file.path,
    relevance: scoreRelevance(file.content, task),
    reason: explainRelevance(file.content, task),
    missingContext: identifyGaps(file.content, task)
  }));
}
```

スコアリング基準:
- **高 (0.8-1.0)**: 対象機能を直接実装
- **中 (0.5-0.7)**: 関連するパターンや型を含む
- **低 (0.2-0.4)**: 間接的に関連
- **なし (0-0.2)**: 関連なし、除外

### フェーズ3: REFINE

評価に基づいて検索条件を更新:

```javascript
function refineQuery(evaluation, previousQuery) {
  return {
    // Add new patterns discovered in high-relevance files
    patterns: [...previousQuery.patterns, ...extractPatterns(evaluation)],

    // Add terminology found in codebase
    keywords: [...previousQuery.keywords, ...extractKeywords(evaluation)],

    // Exclude confirmed irrelevant paths
    excludes: [...previousQuery.excludes, ...evaluation
      .filter(e => e.relevance < 0.2)
      .map(e => e.path)
    ],

    // Target specific gaps
    focusAreas: evaluation
      .flatMap(e => e.missingContext)
      .filter(unique)
  };
}
```

### フェーズ4: LOOP

精緻化された条件で繰り返し（最大3サイクル）:

```javascript
async function iterativeRetrieve(task, maxCycles = 3) {
  let query = createInitialQuery(task);
  let bestContext = [];

  for (let cycle = 0; cycle < maxCycles; cycle++) {
    const candidates = await retrieveFiles(query);
    const evaluation = evaluateRelevance(candidates, task);

    // Check if we have sufficient context
    const highRelevance = evaluation.filter(e => e.relevance >= 0.7);
    if (highRelevance.length >= 3 && !hasCriticalGaps(evaluation)) {
      return highRelevance;
    }

    // Refine and continue
    query = refineQuery(evaluation, query);
    bestContext = mergeContext(bestContext, highRelevance);
  }

  return bestContext;
}
```

## 実践的な事例

### 事例1: バグ修正のコンテキスト

```
Task: "Fix the authentication token expiry bug"

Cycle 1:
  DISPATCH: Search for "token", "auth", "expiry" in src/**
  EVALUATE: Found auth.ts (0.9), tokens.ts (0.8), user.ts (0.3)
  REFINE: Add "refresh", "jwt" keywords; exclude user.ts

Cycle 2:
  DISPATCH: Search refined terms
  EVALUATE: Found session-manager.ts (0.95), jwt-utils.ts (0.85)
  REFINE: Sufficient context (2 high-relevance files)

Result: auth.ts, tokens.ts, session-manager.ts, jwt-utils.ts
```

### 事例2: 機能実装

```
Task: "Add rate limiting to API endpoints"

Cycle 1:
  DISPATCH: Search "rate", "limit", "api" in routes/**
  EVALUATE: No matches - codebase uses "throttle" terminology
  REFINE: Add "throttle", "middleware" keywords

Cycle 2:
  DISPATCH: Search refined terms
  EVALUATE: Found throttle.ts (0.9), middleware/index.ts (0.7)
  REFINE: Need router patterns

Cycle 3:
  DISPATCH: Search "router", "express" patterns
  EVALUATE: Found router-setup.ts (0.8)
  REFINE: Sufficient context

Result: throttle.ts, middleware/index.ts, router-setup.ts
```

## エージェントとの統合

エージェントプロンプトでの使用:

```markdown
When retrieving context for this task:
1. Start with broad keyword search
2. Evaluate each file's relevance (0-1 scale)
3. Identify what context is still missing
4. Refine search criteria and repeat (max 3 cycles)
5. Return files with relevance >= 0.7
```

## ベストプラクティス

1. **広く始めて段階的に絞り込む** - 初回クエリを過度に限定しない
2. **コードベースの用語を学習する** - 最初のサイクルで命名規則が明らかになることが多い
3. **不足を追跡する** - 明示的なギャップ特定が精緻化を駆動する
4. **「十分」で止める** - 3つの高関連ファイルは10の普通のファイルに勝る
5. **自信を持って除外する** - 低関連ファイルが関連性を持つことはない

## 関連

- [The Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352) - サブエージェントオーケストレーションセクション
- `continuous-learning` スキル - 時間とともに改善するパターン用
- ECCにバンドルされたエージェント定義（手動インストールパス: `agents/`）
