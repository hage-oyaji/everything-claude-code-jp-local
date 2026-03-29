---
name: configure-ecc
description: Everything Claude Codeのインタラクティブインストーラー — スキルとルールの選択・インストールをユーザーレベルまたはプロジェクトレベルのディレクトリにガイドし、パスを検証し、オプションでインストール済みファイルを最適化。
origin: ECC
---

# Everything Claude Code（ECC）の設定

Everything Claude Codeプロジェクトのインタラクティブなステップバイステップインストールウィザード。`AskUserQuestion`を使用して、スキルとルールの選択的インストールをガイドし、正確性を検証し、最適化を提供する。

## 発動条件

- ユーザーが「configure ecc」「install ecc」「setup everything claude code」などと発言したとき
- ユーザーがこのプロジェクトからスキルやルールを選択的にインストールしたいとき
- ユーザーが既存のECCインストールを検証または修正したいとき
- ユーザーがインストール済みのスキルやルールをプロジェクトに合わせて最適化したいとき

## 前提条件

このスキルはアクティベーション前にClaude Codeからアクセス可能である必要がある。ブートストラップ方法は2つ:
1. **プラグイン経由**: `/plugin install everything-claude-code` — プラグインがこのスキルを自動的にロード
2. **手動**: このスキルのみを`~/.claude/skills/configure-ecc/SKILL.md`にコピーし、「configure ecc」と発言してアクティベート

---

## ステップ0: ECCリポジトリのクローン

インストール前に、最新のECCソースを`/tmp`にクローンする:

```bash
rm -rf /tmp/everything-claude-code
git clone https://github.com/affaan-m/everything-claude-code.git /tmp/everything-claude-code
```

以降のすべてのコピー操作のソースとして`ECC_ROOT=/tmp/everything-claude-code`を設定する。

クローンが失敗した場合（ネットワーク問題など）、`AskUserQuestion`を使用して既存のECCクローンのローカルパスを提供するようユーザーに問い合わせる。

---

## ステップ1: インストールレベルの選択

`AskUserQuestion`を使用してインストール先を問い合わせる:

```
Question: "Where should ECC components be installed?"
Options:
  - "User-level (~/.claude/)" — "Applies to all your Claude Code projects"
  - "Project-level (.claude/)" — "Applies only to the current project"
  - "Both" — "Common/shared items user-level, project-specific items project-level"
```

選択を`INSTALL_LEVEL`として保存する。ターゲットディレクトリを設定:
- ユーザーレベル: `TARGET=~/.claude`
- プロジェクトレベル: `TARGET=.claude`（現在のプロジェクトルートからの相対パス）
- 両方: `TARGET_USER=~/.claude`、`TARGET_PROJECT=.claude`

ターゲットディレクトリが存在しない場合は作成する:
```bash
mkdir -p $TARGET/skills $TARGET/rules
```

---

## ステップ2: スキルの選択とインストール

### 2a: スコープの選択（コアvsニッチ）

**コア（新規ユーザー推奨）**をデフォルトとする — `.agents/skills/*`に加え、リサーチファーストワークフローのための`skills/search-first/`をコピー。このバンドルはエンジニアリング、評価、検証、セキュリティ、戦略的コンパクション、フロントエンドデザイン、Anthropicクロスファンクショナルスキル（article-writing、content-engine、market-research、frontend-slides）をカバー。

`AskUserQuestion`を使用（単一選択）:
```
Question: "Install core skills only, or include niche/framework packs?"
Options:
  - "Core only (recommended)" — "tdd, e2e, evals, verification, research-first, security, frontend patterns, compacting, cross-functional Anthropic skills"
  - "Core + selected niche" — "Add framework/domain-specific skills after core"
  - "Niche only" — "Skip core, install specific framework/domain skills"
Default: Core only
```

ユーザーがニッチまたはコア + ニッチを選択した場合、以下のカテゴリ選択に進み、選択したニッチスキルのみを含める。

### 2b: スキルカテゴリの選択

以下に7つの選択可能なカテゴリグループがある。後続の詳細確認リストは8カテゴリにわたる45スキルと1つのスタンドアロンテンプレートをカバーする。`AskUserQuestion`で`multiSelect: true`を使用:

```
Question: "Which skill categories do you want to install?"
Options:
  - "Framework & Language" — "Django, Laravel, Spring Boot, Go, Python, Java, Frontend, Backend patterns"
  - "Database" — "PostgreSQL, ClickHouse, JPA/Hibernate patterns"
  - "Workflow & Quality" — "TDD, verification, learning, security review, compaction"
  - "Research & APIs" — "Deep research, Exa search, Claude API patterns"
  - "Social & Content Distribution" — "X/Twitter API, crossposting alongside content-engine"
  - "Media Generation" — "fal.ai image/video/audio alongside VideoDB"
  - "Orchestration" — "dmux multi-agent workflows"
  - "All skills" — "Install every available skill"
```

