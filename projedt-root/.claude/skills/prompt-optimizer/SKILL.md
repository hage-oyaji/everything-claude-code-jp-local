---
name: prompt-optimizer
description: >-
  生のプロンプトを分析し、意図とギャップを特定し、ECCコンポーネント
  （スキル/コマンド/エージェント/フック）にマッチングし、すぐに貼り付けて
  使える最適化済みプロンプトを出力します。アドバイザリー専用 — タスク自体は
  実行しません。
  トリガー条件: ユーザーが「optimize prompt」「improve my prompt」
  「how to write a prompt for」「help me prompt」「rewrite this prompt」
  と言った場合、またはプロンプト品質の向上を明示的に依頼した場合。中国語の
  同等表現でもトリガーされます: 「优化prompt」「改进prompt」「怎么写prompt」
  「帮我优化这个指令」。
  トリガーしない条件: ユーザーがタスクの直接実行を求めている場合、または
  「just do it」/「直接做」と言った場合。「优化代码」「优化性能」
  「optimize performance」「optimize this code」と言った場合もトリガーしません
  — これらはリファクタリング/パフォーマンスタスクであり、プロンプト最適化
  ではありません。
origin: community
metadata:
  author: YannJY02
  version: "1.0.0"
---

# プロンプトオプティマイザー

下書きプロンプトを分析・批評し、ECCエコシステムのコンポーネントにマッチングし、
ユーザーがコピー＆ペーストして実行できる完全な最適化済みプロンプトを出力します。

## 使用タイミング

- ユーザーが「optimize this prompt」「improve my prompt」「rewrite this prompt」と言った場合
- ユーザーが「help me write a better prompt for...」と言った場合
- ユーザーが「what's the best way to ask Claude Code to...」と言った場合
- ユーザーが「优化prompt」「改进prompt」「怎么写prompt」「帮我优化这个指令」と言った場合
- ユーザーがプロンプトの下書きを貼り付けてフィードバックや改善を求めた場合
- ユーザーが「I don't know how to prompt for this」と言った場合
- ユーザーが「how should I use ECC for...」と言った場合
- ユーザーが明示的に `/prompt-optimize` を呼び出した場合

### 使用しない場合

- ユーザーがタスクの直接実行を求めている場合（そのまま実行する）
- ユーザーが「优化代码」「优化性能」「optimize this code」「optimize performance」と言った場合 — これらはリファクタリングタスクであり、プロンプト最適化ではない
- ユーザーがECCの設定について質問している場合（代わりに `configure-ecc` を使用）
- ユーザーがスキルの棚卸しを求めている場合（代わりに `skill-stocktake` を使用）
- ユーザーが「just do it」または「直接做」と言った場合

## 仕組み

**アドバイザリー専用 — ユーザーのタスクを実行しないでください。**

コードの記述、ファイルの作成、コマンドの実行、その他の実装アクションは一切行いません。
唯一の出力は分析と最適化されたプロンプトです。

ユーザーが「just do it」「直接做」「don't optimize, just execute」と言った場合、
このスキル内で実装モードに切り替えないでください。このスキルは最適化されたプロンプト
のみを生成するものであり、実行を求める場合は通常のタスクリクエストを行うよう
ユーザーに伝えてください。

以下の6フェーズパイプラインを順次実行します。結果は下記の出力フォーマットで提示してください。

### 分析パイプライン

### フェーズ0: プロジェクト検出

プロンプトを分析する前に、現在のプロジェクトコンテキストを検出します:

1. 作業ディレクトリに `CLAUDE.md` が存在するか確認 — プロジェクトの規約を読み取る
2. プロジェクトファイルから技術スタックを検出:
   - `package.json` → Node.js / TypeScript / React / Next.js
   - `go.mod` → Go
   - `pyproject.toml` / `requirements.txt` → Python
   - `Cargo.toml` → Rust
   - `build.gradle` / `pom.xml` → Java / Kotlin / Spring Boot
   - `Package.swift` → Swift
   - `Gemfile` → Ruby
   - `composer.json` → PHP
   - `*.csproj` / `*.sln` → .NET
   - `Makefile` / `CMakeLists.txt` → C / C++
   - `cpanfile` / `Makefile.PL` → Perl
3. 検出した技術スタックをフェーズ3とフェーズ4で使用するために記録

プロジェクトファイルが見つからない場合（プロンプトが抽象的または新規プロジェクトの場合）、
検出をスキップしフェーズ4で「技術スタック不明」とフラグを立てます。

### フェーズ1: 意図検出

ユーザーのタスクを1つ以上のカテゴリに分類します:

