# Claude Code Agent Format

[Everything Claude Code (ECC)](https://github.com/affaan-m/everything-claude-code) v1.9.0 を、グローバルプラグインインストールなしに任意のプロジェクトへ組み込めるようにしたローカル配布テンプレートです。

`projedt-root/` 配下の内容をプロジェクトルートにコピーするだけで、28 エージェント・60 コマンド・125 スキル・65 ルール・28 フック・113 スクリプトがすべて日本語で利用可能になります。

## クイックスタート

```bash
# 1. projedt-root/ の中身をプロジェクトルートへコピー
cp -r projedt-root/. /path/to/your-project/

# 2. フックスクリプトの依存関係をインストール
cd /path/to/your-project
npm install
```

これだけで Claude Code / Cursor が自動的にエージェント・コマンド・スキル・ルール・フックを認識します。

---

## ディレクトリ構造

```
projedt-root/
├── CLAUDE.md                  # Claude Code 向けプロジェクトガイダンス
├── AGENTS.md                  # エージェントオーケストレーション指示書
├── CONTRIBUTING.md            # コントリビューションガイド
├── TROUBLESHOOTING.md         # トラブルシューティングガイド
├── package.json               # フックスクリプトの依存管理
├── package-lock.json
│
├── agents/                    # 28 の特化型サブエージェント定義
│
├── scripts/                   # 113 のフック実行スクリプト（Node.js）
│   ├── hooks/                 #   29 のフックハンドラー
│   ├── lib/                   #   50 の共有ユーティリティ
│   ├── ci/                    #   CI バリデーション
│   ├── codex/                 #   Codex 連携
│   └── ...
│
└── .claude/
    ├── settings.json          # フック設定（hooks.json 相当）
    ├── commands/              # 60 のスラッシュコマンド
    ├── skills/                # 125 のスキル定義（各 SKILL.md）
    ├── rules/                 # 65 のルール（共通 + 11 言語別）
    └── hooks/
        └── README.md          # フックの仕組みと一覧
```

---

## ルートファイル (4 ファイル)

| ファイル | 概要 |
|---|---|
| `CLAUDE.md` | Claude Code がプロジェクトを開いた際に最初に読み込む指示ファイル。プロジェクト構造、テスト実行方法、アーキテクチャ概要、コーディング規約を定義 |
| `AGENTS.md` | エージェントの振る舞いを決定する上位指示書。基本原則、利用可能エージェント一覧、オーケストレーション方針、セキュリティガイドライン、テスト要件を網羅 |
| `CONTRIBUTING.md` | コントリビューションガイド。スキル・エージェント・フック・コマンドの追加方法、テンプレート、PR プロセスを説明 |
| `TROUBLESHOOTING.md` | トラブルシューティングガイド。コンテキストオーバーフロー、フックエラー、インストール問題、パフォーマンス問題の解決策 |

---

## agents/ — 28 エージェント

YAML フロントマター付き Markdown で定義された特化型サブエージェント。Claude Code の Task ツール経由で自動委譲されます。

| ファイル | 概要 | モデル |
|---|---|---|
| `architect.md` | ソフトウェアアーキテクチャの専門家。システム設計、スケーラビリティ、技術的意思決定を担当 | opus |
| `build-error-resolver.md` | ビルドおよび TypeScript エラー解決の専門家。最小限の差分でビルド/型エラーのみを修正 | sonnet |
| `chief-of-staff.md` | メール・Slack・LINE・Messenger をトリアージするコミュニケーションチーフオブスタッフ。4 階層に分類し返信を下書き | opus |
| `code-reviewer.md` | コード品質・セキュリティ・保守性をプロアクティブにレビュー。すべてのコード変更に必須 | sonnet |
| `cpp-build-resolver.md` | C++ ビルド・CMake・コンパイルエラーを最小限の変更で修正 | sonnet |
| `cpp-reviewer.md` | メモリ安全性・モダン C++ イディオム・並行処理・パフォーマンスを専門とするレビュアー | sonnet |
| `database-reviewer.md` | PostgreSQL/Supabase 専門。クエリ最適化・スキーマ設計・セキュリティ・パフォーマンス | sonnet |
| `doc-updater.md` | ドキュメントおよびコードマップの更新スペシャリスト | haiku |
| `docs-lookup.md` | Context7 MCP を使用して最新のライブラリ/API ドキュメントを取得し回答 | sonnet |
| `e2e-runner.md` | Vercel Agent Browser + Playwright による E2E テストの生成・保守・実行 | sonnet |
| `flutter-reviewer.md` | Flutter/Dart コードレビュアー。ウィジェット・状態管理・アクセシビリティ・クリーンアーキテクチャ | sonnet |
| `go-build-resolver.md` | Go ビルド・vet・コンパイルエラーを最小限の変更で修正 | sonnet |
| `go-reviewer.md` | 慣用的な Go・並行処理パターン・エラーハンドリング・パフォーマンスのレビュアー | sonnet |
| `harness-optimizer.md` | ローカルエージェントハーネス設定の信頼性・コスト・スループットを分析・改善 | sonnet |
| `java-build-resolver.md` | Java/Maven/Gradle のビルド・コンパイル・依存関係エラーを修正 | sonnet |
| `java-reviewer.md` | Java/Spring Boot のレイヤードアーキテクチャ・JPA・セキュリティ・並行処理レビュアー | sonnet |
| `kotlin-build-resolver.md` | Kotlin/Gradle のビルド・コンパイル・依存関係エラーを修正 | sonnet |
| `kotlin-reviewer.md` | Kotlin/Android/KMP の慣用パターン・コルーチン安全性・Compose レビュアー | sonnet |
| `loop-operator.md` | 自律エージェントループの運用・進捗監視・停滞時の安全な介入 | sonnet |
| `planner.md` | 複雑な機能やリファクタリングの計画立案スペシャリスト | opus |
| `python-reviewer.md` | PEP 8・Pythonic イディオム・型ヒント・セキュリティ・パフォーマンスのレビュアー | sonnet |
| `pytorch-build-resolver.md` | PyTorch ランタイム・CUDA・トレーニングエラーを修正（テンソル形状・デバイス・勾配等） | sonnet |
| `refactor-cleaner.md` | デッドコードのクリーンアップと統合。knip・depcheck・ts-prune で特定し安全に除去 | sonnet |
| `rust-build-resolver.md` | Rust ビルド・ボローチェッカー・Cargo.toml の問題を修正 | sonnet |
| `rust-reviewer.md` | 所有権・ライフタイム・エラーハンドリング・unsafe・慣用パターンのレビュアー | sonnet |
| `security-reviewer.md` | セキュリティ脆弱性の検出と修復。OWASP Top 10・シークレット・SSRF・インジェクション | sonnet |
| `tdd-guide.md` | テストファースト手法を強制する TDD スペシャリスト。80%以上のカバレッジ確保 | sonnet |
| `typescript-reviewer.md` | 型安全性・非同期の正確性・Node/Web セキュリティ・慣用パターンのレビュアー | sonnet |

---

## .claude/commands/ — 60 コマンド

`/command-name` で呼び出すスラッシュコマンド。

| ファイル | 概要 |
|---|---|
| `aside.md` | 現在のタスクを中断せずにちょっとした質問にすぐ答える。回答後は自動的に作業を再開 |
| `build-fix.md` | ビルドエラーの修正ワークフロー |
| `checkpoint.md` | チェックポイントの作成・検証・一覧表示 |
| `claw.md` | NanoClaw v2 を起動。ECC のゼロ依存 REPL。モデルルーティング・スキルホットロード・ブランチ・メトリクス対応 |
| `code-review.md` | 未コミット変更のセキュリティと品質レビュー |
| `context-budget.md` | エージェント・スキル・MCP・ルール全体のコンテキストウィンドウ使用量を分析し最適化の機会を発見 |
| `cpp-build.md` | C++ ビルドエラー・CMake・リンカーの問題を段階的に修正。cpp-build-resolver エージェントを呼び出し |
| `cpp-review.md` | メモリ安全性・モダン C++・並行性・セキュリティの包括的 C++ コードレビュー |
| `cpp-test.md` | C++ の TDD ワークフロー。GoogleTest テストを先に書き、gcov/lcov でカバレッジ検証 |
| `devfleet.md` | Claude DevFleet で並列エージェントをオーケストレーション。分離ワークツリーでディスパッチ・進捗監視・レポート読み取り |
| `docs.md` | Context7 経由でライブラリ/トピックの最新ドキュメントを検索 |
| `e2e.md` | Playwright で E2E テストを生成・実行。スクリーンショット/動画/トレースのキャプチャ・アップロード |
| `eval.md` | 評価駆動開発（EDD）ワークフローの管理（定義・チェック・レポート・一覧） |
| `evolve.md` | インスティンクトを分析し、進化した構造（スキル/コマンド/エージェント）を提案・生成 |
| `go-build.md` | Go ビルドエラー・go vet・リンター問題を段階的修正。go-build-resolver エージェントを呼び出し |
| `go-review.md` | 慣用的 Go パターン・並行性安全性・エラーハンドリング・セキュリティの包括的レビュー |
| `go-test.md` | Go の TDD ワークフロー。テーブル駆動テストを先に書き、go test -cover で 80%以上を検証 |
| `gradle-build.md` | Android/KMP プロジェクトの Gradle ビルドエラー修正 |
| `harness-audit.md` | リポジトリハーネスの決定論的監査とスコアカード（hooks/skills/commands/agents 等） |
| `instinct-export.md` | プロジェクト/グローバルスコープからインスティンクトをファイルにエクスポート |
| `instinct-import.md` | ファイルまたは URL からインスティンクトをインポート |
| `instinct-status.md` | 学習済みインスティンクト（プロジェクト＋グローバル）を信頼度とともに表示 |
| `kotlin-build.md` | Kotlin/Gradle のビルドエラー・コンパイラ警告・依存関係問題を修正 |
| `kotlin-review.md` | Kotlin の慣用パターン・null 安全性・コルーチン安全性・セキュリティの包括的レビュー |
| `kotlin-test.md` | Kotlin の TDD ワークフロー。Kotest テストを先に書き、Kover で 80%以上を検証 |
| `learn.md` | セッションからパターンを抽出して学習スキルとして保存 |
| `learn-eval.md` | セッションからパターンを抽出し品質を自己評価、適切な保存場所（グローバル vs プロジェクト）を決定 |
| `loop-start.md` | 自律エージェントループの開始 |
| `loop-status.md` | 実行中ループの状態確認 |
| `model-route.md` | タスクの複雑度と予算に基づくモデルティア推奨 |
| `multi-backend.md` | マルチモデル協調によるバックエンド特化開発 |
| `multi-execute.md` | マルチモデル協調実行（プラン→実装→監査の一連フロー） |
| `multi-frontend.md` | マルチモデル協調によるフロントエンド特化開発 |
| `multi-plan.md` | マルチモデル協調プランニング（プラン生成のみ、本番コードは変更しない） |
| `multi-workflow.md` | マルチモデル協調の 6 フェーズ開発ワークフロー |
| `orchestrate.md` | 逐次および tmux/worktree オーケストレーションガイダンス |
| `plan.md` | 要件を再確認しリスクを評価、ステップバイステップの実装プランを作成。ユーザー確認を待つ |
| `pm2.md` | プロジェクト分析と PM2 サービス設定・コマンドの生成 |
| `projects.md` | 既知のプロジェクトとインスティンクト統計を一覧表示 |
| `promote.md` | プロジェクトスコープのインスティンクトをグローバルスコープにプロモート |
| `prompt-optimize.md` | ドラフトプロンプトを分析し ECC 強化版を出力。タスクは実行せずアドバイザリー分析のみ |
| `prune.md` | 30 日以上前の昇格されなかった保留中インスティンクトを削除 |
| `python-review.md` | PEP 8・型ヒント・セキュリティ・Pythonic イディオムの包括的 Python コードレビュー |
| `quality-gate.md` | 品質ゲートチェックの実行 |
| `refactor-clean.md` | デッドコードの検出と削除 |
| `resume-session.md` | ~/.claude/session-data/ から最新セッションをロードし前回の作業を再開 |
| `rules-distill.md` | スキルをスキャンし横断的な原則を抽出してルールに蒸留 |
| `rust-build.md` | Rust ビルドエラー・借用チェッカー・依存関係問題を修正。rust-build-resolver を呼び出し |
| `rust-review.md` | 所有権・ライフタイム・エラーハンドリング・unsafe・慣用パターンの包括的 Rust レビュー |
| `rust-test.md` | Rust の TDD ワークフロー。テストを先に書き、cargo-llvm-cov で 80%以上を検証 |
| `save-session.md` | 現在のセッション状態を日付付きファイルに保存し将来のセッションで再開可能に |
| `sessions.md` | セッション履歴・エイリアス・メタデータの管理 |
| `setup-pm.md` | お好みのパッケージマネージャー（npm/pnpm/yarn/bun）を設定 |
| `skill-create.md` | ローカル git 履歴からコーディングパターンを抽出し SKILL.md を生成 |
| `skill-health.md` | スキルポートフォリオのヘルスダッシュボードをチャートと分析とともに表示 |
| `tdd.md` | TDD ワークフローを強制。テストを先に生成し最小限のコードを実装。80%以上のカバレッジ確保 |
| `test-coverage.md` | カバレッジの分析・ギャップ特定・不足テスト生成（80% 目標） |
| `update-codemaps.md` | コードマップの更新 |
| `update-docs.md` | ドキュメントの更新 |
| `verify.md` | 包括的な検証ループの実行 |

---

## .claude/skills/ — 125 スキル

各 `skills/<name>/SKILL.md` に定義されたドメイン知識・ワークフロー定義。Claude が文脈に応じて自動的に読み込みます。

| ディレクトリ | 概要 |
|---|---|
| `agent-eval` | コーディングエージェント（Claude Code、Aider、Codex 等）のカスタムタスクによる直接比較。合格率・コスト・時間・一貫性メトリクスを測定 |
| `agent-harness-construction` | AI エージェントのアクションスペース・ツール定義・オブザベーションフォーマットを設計し完了率を向上 |
| `agentic-engineering` | 評価ファーストの実行・分解・コスト考慮型モデルルーティングによるエージェンティックエンジニア運用 |
| `ai-first-engineering` | AI エージェントが実装出力の大部分を生成するチームのエンジニアリング運用モデル |
| `ai-regression-testing` | AI 支援開発のリグレッションテスト戦略。サンドボックスモード API テスト、自動バグチェック、AI の盲点検出 |
| `android-clean-architecture` | Android/KMP のクリーンアーキテクチャパターン。モジュール構成・依存関係ルール・UseCase・Repository |
| `api-design` | REST API 設計パターン。リソース命名・ステータスコード・ページネーション・フィルタリング・エラーレスポンス・バージョニング |
| `architecture-decision-records` | セッション中のアーキテクチャ意思決定を構造化 ADR として記録。意思決定の自動検出・コンテキスト・代替案・根拠を保存 |
| `article-writing` | 独自の文体で記事・ガイド・ブログ・チュートリアル・ニュースレターを執筆 |
| `autonomous-loops` | 自律型 Claude Code ループのパターン。シーケンシャルパイプラインから RFC 駆動マルチエージェント DAG まで |
| `backend-patterns` | バックエンドアーキテクチャ・API 設計・データベース最適化。Node.js・Express・Next.js API ルート |
| `benchmark` | パフォーマンスベースラインとリグレッション検出（Core Web Vitals・バンドルサイズ・デプロイ前後比較等） |
| `blueprint` | 一行の目的をマルチセッション構築計画に変換。自己完結型コンテキストブリーフ・敵対的レビューゲート・依存関係グラフ |
| `browser-qa` | ブラウザベースの QA テスト |
| `bun-runtime` | Bun をランタイム・パッケージマネージャー・バンドラー・テストランナーとして使用。Bun vs Node の選択基準 |
| `canary-watch` | デプロイ後モニタリング（HTTP・コンソール・ネットワーク・パフォーマンスのリグレッション検知） |
| `carrier-relationship-management` | キャリアポートフォリオ管理・運賃交渉・パフォーマンス追跡・貨物配分の専門知識 |
| `claude-api` | Anthropic Claude API パターン（Python/TypeScript）。Messages API・ストリーミング・ツール使用・ビジョン・拡張思考・バッチ・プロンプトキャッシング・Claude Agent SDK |
| `claude-devfleet` | Claude DevFleet によるマルチエージェントタスクオーケストレーション。分離ワークツリーで並列ディスパッチ |
| `click-path-audit` | ユーザー向けボタンの完全なステート変更シーケンスをトレースし互いに打ち消し合うバグを発見 |
| `clickhouse-io` | ClickHouse パターン・クエリ最適化・アナリティクス・高パフォーマンス分析ワークロード |
| `codebase-onboarding` | 未知のコードベースを分析しオンボーディングガイドを生成。アーキテクチャマップ・エントリポイント・スターター CLAUDE.md |
| `coding-standards` | TypeScript・JavaScript・React・Node.js の汎用コーディング規約とベストプラクティス |
| `compose-multiplatform-patterns` | KMP の Compose Multiplatform/Jetpack Compose パターン。ステート管理・ナビゲーション・テーマ・パフォーマンス |
| `configure-ecc` | ECC インタラクティブインストーラー。スキルとルールの選択・インストールをガイド |
| `content-engine` | X・LinkedIn・TikTok・YouTube・ニュースレターのプラットフォームネイティブコンテンツシステム |
| `content-hash-cache-pattern` | SHA-256 コンテンツハッシュによるファイル処理結果キャッシュ。パス非依存・自動無効化 |
| `context-budget` | コンテキストウィンドウ消費の監査。肥大化・冗長コンポーネント特定・トークン節約推奨 |
| `continuous-agent-loop` | 品質ゲート・評価・リカバリー制御付きの継続的自律エージェントループパターン |
| `continuous-learning` | セッションから再利用可能なパターンを自動抽出し学習済みスキルとして保存 |
| `continuous-learning-v2` | インスティンクトベースの学習システム。フック経由でセッション観察・信頼度スコアリング・スキル/コマンド/エージェントへの進化。v2.1 でプロジェクトスコープ対応 |
| `cost-aware-llm-pipeline` | LLM API コスト最適化。タスク複雑度によるモデルルーティング・予算追跡・リトライロジック・プロンプトキャッシュ |
| `cpp-coding-standards` | C++ Core Guidelines に基づくコーディング規約。モダンで安全かつイディオマティックなプラクティス |
| `cpp-testing` | C++ テストの作成・更新・修正。GoogleTest/CTest 設定・カバレッジ/サニタイザー |
| `crosspost` | X・LinkedIn・Threads・Bluesky のマルチプラットフォームコンテンツ配信。同一コンテンツの投稿禁止 |
| `customs-trade-compliance` | 税関書類・関税分類・関税最適化・制限当事者スクリーニング・規制コンプライアンス |
| `data-scraper-agent` | 公開ソース向け AI 駆動データ収集エージェント構築。Gemini Flash で強化、GitHub Actions で無料稼働 |
| `database-migrations` | DB マイグレーションベストプラクティス。Prisma・Drizzle・Kysely・Django・TypeORM・golang-migrate 対応 |
| `deep-research` | firecrawl と exa MCP によるマルチソースディープリサーチ。ソース帰属付き引用レポート |
| `deployment-patterns` | デプロイメントワークフロー・CI/CD パイプライン・Docker コンテナ化・ヘルスチェック・ロールバック |
| `design-system` | デザインシステムの生成とビジュアル監査 |
| `django-patterns` | Django アーキテクチャ・DRF REST API・ORM ベストプラクティス・キャッシュ・シグナル・ミドルウェア |
| `django-security` | Django セキュリティ。認証・認可・CSRF・SQL インジェクション・XSS 防止・セキュアデプロイ |
| `django-tdd` | pytest-django による Django テスト戦略・TDD・factory_boy・モック・カバレッジ |
| `django-verification` | Django プロジェクトの検証ループ。マイグレーション・リンティング・カバレッジ・セキュリティスキャン |
| `dmux-workflows` | dmux（AI エージェント用 tmux ペインマネージャ）によるマルチエージェントオーケストレーション |
| `docker-patterns` | Docker/Docker Compose パターン。コンテナセキュリティ・ネットワーキング・ボリューム・マルチサービス |
| `documentation-lookup` | Context7 MCP で最新のライブラリ/フレームワークドキュメントを取得。API リファレンス・コード例 |
| `e2e-testing` | Playwright E2E テストパターン・Page Object モデル・CI/CD 統合・フレーキーテスト戦略 |
| `energy-procurement` | 電力・ガス調達・料金最適化・デマンドチャージ管理・再エネ PPA 評価 |
| `enterprise-agent-ops` | オブザーバビリティ・セキュリティ境界・ライフサイクル管理付き長期稼働エージェント運用 |
| `eval-harness` | 評価駆動開発（EDD）原則を実装する正式な評価フレームワーク |
| `exa-search` | Exa MCP によるニューラル検索。ウェブ・コード・企業リサーチ・人物検索 |
| `fal-ai-media` | fal.ai MCP による統合メディア生成。画像（Nano Banana）・動画（Seedance, Kling, Veo 3）・音声（CSM-1B, ThinkSound） |
| `flutter-dart-code-review` | Flutter/Dart コードレビューチェックリスト。BLoC・Riverpod・Provider・GetX・MobX・Signals 対応 |
| `foundation-models-on-device` | Apple FoundationModels によるオンデバイス LLM。テキスト生成・@Generable・ツール呼び出し・iOS 26+ |
| `frontend-patterns` | React・Next.js・状態管理・パフォーマンス最適化・UI ベストプラクティス |
| `frontend-slides` | アニメーション豊かな HTML プレゼンテーション作成。PPT/PPTX の Web 変換にも対応 |
| `golang-patterns` | イディオマティックな Go パターン・ベストプラクティス・規約 |
| `golang-testing` | Go テストパターン。テーブル駆動テスト・サブテスト・ベンチマーク・ファジング・カバレッジ |
| `inventory-demand-planning` | 需要予測・安全在庫最適化・補充計画・プロモーション効果推定 |
| `investor-materials` | ピッチデッキ・投資家メモ・財務モデル・資金調達資料の作成と更新 |
| `investor-outreach` | 資金調達向けコールドメール・フォローアップ・投資家コミュニケーションの下書き |
| `iterative-retrieval` | サブエージェントのコンテキスト問題を解決する段階的コンテキスト取得パターン |
| `java-coding-standards` | Spring Boot サービスのコーディング規約。命名・イミュータビリティ・Optional・ストリーム・例外 |
| `jpa-patterns` | JPA/Hibernate パターン。エンティティ設計・リレーションシップ・クエリ最適化・トランザクション・監査 |
| `kotlin-coroutines-flows` | Kotlin コルーチンと Flow パターン。構造化並行処理・StateFlow・エラーハンドリング・テスト |
| `kotlin-exposed-patterns` | JetBrains Exposed ORM パターン。DSL クエリ・DAO・HikariCP・Flyway マイグレーション |
| `kotlin-ktor-patterns` | Ktor サーバーパターン。ルーティング DSL・プラグイン・認証・Koin DI・WebSocket・テスト |
| `kotlin-patterns` | イディオマティックな Kotlin パターン。コルーチン・null 安全・DSL ビルダー |
| `kotlin-testing` | Kotlin テストパターン。Kotest・MockK・コルーチンテスト・プロパティベーステスト・Kover |
| `laravel-patterns` | Laravel アーキテクチャ。ルーティング・Eloquent ORM・サービスレイヤー・キュー・イベント・キャッシュ |
| `laravel-security` | Laravel セキュリティ。認証/認可・バリデーション・CSRF・マスアサインメント・レート制限 |
| `laravel-tdd` | PHPUnit/Pest による Laravel TDD。ファクトリー・データベーステスト・フェイク・カバレッジ |
| `laravel-verification` | Laravel プロジェクトの検証ループ。環境チェック・リンティング・静的解析・セキュリティスキャン |
| `liquid-glass-design` | iOS 26 Liquid Glass デザインシステム。ブラー・反射・インタラクティブモーフィング（SwiftUI/UIKit/WidgetKit） |
| `logistics-exception-management` | 貨物例外処理・出荷遅延・損傷・紛失・運送業者紛争の処理 |
| `market-research` | 市場調査・競合分析・投資家デューデリジェンス・業界インテリジェンス |
| `mcp-server-patterns` | Node/TypeScript SDK による MCP サーバー構築。ツール・リソース・Zod バリデーション・stdio vs Streamable HTTP |
| `nanoclaw-repl` | NanoClaw v2 の操作と拡張。ECC のゼロ依存セッション対応 REPL |
| `nextjs-turbopack` | Next.js 16+ と Turbopack。インクリメンタルバンドル・FS キャッシュ・Turbopack vs webpack の使い分け |
| `nutrient-document-processing` | Nutrient DWS API によるドキュメント処理・変換・OCR・抽出・墨消し・署名・フォーム入力 |
| `nuxt4-patterns` | Nuxt 4 パターン。ハイドレーション安全性・遅延読み込み・useFetch/useAsyncData による SSR 安全データフェッチ |
| `perl-patterns` | モダン Perl 5.36+ のイディオム・ベストプラクティス・規約 |
| `perl-security` | Perl セキュリティ。テイントモード・入力バリデーション・DBI パラメータ化クエリ・XSS/SQLi/CSRF 防止 |
| `perl-testing` | Perl テストパターン。Test2::V0・Test::More・prove ランナー・Devel::Cover カバレッジ |
| `plankton-code-quality` | Plankton による書き込み時コード品質強制。フック経由の自動フォーマット・リンティング・Claude 修正 |
| `postgres-patterns` | PostgreSQL パターン。クエリ最適化・スキーマ設計・インデックス・セキュリティ。Supabase ベストプラクティス |
| `product-lens` | プロダクトレンズ |
| `production-scheduling` | 生産スケジューリング・ジョブシーケンシング・ラインバランシング・段取り替え最適化・ボトルネック解消 |
| `project-guidelines-example` | 実際の本番アプリに基づくプロジェクト固有スキルテンプレートの例 |
| `prompt-optimizer` | ドラフトプロンプトを分析し ECC コンポーネントにマッチングして最適化版を出力。アドバイザリー専用 |
| `python-patterns` | Pythonic イディオム・PEP 8・型ヒント・ベストプラクティス |
| `python-testing` | pytest による Python テスト戦略。TDD・フィクスチャ・モック・パラメトライズ・カバレッジ |
| `pytorch-patterns` | PyTorch ディープラーニングパターン。トレーニングパイプライン・モデルアーキテクチャ・データローディング |
| `quality-nonconformance` | 品質管理・不適合調査・根本原因分析・是正措置・サプライヤー品質管理（FDA/IATF 16949/AS9100） |
| `ralphinho-rfc-pipeline` | RFC ドリブンのマルチエージェント DAG 実行パターン。品質ゲート・マージキュー・ワークユニット |
| `regex-vs-llm-structured-text` | 構造化テキスト解析の正規表現 vs LLM 判断フレームワーク |
| `returns-reverse-logistics` | 返品承認・受入検査・処分判断・返金処理・不正検出・保証請求管理 |
| `rules-distill` | スキルから横断的原則を抽出しルールに蒸留。既存ルールへの追記・修正・新規作成 |
| `rust-patterns` | イディオマティックな Rust パターン。所有権・エラーハンドリング・トレイト・並行処理 |
| `rust-testing` | Rust テストパターン。ユニット・統合・非同期・プロパティベース・モック・カバレッジ |
| `safety-guard` | セーフティガード |
| `santa-method` | マルチエージェント対抗検証と収束ループ。2 つの独立レビューエージェントが両方パスで出力リリース |
| `search-first` | コーディング前リサーチワークフロー。カスタムコードを書く前に既存ツール・ライブラリ・パターンを検索 |
| `security-review` | 包括的セキュリティチェックリストとパターン。認証・ユーザー入力・シークレット・API エンドポイント・決済 |
| `security-scan` | AgentShield で Claude Code 設定（.claude/）のセキュリティ脆弱性・設定ミス・インジェクションリスクをスキャン |
| `skill-comply` | スキル・ルール・エージェント定義の遵守状況を可視化。3 段階のプロンプト厳密度でコンプライアンスレートをレポート |
| `skill-stocktake` | スキルとコマンドの品質監査。クイックスキャン（変更分のみ）とフルストックテイクモード |
| `springboot-patterns` | Spring Boot アーキテクチャ・REST API・レイヤードサービス・データアクセス・キャッシング・非同期処理 |
| `springboot-security` | Spring Security ベストプラクティス。認証/認可・バリデーション・CSRF・シークレット・レート制限 |
| `springboot-tdd` | JUnit 5・Mockito・MockMvc・Testcontainers・JaCoCo による Spring Boot TDD |
| `springboot-verification` | Spring Boot の検証ループ。ビルド・静的解析・カバレッジ・セキュリティスキャン・差分レビュー |
| `strategic-compact` | 論理的な間隔での手動コンテキストコンパクション提案。任意の自動コンパクションではなくフェーズ間でコンテキスト保持 |
| `swift-actor-persistence` | Swift アクターによるスレッドセーフデータ永続化。インメモリキャッシュ + ファイルバックドストレージ |
| `swift-concurrency-6-2` | Swift 6.2 Approachable Concurrency。シングルスレッドデフォルト・@concurrent・分離適合 |
| `swift-protocol-di-testing` | プロトコルベース DI によるテスト可能な Swift コード。ファイルシステム・ネットワーク・外部 API のモック |
| `swiftui-patterns` | SwiftUI アーキテクチャ。@Observable 状態管理・ビュー合成・ナビゲーション・パフォーマンス最適化 |
| `tdd-workflow` | TDD ワークフロー。ユニット・インテグレーション・E2E テストで 80%以上のカバレッジ |
| `team-builder` | 並列チーム構成とディスパッチのインタラクティブエージェントピッカー |
| `verification-loop` | Claude Code セッションの包括的検証システム |
| `video-editing` | AI 支援動画編集ワークフロー。FFmpeg・Remotion・ElevenLabs・fal.ai・Descript/CapCut |
| `videodb` | 動画/音声の認識・理解・操作。取り込み・インデックス構築・タイムライン編集・メディア生成・リアルタイムアラート |
| `visa-doc-translate` | ビザ申請書類（画像）を英語に翻訳しバイリンガル PDF を作成 |
| `x-api` | X/Twitter API インテグレーション。ツイート投稿・スレッド・タイムライン・検索・アナリティクス |

---

## .claude/rules/ — 65 ルール

Claude が常時参照するガイドライン。共通ルール + 11 言語別ディレクトリ。

### README

| ファイル | 概要 |
|---|---|
| `README.md` | ルールの構成・各ルールカテゴリの説明・適用方法 |

### common/ — 共通ルール (9 ファイル)

| ファイル | 概要 |
|---|---|
| `agents.md` | エージェントオーケストレーション — どのエージェントをいつ呼び出すかのルール |
| `coding-style.md` | コーディングスタイル — 不変性・ファイル構成・エラーハンドリング・入力バリデーション |
| `development-workflow.md` | 開発ワークフロー — 計画→TDD→レビュー→コミットの流れ |
| `git-workflow.md` | Git ワークフロー — Conventional Commits・PR プロセス |
| `hooks.md` | フックシステム — フック設定の規約と利用方法 |
| `patterns.md` | 共通パターン — API レスポンス形式・リポジトリパターン・スケルトンプロジェクト |
| `performance.md` | パフォーマンス最適化 — コンテキスト管理・ビルドトラブルシューティング |
| `security.md` | セキュリティガイドライン — シークレット管理・入力バリデーション・脆弱性対策 |
| `testing.md` | テスト要件 — 80%カバレッジ・TDD ワークフロー・テストタイプ |

### 言語別ルール (各 5 ファイル × 11 言語 = 55 ファイル)

各言語ディレクトリには以下の 5 ファイルが含まれます：

| ファイル | 概要 |
|---|---|
| `coding-style.md` | その言語固有のコーディングスタイルと命名規則 |
| `hooks.md` | その言語固有のフック設定と推奨フック |
| `patterns.md` | その言語固有のアーキテクチャ・設計パターン |
| `security.md` | その言語固有のセキュリティベストプラクティス |
| `testing.md` | その言語固有のテスト戦略・フレームワーク・カバレッジ |

**対応言語ディレクトリ：**

| ディレクトリ | 対象言語 |
|---|---|
| `typescript/` | TypeScript / JavaScript |
| `python/` | Python |
| `golang/` | Go |
| `java/` | Java |
| `kotlin/` | Kotlin |
| `rust/` | Rust |
| `swift/` | Swift |
| `cpp/` | C++ |
| `csharp/` | C# |
| `perl/` | Perl |
| `php/` | PHP |

---

## .claude/settings.json — フック設定 (28 フックエントリ)

| フックタイプ | マッチャー | 概要 |
|---|---|---|
| **PreToolUse** | `Bash` | git フックバイパス防止（`block-no-verify`） |
| **PreToolUse** | `Bash` | tmux でのディレクトリベース dev サーバー自動起動 |
| **PreToolUse** | `Bash` | 長時間コマンドに tmux 使用をリマインド |
| **PreToolUse** | `Bash` | git push 前に変更レビューをリマインド |
| **PreToolUse** | `Write` | 非標準ドキュメントファイルへの警告 |
| **PreToolUse** | `Edit\|Write` | 論理的な間隔で手動 `/compact` を提案 |
| **PreToolUse** | `*` | continuous-learning-v2 のツール使用観察キャプチャ（async） |
| **PreToolUse** | `Bash\|Write\|Edit\|MultiEdit` | InsAIts AI セキュリティモニター（オプトイン、`ECC_ENABLE_INSAITS=1`） |
| **PreToolUse** | `Bash\|Write\|Edit\|MultiEdit` | ガバナンスイベントキャプチャ（`ECC_GOVERNANCE_CAPTURE=1`） |
| **PreToolUse** | `Write\|Edit\|MultiEdit` | リンター/フォーマッター設定ファイルの変更をブロック |
| **PreToolUse** | `*` | MCP サーバーヘルスチェック（不健全なら呼び出しブロック） |
| **PreCompact** | `*` | コンテキスト圧縮前に状態を保存 |
| **SessionStart** | `*` | 前回セッションの文脈復元・パッケージマネージャー検出 |
| **PostToolUse** | `Bash` | PR 作成後に URL とレビューコマンドをログ |
| **PostToolUse** | `Bash` | ビルド完了後の分析（async） |
| **PostToolUse** | `Edit\|Write\|MultiEdit` | ファイル編集後の品質ゲートチェック（async） |
| **PostToolUse** | `Edit` | JS/TS ファイルの自動フォーマット（Biome/Prettier 自動検出） |
| **PostToolUse** | `Edit` | .ts/.tsx 編集後の TypeScript 型チェック |
| **PostToolUse** | `Edit` | 編集後の console.log 残存警告 |
| **PostToolUse** | `Bash\|Write\|Edit\|MultiEdit` | ガバナンスイベントのツール出力キャプチャ |
| **PostToolUse** | `*` | continuous-learning-v2 のツール使用結果キャプチャ（async） |
| **PostToolUseFailure** | `*` | 失敗した MCP ツール呼び出しの追跡・不健全サーバーマーク・再接続 |
| **Stop** | `*` | 変更ファイルの console.log チェック |
| **Stop** | `*` | セッション状態の永続化（async） |
| **Stop** | `*` | セッションからパターン抽出・評価（async） |
| **Stop** | `*` | トークン・コストメトリクス追跡（async） |
| **Stop** | `*` | macOS デスクトップ通知（async） |
| **SessionEnd** | `*` | セッション終了ライフサイクルマーカー（async） |

---

## scripts/ — 113 スクリプト

フックから呼び出される Node.js スクリプト群。

### scripts/hooks/ — 29 フックハンドラー

| ファイル | 概要 |
|---|---|
| `auto-tmux-dev.js` | Pre-Bash: dev サーバーを tmux/cmd で実行しハーネスをブロックしない |
| `check-console-log.js` | Stop: 変更 JS/TS ファイルに console.log が残存していれば警告 |
| `check-hook-enabled.js` | 指定フック ID が現在のプロファイルで有効かどうかを出力 |
| `config-protection.js` | リンター/フォーマッター設定ファイルの編集をブロック（ソース修正を誘導） |
| `cost-tracker.js` | セッション使用メトリクスを `~/.claude/metrics/costs.jsonl` に追記 |
| `desktop-notify.js` | Stop: タスクサマリー付きネイティブデスクトップ通知（macOS） |
| `doc-file-warning.js` | PreToolUse Write: 非標準ドキュメントパスへの警告 |
| `evaluate-session.js` | 継続学習 — Stop 時にトランスクリプトからパターンを抽出 |
| `governance-capture.js` | Pre/PostToolUse: ガバナンスイベントをステートストアにキャプチャ |
| `insaits-security-wrapper.js` | Python InsAIts セキュリティモニターへのラッパー |
| `mcp-health-check.js` | ツール使用前と失敗時に MCP サーバーのヘルスを検査 |
| `post-bash-build-complete.js` | Post-Bash: npm/pnpm/yarn build 後の stderr ヒント |
| `post-bash-pr-created.js` | Post-Bash: `gh pr create` 後に PR URL とレビューコマンドを出力 |
| `post-edit-console-warn.js` | Post-Edit: 編集 JS/TS に console.log があれば行番号付き警告 |
| `post-edit-format.js` | Post-Edit: Biome/Prettier で JS/TS を自動フォーマット |
| `post-edit-typecheck.js` | Post-Edit: 最寄りの tsconfig.json で `tsc --noEmit` 実行 |
| `pre-bash-dev-server-block.js` | Pre-Bash（非 Windows）: tmux 外での dev サーバーコマンドをブロック |
| `pre-bash-git-push-reminder.js` | Pre-Bash: `git push` コマンド検出時に stderr リマインダー |
| `pre-bash-tmux-reminder.js` | Pre-Bash: 長時間の install/test/build コマンドに tmux 提案 |
| `pre-compact.js` | PreCompact: コンテキスト圧縮前の状態保存 |
| `pre-write-doc-warn.js` | 後方互換エントリポイント。`doc-file-warning.js` に委譲 |
| `quality-gate.js` | Post-Edit: Biome/Prettier/Go/Python の品質チェック |
| `run-with-flags.js` | ECC フックプロファイルフラグで有効な場合のみスクリプトを実行 |
| `run-with-flags-shell.sh` | シェルスクリプト版のフラグ付き実行ランナー |
| `session-end-marker.js` | パススルーフック: stdin を stdout にエコー |
| `session-end.js` | Stop: セッションサマリーを永続化しクロスセッション継続性を確保 |
| `session-start.js` | SessionStart: 最近のセッション文脈をロードしセッション/スキル状況を報告 |
| `suggest-compact.js` | 論理的な間隔で手動コンパクションを提案 |
| `insaits-security-monitor.py` | InsAIts AI セキュリティモニター本体（Python）。`insaits-security-wrapper.js` から呼び出される |

### scripts/lib/ — 50 共有ユーティリティ

| ファイル | 概要 |
|---|---|
| `agent-compress.js` | エージェント Markdown のロード/圧縮（カタログ/サマリーモード）、フロントマター解析 |
| `hook-flags.js` | フックの有効/無効管理: `ECC_HOOK_PROFILE`, `ECC_DISABLED_HOOKS` |
| `inspection.js` | スキル検査ヒューリスティクスの失敗理由の正規化とクラスタリング |
| `install-executor.js` | マニフェスト/レガシーインストールプランの構築とインストール操作のオーケストレーション |
| `install-lifecycle.js` | ECC インストール状態の発見・修復・アンインストール |
| `install-manifests.js` | マニフェストからインストールコンポーネント・モジュール・プロファイル・ファイル操作を解決 |
| `install-state.js` | `install-state` JSON の読み書き/バリデーション（スキーマ対応） |
| `orchestration-session.js` | tmux/dmux セッション出力からオーケストレーションスナップショットを収集 |
| `package-manager.js` | パッケージマネージャーの検出と選択（npm/pnpm/yarn/bun） |
| `package-manager.d.ts` | パッケージマネージャー検出の型宣言 |
| `project-detect.js` | マーカーファイルからのプロジェクトタイプ・フレームワーク検出 |
| `resolve-ecc-root.js` | ECC プラグイン/ソースルートの解決 |
| `resolve-formatter.js` | post-edit-format/quality-gate 用のフォーマッター/バイナリ解決キャッシュ |
| `session-aliases.js` | セッションエイリアスの CRUD ライブラリ |
| `session-aliases.d.ts` | セッションエイリアスストアの型宣言 |
| `session-manager.js` | セッションファイルの一覧/ロード/管理とレガシーパス対応 |
| `session-manager.d.ts` | セッションファイル CRUD の型宣言 |
| `shell-split.js` | `&&`, `\|\|`, `;`, `&` でシェルコマンドを分割（クォート・エスケープ考慮、リダイレクションは非分割） |
| `tmux-worktree-orchestrator.js` | tmux ベースのマルチワークツリーオーケストレーションの計画/構築/実行 |
| `utils.js` | クロスプラットフォームユーティリティ（FS・パス・git・プラットフォームヘルパー） |
| `utils.d.ts` | ユーティリティの型宣言 |
| `install/apply.js` | インストールプランの適用: ファイルコピーとインストール状態書き込み |
| `install/config.js` | `ecc-install.json` のロードと JSON Schema バリデーション |
| `install/request.js` | インストール CLI 引数とターゲットの解析/正規化 |
| `install/runtime.js` | 正規化されたインストールリクエストをレガシー/マニフェストプランにマッピング |
| `install-targets/antigravity-project.js` | インストールターゲットアダプター: Antigravity プロジェクト（`.agent`） |
| `install-targets/claude-home.js` | インストールターゲットアダプター: Claude ホーム（`~/.claude`） |
| `install-targets/codex-home.js` | インストールターゲットアダプター: Codex ホーム（`~/.codex`） |
| `install-targets/cursor-project.js` | インストールターゲットアダプター: Cursor プロジェクト（`.cursor`） |
| `install-targets/helpers.js` | インストールターゲットアダプターの共有ファクトリ/ヘルパー |
| `install-targets/opencode-home.js` | インストールターゲットアダプター: OpenCode ホーム（`~/.opencode`） |
| `install-targets/registry.js` | インストールターゲットアダプターのレジストリ |
| `session-adapters/canonical-session.js` | 正規セッションレコードの正規化と記録パス |
| `session-adapters/claude-history.js` | アダプター: Claude セッション履歴/ファイル/エイリアスターゲット |
| `session-adapters/dmux-tmux.js` | アダプター: dmux/tmux プランファイルとライブセッションターゲット |
| `session-adapters/registry.js` | セッションアダプターのファクトリ |
| `skill-evolution/dashboard.js` | スキル進化メトリクスのターミナルダッシュボード/スパークライン |
| `skill-evolution/health.js` | スキルヘルスメトリクスの収集と集約 |
| `skill-evolution/index.js` | skill-evolution サブモジュールのバレルエクスポート |
| `skill-evolution/provenance.js` | スキル出自（`.provenance.json`）と curated/learned/imported タイプ |
| `skill-evolution/tracker.js` | スキル実行結果とレビューフィードバックの追跡 |
| `skill-evolution/versioning.js` | スキルバージョンディレクトリと進化ログ追記ヘルパー |
| `skill-improvement/amendify.js` | ヘルスレポートからスキル修正提案（失敗駆動パッチ） |
| `skill-improvement/evaluate.js` | 実行履歴からスキル評価スキャフォールドレコードを構築 |
| `skill-improvement/health.js` | テレメトリバリアントランからスキルヘルスレポートを構築 |
| `skill-improvement/observations.js` | `.claude/ecc/skills` 配下のスキル観察ファイルの読み書き |
| `state-store/index.js` | SQLite ECC ステートストア（sql.js）: パス解決・マイグレーション・クエリ API |
| `state-store/migrations.js` | ECC ステートデータベーステーブルの SQL マイグレーション |
| `state-store/queries.js` | セッション・スキル実行・インストール・ガバナンスのクエリ/更新ヘルパー |
| `state-store/schema.js` | 永続化エンティティの Ajv バリデーション |

### scripts/ ルート — 20 スクリプト

| ファイル | 概要 |
|---|---|
| `claw.js` | NanoClaw REPL CLI |
| `doctor.js` | ECC install-state に対するドリフト・不足ファイルの診断 |
| `ecc.js` | ECC メイン CLI（`npx ecc` エントリポイント） |
| `harness-audit.js` | エージェントハーネス設定の監査 CLI |
| `install-apply.js` | リファクタ済み ECC インストーラーランタイム |
| `install-plan.js` | 選択的インストールプロファイルとモジュールプランの検査（変更なし） |
| `list-installed.js` | 現在のホーム/プロジェクトの ECC `install-state` を検査 |
| `orchestrate-codex-worker.sh` | Bash ワーカー: タスク/ハンドオフファイルから Codex タスクを実行 |
| `orchestrate-worktrees.js` | tmux ワークツリーオーケストレーションプランの構築/実行/マテリアライズ CLI |
| `orchestration-status.js` | オーケストレーションセッションターゲットの検査 CLI |
| `release.sh` | プラグインバージョンのバンプ（package.json・plugin JSON・marketplace・OpenCode） |
| `repair.js` | `install-state` から ECC 管理ファイルを再構築 |
| `session-inspect.js` | セッションの検査 CLI（スキルヘルス・修正提案・評価スキャフォールド） |
| `sessions-cli.js` | SQLite ステートから ECC セッションの一覧/検査 CLI |
| `setup-package-manager.js` | インタラクティブパッケージマネージャーセットアップ CLI |
| `skill-create-output.js` | `/skill-create` 用のターミナル出力フォーマッター |
| `skills-health.js` | スキル進化ヘルスレポートとダッシュボード CLI |
| `status.js` | ECC SQLite ステートのクエリ CLI（セッション・スキル実行・インストール・ガバナンス） |
| `sync-ecc-to-codex.sh` | ECC アセットをローカル Codex CLI に同期 |
| `uninstall.js` | `install-state` で記録された ECC 管理ファイルを削除 |

### scripts/ci/ — 8 CI バリデーション

| ファイル | 概要 |
|---|---|
| `catalog.js` | リポジトリカタログ数を README.md と AGENTS.md に対して検証 |
| `validate-agents.js` | エージェント Markdown に必須フロントマターがあるか検証 |
| `validate-commands.js` | コマンド Markdown とコマンド/エージェント/スキルへの相互参照を検証 |
| `validate-hooks.js` | hooks.json のスキーマとフックエントリルールを検証 |
| `validate-install-manifests.js` | 選択的インストールマニフェストとプロファイル/モジュール関係を検証 |
| `validate-no-personal-paths.js` | 公開ドキュメント/スキル/コマンドにユーザー固有の絶対パスが含まれていないか検証 |
| `validate-rules.js` | ルール Markdown の検証 |
| `validate-skills.js` | `skills/` 配下のキュレーション済みスキルディレクトリを検証 |

### scripts/codex/ — 3 Codex 連携

| ファイル | 概要 |
|---|---|
| `check-codex-global-state.sh` | `~/.codex` 統合のグローバルリグレッションチェック |
| `install-global-git-hooks.sh` | `core.hooksPath` 経由で ECC git セーフティフックをグローバルインストール |
| `merge-mcp-config.js` | ECC 推奨 MCP サーバーを Codex `config.toml` にマージ（追加のみ） |

### scripts/codex-git-hooks/ — 2 Git フック

| ファイル | 概要 |
|---|---|
| `pre-commit` | ECC Codex git hook: 高信頼シークレットを追加するコミットをブロック |
| `pre-push` | ECC Codex git hook: プッシュ前の軽量検証 |

### scripts/codemaps/ — 1 ファイル

| ファイル | 概要 |
|---|---|
| `generate.ts` | コードマップジェネレーター。ツリーをスキャンし `docs/CODEMAPS/` にアーキテクチャマップを書き出し |

---

## .claude/hooks/README.md

フックの仕組み・全フック一覧・マッチャー構文・設定方法を説明するドキュメント。

---

## package.json / package-lock.json

| フィールド | 値 |
|---|---|
| **name** | `ecc-universal` |
| **version** | `1.9.0` |
| **dependencies** | `@iarna/toml` ^2.2.5, `sql.js` ^1.14.1 |
| **devDependencies** | `@eslint/js`, `ajv`, `c8`, `eslint`, `globals`, `markdownlint-cli` |
| **engines** | Node.js >= 18 |

---

## セットアップ手順

### 1. ファイルのコピー

```powershell
# PowerShell
Copy-Item -Path "projedt-root\*" -Destination "C:\path\to\your-project\" -Recurse -Force
Copy-Item -Path "projedt-root\.claude" -Destination "C:\path\to\your-project\.claude" -Recurse -Force
```

```bash
# macOS / Linux
cp -r projedt-root/. /path/to/your-project/
```

### 2. 依存関係のインストール

```bash
cd /path/to/your-project
npm install
```

### 3. 動作確認

Claude Code / Cursor でプロジェクトを開くと以下が自動的に有効になります：

- `CLAUDE.md` / `AGENTS.md` の読み込み
- `.claude/settings.json` のフック
- `.claude/commands/` のスラッシュコマンド
- `.claude/skills/` のスキル（文脈に応じて自動読み込み）
- `.claude/rules/` のルール（常時適用）
- `agents/` のエージェント（Task ツール経由で委譲）

## カスタマイズ

### 不要なコンポーネントの削除

```bash
# 例：Perl と PHP のルールが不要な場合
rm -rf .claude/rules/perl .claude/rules/php
```

### フックの無効化

`.claude/settings.json` 内の不要なフックエントリを削除するか、環境変数 `ECC_HOOK_FLAGS` で制御できます。

### プロジェクト固有の CLAUDE.md

`CLAUDE.md` をプロジェクトの実情に合わせて編集してください。

## メモリー・学習データの保存先

フックにより以下が `~/.claude/` 配下に自動生成されます：

| 保存先 | 内容 |
|---|---|
| `~/.claude/session-data/` | セッションサマリー（次回開始時に文脈復元） |
| `~/.claude/skills/learned/` | `/learn` で抽出されたパターン |
| `~/.claude/homunculus/` | continuous-learning-v2 のインスティンクト（学習データ） |
| `~/.claude/metrics/` | コスト・トークン使用量の記録 |

## 動作要件

- **Node.js** >= 18
- **Claude Code** または **Cursor**
- **npm**（フックスクリプトの依存関係インストール用）

## ライセンス

MIT License - [Everything Claude Code](https://github.com/affaan-m/everything-claude-code)