### 2c: 個別スキルの確認

選択されたカテゴリごとに、以下の完全なスキルリストを表示し、ユーザーに特定のものを確認または選択解除するよう求める。リストが4項目を超える場合、テキストとしてリストを表示し、「Install all listed」オプションと、ユーザーが特定の名前を貼り付けるための「Other」オプション付きの`AskUserQuestion`を使用する。

**カテゴリ: フレームワークと言語（21スキル）**

| スキル | 説明 |
|-------|-------------|
| `backend-patterns` | Node.js/Express/Next.jsのバックエンドアーキテクチャ、API設計、サーバーサイドベストプラクティス |
| `coding-standards` | TypeScript、JavaScript、React、Node.jsの汎用コーディング規約 |
| `django-patterns` | Djangoアーキテクチャ、DRFによるREST API、ORM、キャッシュ、シグナル、ミドルウェア |
| `django-security` | Djangoセキュリティ: 認証、CSRF、SQLインジェクション、XSS防止 |
| `django-tdd` | pytest-djangoによるDjangoテスト、factory_boy、モック、カバレッジ |
| `django-verification` | Django検証ループ: マイグレーション、リンティング、テスト、セキュリティスキャン |
| `laravel-patterns` | Laravelアーキテクチャパターン: ルーティング、コントローラー、Eloquent、キュー、キャッシュ |
| `laravel-security` | Laravelセキュリティ: 認証、ポリシー、CSRF、マスアサインメント、レートリミット |
| `laravel-tdd` | PHPUnitとPestによるLaravelテスト、ファクトリ、フェイク、カバレッジ |
| `laravel-verification` | Laravel検証: リンティング、静的解析、テスト、セキュリティスキャン |
| `frontend-patterns` | React、Next.js、ステート管理、パフォーマンス、UIパターン |
| `frontend-slides` | ゼロ依存HTMLプレゼンテーション、スタイルプレビュー、PPTX-to-Web変換 |
| `golang-patterns` | 堅牢なGoアプリケーションのためのイディオマティックGoパターンと規約 |
| `golang-testing` | Goテスト: テーブル駆動テスト、サブテスト、ベンチマーク、ファジング |
| `java-coding-standards` | Spring Boot向けJavaコーディング規約: 命名、イミュータビリティ、Optional、ストリーム |
| `python-patterns` | Pythonイディオム、PEP 8、型ヒント、ベストプラクティス |
| `python-testing` | pytest、TDD、フィクスチャ、モック、パラメトライズによるPythonテスト |
| `springboot-patterns` | Spring Bootアーキテクチャ、REST API、レイヤードサービス、キャッシュ、非同期 |
| `springboot-security` | Spring Security: 認証/認可、バリデーション、CSRF、シークレット、レートリミット |
| `springboot-tdd` | JUnit 5、Mockito、MockMvc、TestcontainersによるSpring Boot TDD |
| `springboot-verification` | Spring Boot検証: ビルド、静的解析、テスト、セキュリティスキャン |

**カテゴリ: データベース（3スキル）**

| スキル | 説明 |
|-------|-------------|
| `clickhouse-io` | ClickHouseパターン、クエリ最適化、アナリティクス、データエンジニアリング |
| `jpa-patterns` | JPA/Hibernateエンティティ設計、リレーション、クエリ最適化、トランザクション |
| `postgres-patterns` | PostgreSQLクエリ最適化、スキーマ設計、インデックス、セキュリティ |

**カテゴリ: ワークフローと品質（8スキル）**

| スキル | 説明 |
|-------|-------------|
| `continuous-learning` | セッションから再利用可能なパターンを学習済みスキルとして自動抽出 |
| `continuous-learning-v2` | 信頼度スコアリング付きインスティンクトベース学習、スキル/コマンド/エージェントへ進化 |
| `eval-harness` | 評価駆動開発（EDD）のための正式な評価フレームワーク |
| `iterative-retrieval` | サブエージェントコンテキスト問題のための段階的コンテキスト精錬 |
| `security-review` | セキュリティチェックリスト: 認証、入力、シークレット、API、決済機能 |
| `strategic-compact` | 論理的な区切りで手動コンテキストコンパクションを提案 |
| `tdd-workflow` | 80%以上のカバレッジでTDDを強制: ユニット、統合、E2E |
| `verification-loop` | 検証と品質ループパターン |

**カテゴリ: ビジネスとコンテンツ（5スキル）**