| カテゴリ | シグナルワード | 例 |
|----------|-------------|---------|
| 新機能 | build, create, add, implement, 創建, 実現, 添加 | 「Build a login page」 |
| バグ修正 | fix, broken, not working, error, 修復, 報錯 | 「Fix the auth flow」 |
| リファクタリング | refactor, clean up, restructure, 重構, 整理 | 「Refactor the API layer」 |
| リサーチ | how to, what is, explore, investigate, 怎么, 如何 | 「How to add SSO」 |
| テスト | test, coverage, verify, 測試, 覆蓋率 | 「Add tests for the cart」 |
| レビュー | review, audit, check, 審查, 検査 | 「Review my PR」 |
| ドキュメント | document, update docs, 文檔 | 「Update the API docs」 |
| インフラ | deploy, CI, docker, database, 部署, 数據庫 | 「Set up CI/CD pipeline」 |
| 設計 | design, architecture, plan, 設計, 架構 | 「Design the data model」 |

### フェーズ2: スコープ評価

フェーズ0でプロジェクトが検出された場合、コードベースのサイズをシグナルとして使用します。
それ以外の場合、プロンプトの説明のみから推定し、推定値を不確実とマークします。

| スコープ | ヒューリスティック | オーケストレーション |
|-------|-----------|---------------|
| TRIVIAL | 単一ファイル、50行未満 | 直接実行 |
| LOW | 単一コンポーネントまたはモジュール | 単一のコマンドまたはスキル |
| MEDIUM | 複数コンポーネント、同一ドメイン | コマンドチェーン + /verify |
| HIGH | クロスドメイン、5ファイル以上 | まず /plan、次にフェーズ実行 |
| EPIC | マルチセッション、マルチPR、アーキテクチャの変更 | blueprintスキルでマルチセッション計画を使用 |

### フェーズ3: ECCコンポーネントマッチング

意図 + スコープ + 技術スタック（フェーズ0から）を特定のECCコンポーネントにマッピングします。

#### 意図タイプ別

| 意図 | コマンド | スキル | エージェント |
|--------|----------|--------|--------|
| 新機能 | /plan, /tdd, /code-review, /verify | tdd-workflow, verification-loop | planner, tdd-guide, code-reviewer |
| バグ修正 | /tdd, /build-fix, /verify | tdd-workflow | tdd-guide, build-error-resolver |
| リファクタリング | /refactor-clean, /code-review, /verify | verification-loop | refactor-cleaner, code-reviewer |
| リサーチ | /plan | search-first, iterative-retrieval | — |
| テスト | /tdd, /e2e, /test-coverage | tdd-workflow, e2e-testing | tdd-guide, e2e-runner |
| レビュー | /code-review | security-review | code-reviewer, security-reviewer |
| ドキュメント | /update-docs, /update-codemaps | — | doc-updater |
| インフラ | /plan, /verify | docker-patterns, deployment-patterns, database-migrations | architect |
| 設計 (MEDIUM-HIGH) | /plan | — | planner, architect |
| 設計 (EPIC) | — | blueprint（スキルとして呼び出し） | planner, architect |

#### 技術スタック別

| 技術スタック | 追加スキル | エージェント |
|------------|--------------|-------|
| Python / Django | django-patterns, django-tdd, django-security, django-verification, python-patterns, python-testing | python-reviewer |
| Go | golang-patterns, golang-testing | go-reviewer, go-build-resolver |
| Spring Boot / Java | springboot-patterns, springboot-tdd, springboot-security, springboot-verification, java-coding-standards, jpa-patterns | code-reviewer |
| Kotlin / Android | kotlin-coroutines-flows, compose-multiplatform-patterns, android-clean-architecture | kotlin-reviewer |
| TypeScript / React | frontend-patterns, backend-patterns, coding-standards | code-reviewer |
| Swift / iOS | swiftui-patterns, swift-concurrency-6-2, swift-actor-persistence, swift-protocol-di-testing | code-reviewer |
| PostgreSQL | postgres-patterns, database-migrations | database-reviewer |
| Perl | perl-patterns, perl-testing, perl-security | code-reviewer |
| C++ | cpp-coding-standards, cpp-testing | code-reviewer |
| その他 / 未リスト | coding-standards（汎用） | code-reviewer |

### フェーズ4: 不足コンテキストの検出

プロンプトに不足している重要な情報をスキャンします。各項目を確認し、
フェーズ0で自動検出されたか、ユーザーが提供する必要があるかをマークします:

- [ ] **技術スタック** — フェーズ0で検出済みか、ユーザーが指定する必要があるか？
- [ ] **対象スコープ** — ファイル、ディレクトリ、またはモジュールが言及されているか？
- [ ] **受入基準** — タスクの完了をどのように判断するか？
- [ ] **エラーハンドリング** — エッジケースと障害モードに対処しているか？
- [ ] **セキュリティ要件** — 認証、入力検証、シークレット？
- [ ] **テスト期待値** — ユニット、インテグレーション、E2E？
- [ ] **パフォーマンス制約** — 負荷、レイテンシ、リソース制限？
- [ ] **UI/UX要件** — デザイン仕様、レスポンシブ、a11y？（フロントエンドの場合）
- [ ] **データベース変更** — スキーマ、マイグレーション、インデックス？（データレイヤーの場合）
- [ ] **既存パターン** — 従うべき参照ファイルまたは規約？
- [ ] **スコープ境界** — 何をしないか？

