---
name: codebase-onboarding
description: 未知のコードベースを分析し、アーキテクチャマップ、主要エントリポイント、規約、スターター CLAUDE.md を含む構造化されたオンボーディングガイドを生成する。新しいプロジェクトに参加する際や、リポジトリで初めて Claude Code をセットアップする際に使用。
origin: ECC
---

# コードベースオンボーディング

未知のコードベースを体系的に分析し、構造化されたオンボーディングガイドを作成する。新しいプロジェクトに参加する開発者や、既存リポジトリで初めて Claude Code をセットアップする場合向け。

## いつ使うか

- プロジェクトで初めて Claude Code を開く
- 新しいチームやリポジトリに参加する
- ユーザーが「このコードベースの理解を助けて」と依頼する
- ユーザーがプロジェクト用の CLAUDE.md の生成を依頼する
- ユーザーが「オンボードして」や「このリポジトリを案内して」と言う

## 仕組み

### フェーズ 1：偵察

すべてのファイルを読まずに、プロジェクトに関する生のシグナルを収集する。以下のチェックを並列に実行：

```
1. Package manifest detection
   → package.json, go.mod, Cargo.toml, pyproject.toml, pom.xml, build.gradle,
     Gemfile, composer.json, mix.exs, pubspec.yaml

2. Framework fingerprinting
   → next.config.*, nuxt.config.*, angular.json, vite.config.*,
     django settings, flask app factory, fastapi main, rails config

3. Entry point identification
   → main.*, index.*, app.*, server.*, cmd/, src/main/

4. Directory structure snapshot
   → Top 2 levels of the directory tree, ignoring node_modules, vendor,
     .git, dist, build, __pycache__, .next

5. Config and tooling detection
   → .eslintrc*, .prettierrc*, tsconfig.json, Makefile, Dockerfile,
     docker-compose*, .github/workflows/, .env.example, CI configs

6. Test structure detection
   → tests/, test/, __tests__/, *_test.go, *.spec.ts, *.test.js,
     pytest.ini, jest.config.*, vitest.config.*
```

### フェーズ 2：アーキテクチャマッピング

偵察データから以下を特定：

**技術スタック**
- 言語とバージョン制約
- フレームワークと主要ライブラリ
- データベースと ORM
- ビルドツールとバンドラー
- CI/CD プラットフォーム

**アーキテクチャパターン**
- モノリス、モノレポ、マイクロサービス、サーバーレス
- フロントエンド/バックエンド分離またはフルスタック
- API スタイル：REST、GraphQL、gRPC、tRPC

**主要ディレクトリ**
トップレベルのディレクトリをその目的にマッピング：

<!-- Example for a React project — replace with detected directories -->
```
src/components/  → React UI components
src/api/         → API route handlers
src/lib/         → Shared utilities
src/db/          → Database models and migrations
tests/           → Test suites
scripts/         → Build and deployment scripts
```

**データフロー**
1 つのリクエストをエントリからレスポンスまでトレース：
- リクエストはどこから入るか？（ルーター、ハンドラ、コントローラ）
- どのようにバリデーションされるか？（ミドルウェア、スキーマ、ガード）
- ビジネスロジックはどこにあるか？（サービス、モデル、ユースケース）
- データベースにどう到達するか？（ORM、生クエリ、リポジトリ）

### フェーズ 3：規約の検出

コードベースがすでに従っているパターンを特定：

**命名規約**
- ファイル命名：kebab-case、camelCase、PascalCase、snake_case
- コンポーネント/クラスの命名パターン
- テストファイル命名：`*.test.ts`、`*.spec.ts`、`*_test.go`

**コードパターン**
- エラーハンドリングスタイル：try/catch、Result 型、エラーコード
- 依存性注入または直接インポート
- ステート管理アプローチ
- 非同期パターン：コールバック、Promise、async/await、チャネル

**Git 規約**
- 最近のブランチからブランチ命名規則
- 最近のコミットからコミットメッセージスタイル
- PR ワークフロー（squash、merge、rebase）
- リポジトリにコミットがない、または浅い履歴の場合（例：`git clone --depth 1`）、このセクションをスキップし「Git 履歴が利用不可または規約検出には浅すぎる」と記載

### フェーズ 4：オンボーディング成果物の生成

2 つの出力を生成：

#### 出力 1：オンボーディングガイド