| スキル | 説明 |
|-------|-------------|
| `article-writing` | 提供されたボイスを使用した、ノート、例、ソースドキュメントからの長文ライティング |
| `content-engine` | マルチプラットフォームソーシャルコンテンツ、スクリプト、リパーパスワークフロー |
| `market-research` | ソース帰属付き市場、競合、ファンド、テクノロジーリサーチ |
| `investor-materials` | ピッチデッキ、ワンページャー、投資家メモ、財務モデル |
| `investor-outreach` | パーソナライズされた投資家コールドメール、ウォームイントロ、フォローアップ |

**カテゴリ: リサーチとAPI（3スキル）**

| スキル | 説明 |
|-------|-------------|
| `deep-research` | firecrawlとexa MCPを使用した、引用付きレポートによるマルチソースディープリサーチ |
| `exa-search` | Web、コード、企業、人物リサーチのためのExa MCPによるニューラル検索 |
| `claude-api` | Anthropic Claude APIパターン: Messages、ストリーミング、ツール使用、ビジョン、バッチ、Agent SDK |

**カテゴリ: ソーシャルとコンテンツ配信（2スキル）**

| スキル | 説明 |
|-------|-------------|
| `x-api` | 投稿、スレッド、検索、アナリティクスのためのX/Twitter API統合 |
| `crosspost` | プラットフォームネイティブ適応によるマルチプラットフォームコンテンツ配信 |

**カテゴリ: メディア生成（2スキル）**

| スキル | 説明 |
|-------|-------------|
| `fal-ai-media` | fal.ai MCPによる統合AIメディア生成（画像、動画、音声） |
| `video-editing` | 実映像のカット、構成、拡張のためのAI支援動画編集 |

**カテゴリ: オーケストレーション（1スキル）**

| スキル | 説明 |
|-------|-------------|
| `dmux-workflows` | 並列エージェントセッションのためのdmuxによるマルチエージェントオーケストレーション |

**スタンドアロン**

| スキル | 説明 |
|-------|-------------|
| `project-guidelines-example` | プロジェクト固有スキル作成用テンプレート |

### 2d: インストールの実行

選択されたスキルごとに、スキルディレクトリ全体をコピーする:
```bash
cp -r $ECC_ROOT/skills/<skill-name> $TARGET/skills/
```

注: `continuous-learning`と`continuous-learning-v2`には追加ファイル（config.json、フック、スクリプト）がある — SKILL.mdだけでなくディレクトリ全体がコピーされていることを確認する。

---

## ステップ3: ルールの選択とインストール

`AskUserQuestion`で`multiSelect: true`を使用:

```
Question: "Which rule sets do you want to install?"
Options:
  - "Common rules (Recommended)" — "Language-agnostic principles: coding style, git workflow, testing, security, etc. (8 files)"
  - "TypeScript/JavaScript" — "TS/JS patterns, hooks, testing with Playwright (5 files)"
  - "Python" — "Python patterns, pytest, black/ruff formatting (5 files)"
  - "Go" — "Go patterns, table-driven tests, gofmt/staticcheck (5 files)"
```

インストールの実行:
```bash
# Common rules (flat copy into rules/)
cp -r $ECC_ROOT/rules/common/* $TARGET/rules/

# Language-specific rules (flat copy into rules/)
cp -r $ECC_ROOT/rules/typescript/* $TARGET/rules/   # if selected
cp -r $ECC_ROOT/rules/python/* $TARGET/rules/        # if selected
cp -r $ECC_ROOT/rules/golang/* $TARGET/rules/        # if selected
```

**重要**: ユーザーが言語固有ルールを選択しても共通ルールを選択しなかった場合、警告する:
> 「言語固有ルールは共通ルールを拡張します。共通ルールなしでインストールすると、カバレッジが不完全になる可能性があります。共通ルールもインストールしますか？」

---

## ステップ4: インストール後の検証

インストール後、以下の自動チェックを実行する:

### 4a: ファイル存在の検証

すべてのインストール済みファイルをリストし、ターゲットの場所に存在することを確認する:
```bash
ls -la $TARGET/skills/
ls -la $TARGET/rules/
```

### 4b: パス参照の確認

すべてのインストール済み`.md`ファイルのパス参照をスキャンする:
```bash
grep -rn "~/.claude/" $TARGET/skills/ $TARGET/rules/
grep -rn "../common/" $TARGET/rules/
grep -rn "skills/" $TARGET/skills/
```