**重要な項目が3つ以上不足している場合**、最適化されたプロンプトを生成する前に
ユーザーに最大3つの確認質問をしてください。その後、回答を最適化されたプロンプトに
組み込みます。

### フェーズ5: ワークフローとモデルの推奨

このプロンプトが開発ライフサイクルのどこに位置するかを判断します:

```
リサーチ → 計画 → 実装 (TDD) → レビュー → 検証 → コミット
```

MEDIUM以上のタスクでは、常に /plan から開始します。EPICタスクでは、blueprintスキルを使用します。

**モデル推奨**（出力に含める）:

| スコープ | 推奨モデル | 理由 |
|-------|------------------|-----------|
| TRIVIAL-LOW | Sonnet 4.6 | シンプルなタスクに対して高速でコスト効率が良い |
| MEDIUM | Sonnet 4.6 | 標準的な作業に最適なコーディングモデル |
| HIGH | Sonnet 4.6（メイン）+ Opus 4.6（計画） | アーキテクチャにOpus、実装にSonnet |
| EPIC | Opus 4.6（blueprint）+ Sonnet 4.6（実行） | マルチセッション計画に深い推論が必要 |

**マルチプロンプト分割**（HIGH/EPICスコープの場合）:

単一セッションを超えるタスクの場合、順次プロンプトに分割します:
- プロンプト1: リサーチ + 計画（search-firstスキルを使用、次に /plan）
- プロンプト2-N: プロンプトごとに1フェーズを実装（各フェーズは /verify で終了）
- 最終プロンプト: 全フェーズにわたるインテグレーションテスト + /code-review
- セッション間のコンテキスト保持には /save-session と /resume-session を使用

---

## 出力フォーマット

この正確な構造で分析を提示してください。ユーザーの入力と同じ言語で応答してください。

### セクション1: プロンプト診断

**強み:** 元のプロンプトの良い点をリストアップ。

**課題:**

| 課題 | 影響 | 提案する修正 |
|-------|--------|---------------|
| （問題） | （結果） | （修正方法） |

**確認が必要な点:** ユーザーが回答すべき質問の番号付きリスト。
フェーズ0で自動検出された場合は、質問する代わりにその結果を記載。

### セクション2: 推奨ECCコンポーネント

| タイプ | コンポーネント | 目的 |
|------|-----------|---------|
| コマンド | /plan | コーディング前にアーキテクチャを計画 |
| スキル | tdd-workflow | TDD方法論のガイダンス |
| エージェント | code-reviewer | 実装後のレビュー |
| モデル | Sonnet 4.6 | このスコープに推奨 |

### セクション3: 最適化プロンプト — フルバージョン

完全な最適化プロンプトを単一のフェンスドコードブロック内に提示します。
プロンプトは自己完結型で、コピー＆ペーストですぐに使えるものでなければなりません。
以下を含めてください:
- コンテキスト付きの明確なタスク説明
- 技術スタック（検出済みまたは指定済み）
- 適切なワークフロー段階での /command 呼び出し
- 受入基準
- 検証ステップ
- スコープ境界（何をしないか）

blueprintを参照する項目には、「Use the blueprint skill to...」と記載してください
（`/blueprint` ではなく、blueprintはコマンドではなくスキルであるため）。

### セクション4: 最適化プロンプト — クイックバージョン

経験豊富なECCユーザー向けのコンパクト版。意図タイプごとに変更:

| 意図 | クイックパターン |
|--------|--------------|
| 新機能 | `/plan [feature]. /tdd to implement. /code-review. /verify.` |
| バグ修正 | `/tdd — write failing test for [bug]. Fix to green. /verify.` |
| リファクタリング | `/refactor-clean [scope]. /code-review. /verify.` |
| リサーチ | `Use search-first skill for [topic]. /plan based on findings.` |
| テスト | `/tdd [module]. /e2e for critical flows. /test-coverage.` |
| レビュー | `/code-review. Then use security-reviewer agent.` |
| ドキュメント | `/update-docs. /update-codemaps.` |
| EPIC | `Use blueprint skill for "[objective]". Execute phases with /verify gates.` |

### セクション5: 改善の根拠

| 改善内容 | 理由 |
|-------------|--------|
| （追加した内容） | （なぜ重要か） |

### フッター