```markdown
# Onboarding Guide: [Project Name]

## Overview
[2-3 sentences: what this project does and who it serves]

## Tech Stack
<!-- Example for a Next.js project — replace with detected stack -->
| Layer | Technology | Version |
|-------|-----------|---------|
| Language | TypeScript | 5.x |
| Framework | Next.js | 14.x |
| Database | PostgreSQL | 16 |
| ORM | Prisma | 5.x |
| Testing | Jest + Playwright | - |

## Architecture
[Diagram or description of how components connect]

## Key Entry Points
<!-- Example for a Next.js project — replace with detected paths -->
- **API routes**: `src/app/api/` — Next.js route handlers
- **UI pages**: `src/app/(dashboard)/` — authenticated pages
- **Database**: `prisma/schema.prisma` — data model source of truth
- **Config**: `next.config.ts` — build and runtime config

## Directory Map
[Top-level directory → purpose mapping]

## Request Lifecycle
[Trace one API request from entry to response]

## Conventions
- [File naming pattern]
- [Error handling approach]
- [Testing patterns]
- [Git workflow]

## Common Tasks
<!-- Example for a Node.js project — replace with detected commands -->
- **Run dev server**: `npm run dev`
- **Run tests**: `npm test`
- **Run linter**: `npm run lint`
- **Database migrations**: `npx prisma migrate dev`
- **Build for production**: `npm run build`

## Where to Look
<!-- Example for a Next.js project — replace with detected paths -->
| I want to... | Look at... |
|--------------|-----------|
| Add an API endpoint | `src/app/api/` |
| Add a UI page | `src/app/(dashboard)/` |
| Add a database table | `prisma/schema.prisma` |
| Add a test | `tests/` matching the source path |
| Change build config | `next.config.ts` |
```

#### 出力 2：スターター CLAUDE.md

検出された規約に基づいてプロジェクト固有の CLAUDE.md を生成または更新する。`CLAUDE.md` がすでに存在する場合は、まず読み込んで拡張する — 既存のプロジェクト固有の指示を保持し、追加・変更された内容を明確に示す。

```markdown
# Project Instructions

## Tech Stack
[Detected stack summary]

## Code Style
- [Detected naming conventions]
- [Detected patterns to follow]

## Testing
- Run tests: `[detected test command]`
- Test pattern: [detected test file convention]
- Coverage: [if configured, the coverage command]

## Build & Run
- Dev: `[detected dev command]`
- Build: `[detected build command]`
- Lint: `[detected lint command]`

## Project Structure
[Key directory → purpose map]

## Conventions
- [Commit style if detectable]
- [PR workflow if detectable]
- [Error handling patterns]
```

## ベストプラクティス

1. **すべてを読まない** — 偵察には Glob と Grep を使用し、すべてのファイルに Read を使わない。曖昧なシグナルの場合にのみ選択的に読む。
2. **推測ではなく検証する** — 設定からフレームワークが検出されても、実際のコードが異なるものを使っている場合、コードを信頼する。
3. **既存の CLAUDE.md を尊重する** — すでに存在する場合は、置き換えるのではなく拡張する。新規と既存の内容を明確にする。
4. **簡潔に保つ** — オンボーディングガイドは 2 分でスキャンできるべき。詳細はガイドではなくコードに属する。
5. **不明点を示す** — 規約を確信を持って検出できない場合、推測するのではなく明記する。「テストランナーを特定できませんでした」は間違った回答よりもよい。

## 避けるべきアンチパターン

- 100 行を超える CLAUDE.md の生成 — 焦点を絞る
- すべての依存関係の列挙 — コードの書き方に影響するものだけをハイライトする
- 明らかなディレクトリ名の説明 — `src/` に説明は不要
- README のコピー — オンボーディングガイドは README にない構造的な洞察を追加する

## 例

### 例 1：新しいリポジトリで初めて
**ユーザー**: 「このコードベースにオンボードして」
**アクション**: 4 フェーズのワークフロー全体を実行 → オンボーディングガイド + スターター CLAUDE.md を生成
**出力**: オンボーディングガイドを会話に直接出力し、プロジェクトルートに `CLAUDE.md` を書き込む

### 例 2：既存プロジェクト用の CLAUDE.md を生成
**ユーザー**: 「このプロジェクトの CLAUDE.md を生成して」
**アクション**: フェーズ 1-3 を実行、オンボーディングガイドはスキップ、CLAUDE.md のみ生成
**出力**: 検出された規約を含むプロジェクト固有の `CLAUDE.md`

### 例 3：既存の CLAUDE.md を拡張
**ユーザー**: 「現在のプロジェクト規約で CLAUDE.md を更新して」
**アクション**: 既存の CLAUDE.md を読み込み、フェーズ 1-3 を実行、新しい発見をマージ
**出力**: 追加内容が明確にマークされた更新済み `CLAUDE.md`