**プロジェクトレベルインストールの場合**、`~/.claude/`パスへの参照にフラグを立てる:
- スキルが`~/.claude/settings.json`を参照 — これは通常問題ない（settingsは常にユーザーレベル）
- スキルが`~/.claude/skills/`や`~/.claude/rules/`を参照 — プロジェクトレベルのみにインストールされた場合、壊れる可能性あり
- スキルが名前で別のスキルを参照 — 参照されたスキルもインストールされていることを確認

### 4c: スキル間の相互参照の確認

一部のスキルは他のスキルを参照する。以下の依存関係を検証する:
- `django-tdd`は`django-patterns`を参照する可能性あり
- `laravel-tdd`は`laravel-patterns`を参照する可能性あり
- `springboot-tdd`は`springboot-patterns`を参照する可能性あり
- `continuous-learning-v2`は`~/.claude/homunculus/`ディレクトリを参照
- `python-testing`は`python-patterns`を参照する可能性あり
- `golang-testing`は`golang-patterns`を参照する可能性あり
- `crosspost`は`content-engine`と`x-api`を参照
- `deep-research`は`exa-search`を参照（補完的MCPツール）
- `fal-ai-media`は`videodb`を参照（補完的メディアスキル）
- `x-api`は`content-engine`と`crosspost`を参照
- 言語固有ルールは`common/`の対応するルールを参照

### 4d: 問題の報告

発見された問題ごとに以下を報告する:
1. **ファイル**: 問題のある参照を含むファイル
2. **行**: 行番号
3. **問題**: 何が問題か（例: 「~/.claude/skills/python-patternsを参照しているがpython-patternsがインストールされていない」）
4. **推奨修正**: 対処法（例: 「python-patternsスキルをインストール」または「パスを.claude/skills/に更新」）

---

## ステップ5: インストール済みファイルの最適化（オプション）

`AskUserQuestion`を使用:

```
Question: "Would you like to optimize the installed files for your project?"
Options:
  - "Optimize skills" — "Remove irrelevant sections, adjust paths, tailor to your tech stack"
  - "Optimize rules" — "Adjust coverage targets, add project-specific patterns, customize tool configs"
  - "Optimize both" — "Full optimization of all installed files"
  - "Skip" — "Keep everything as-is"
```

### スキルを最適化する場合:
1. インストール済みの各SKILL.mdを読む
2. プロジェクトの技術スタックを問い合わせる（まだ不明な場合）
3. 各スキルについて、関連しないセクションの削除を提案する
4. インストールターゲットのSKILL.mdファイルをインプレースで編集する（ソースリポジトリではない）
5. ステップ4で発見されたパスの問題を修正する

### ルールを最適化する場合:
1. インストール済みの各ルール.mdファイルを読む
2. ユーザーの設定について問い合わせる:
   - テストカバレッジ目標（デフォルト80%）
   - 優先するフォーマットツール
   - Gitワークフロー規約
   - セキュリティ要件
3. インストールターゲットのルールファイルをインプレースで編集する

**重要**: インストールターゲット（`$TARGET/`）のファイルのみを変更する。ソースECCリポジトリ（`$ECC_ROOT/`）のファイルは**絶対に**変更しない。

---

## ステップ6: インストールサマリー

クローンしたリポジトリを`/tmp`からクリーンアップする:

```bash
rm -rf /tmp/everything-claude-code
```

次にサマリーレポートを出力する:

```
## ECC Installation Complete

### Installation Target
- Level: [user-level / project-level / both]
- Path: [target path]

### Skills Installed ([count])
- skill-1, skill-2, skill-3, ...

### Rules Installed ([count])
- common (8 files)
- typescript (5 files)
- ...

### Verification Results
- [count] issues found, [count] fixed
- [list any remaining issues]

### Optimizations Applied
- [list changes made, or "None"]
```

---

## トラブルシューティング

### 「スキルがClaude Codeに認識されない」
- スキルディレクトリに`SKILL.md`ファイルが含まれていることを確認（単なるルーズな.mdファイルではない）
- ユーザーレベルの場合: `~/.claude/skills/<skill-name>/SKILL.md`が存在することを確認
- プロジェクトレベルの場合: `.claude/skills/<skill-name>/SKILL.md`が存在することを確認

### 「ルールが機能しない」
- ルールはサブディレクトリではなくフラットファイル: `$TARGET/rules/coding-style.md`（正しい）vs `$TARGET/rules/common/coding-style.md`（フラットインストールでは不正）
- ルールをインストールした後、Claude Codeを再起動する

### 「プロジェクトレベルインストール後のパス参照エラー」
- 一部のスキルは`~/.claude/`パスを前提としている。ステップ4の検証を実行してこれらを見つけて修正する。
- `continuous-learning-v2`の場合、`~/.claude/homunculus/`ディレクトリは常にユーザーレベル — これは想定通りでありエラーではない。