> 必要なものと異なりますか？調整内容をお伝えいただくか、プロンプト最適化
> ではなく実行をご希望の場合は通常のタスクリクエストを行ってください。

---

## 例

### トリガー例

- 「Optimize this prompt for ECC」
- 「Rewrite this prompt so Claude Code uses the right commands」
- 「帮我优化这个指令」
- 「How should I prompt ECC for this task?」

### 例1: 曖昧な中国語プロンプト（プロジェクト検出あり）

**ユーザー入力:**
```
帮我写一个用户登录页面
```

**フェーズ0で検出:** `package.json` にNext.js 15、TypeScript、Tailwind CSS

**最適化プロンプト（フル）:**
```
使用项目现有技术栈（Next.js 15 + TypeScript + Tailwind CSS）实现用户登录页面。

技术要求：
- 沿用项目现有的组件结构和路由约定
- 表单验证使用项目中已有的验证方案（检查是否已用 Zod/Yup/其他）
- 认证方式：沿用项目现有认证方案（如无，默认 JWT）
- 包含：邮箱/密码登录表单、表单验证、错误提示、加载状态、响应式布局

工作流：
1. /plan 先规划组件结构和认证流程，参考现有页面的模式
2. /tdd 测试先行：编写登录表单的单元测试和认证流程的集成测试
3. 实现登录页面和认证逻辑
4. /code-review 审查实现
5. /verify 验证所有测试通过且页面正常渲染

安全要求：
- 密码不明文传输
- 防止暴力破解（rate limiting）
- XSS 防护
- CSRF token

验收标准：
- 所有测试通过，覆盖率 80%+
- 页面在移动端和桌面端正常渲染
- 登录成功跳转到 dashboard，失败显示错误信息

不要做：
- 不要实现注册页面
- 不要实现忘记密码功能
- 不要修改现有的路由结构
```

### 例2: 中程度の英語プロンプト

**ユーザー入力:**
```
Add a REST API endpoint for user profile updates with validation
```

**フェーズ0で検出:** `go.mod` にGo 1.22、Chiルーター

**最適化プロンプト（フル）:**
```
Add a REST API endpoint for user profile updates (PATCH /api/users/:id).

Tech stack: Go 1.22 + Chi router (detected from project)

Requirements:
- PATCH /api/users/:id — partial update of user profile
- Input validation for fields: name, email, avatar_url, bio
- Auth: require valid token, users can only update own profile
- Return 200 with updated user on success
- Return 400 with validation errors on invalid input
- Return 401/403 for auth failures
- Follow existing API patterns in the codebase

Workflow:
1. /plan the endpoint structure, middleware chain, and validation logic
2. /tdd — write table-driven tests for success, validation failure, auth failure, not-found
3. Implement following existing handler patterns
4. /go-review
5. /verify — run full test suite, confirm no regressions

Do not:
- Modify existing endpoints
- Change the database schema (use existing user table)
- Add new dependencies without checking existing ones first (use search-first skill)
```

### 例3: EPICプロジェクト

**ユーザー入力:**
```
Migrate our monolith to microservices
```

**最適化プロンプト（フル）:**
```
Use the blueprint skill to plan: "Migrate monolith to microservices architecture"

Before executing, answer these questions in the blueprint:
1. Which domain boundaries exist in the current monolith?
2. Which service should be extracted first (lowest coupling)?
3. Communication pattern: REST APIs, gRPC, or event-driven (Kafka/RabbitMQ)?
4. Database strategy: shared DB initially or database-per-service from start?
5. Deployment target: Kubernetes, Docker Compose, or serverless?

The blueprint should produce phases like:
- Phase 1: Identify service boundaries and create domain map
- Phase 2: Set up infrastructure (API gateway, service mesh, CI/CD per service)
- Phase 3: Extract first service (strangler fig pattern)
- Phase 4: Verify with integration tests, then extract next service
- Phase N: Decommission monolith

Each phase = 1 PR, with /verify gates between phases.
Use /save-session between phases. Use /resume-session to continue.
Use git worktrees for parallel service extraction when dependencies allow.

Recommended: Opus 4.6 for blueprint planning, Sonnet 4.6 for phase execution.
```

---

## 関連コンポーネント

| コンポーネント | 参照タイミング |
|-----------|------------------|
| `configure-ecc` | ユーザーがまだECCをセットアップしていない場合 |
| `skill-stocktake` | どのコンポーネントがインストールされているかの監査（ハードコードされたカタログの代わりに使用） |
| `search-first` | 最適化プロンプトのリサーチフェーズ |
| `blueprint` | EPICスコープの最適化プロンプト（コマンドではなくスキルとして呼び出し） |
| `strategic-compact` | 長時間セッションのコンテキスト管理 |
| `cost-aware-llm-pipeline` | トークン最適化の推奨 |
