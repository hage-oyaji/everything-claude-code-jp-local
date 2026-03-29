# Everything Claude Codeへのコントリビューション

コントリビューションにご興味をお持ちいただきありがとうございます！このリポジトリはClaude Codeユーザーのためのコミュニティリソースです。

## 目次

- [求めているもの](#求めているもの)
- [クイックスタート](#クイックスタート)
- [スキルのコントリビューション](#スキルのコントリビューション)
- [エージェントのコントリビューション](#エージェントのコントリビューション)
- [フックのコントリビューション](#フックのコントリビューション)
- [コマンドのコントリビューション](#コマンドのコントリビューション)
- [MCPとドキュメント（例：Context7）](#mcpとドキュメント例context7)
- [クロスハーネスと翻訳](#クロスハーネスと翻訳)
- [プルリクエストプロセス](#プルリクエストプロセス)

---

## 求めているもの

### エージェント
特定のタスクを適切に処理する新しいエージェント：
- 言語固有のレビューア（Python、Go、Rust）
- フレームワークエキスパート（Django、Rails、Laravel、Spring）
- DevOpsスペシャリスト（Kubernetes、Terraform、CI/CD）
- ドメインエキスパート（MLパイプライン、データエンジニアリング、モバイル）

### スキル
ワークフロー定義とドメイン知識：
- 言語のベストプラクティス
- フレームワークパターン
- テスト戦略
- アーキテクチャガイド

### フック
便利な自動化：
- リンティング/フォーマットフック
- セキュリティチェック
- バリデーションフック
- 通知フック

### コマンド
便利なワークフローを呼び出すスラッシュコマンド：
- デプロイコマンド
- テストコマンド
- コード生成コマンド

---

## クイックスタート

```bash
# 1. Fork and clone
gh repo fork affaan-m/everything-claude-code --clone
cd everything-claude-code

# 2. Create a branch
git checkout -b feat/my-contribution

# 3. Add your contribution (see sections below)

# 4. Test locally
cp -r skills/my-skill ~/.claude/skills/  # for skills
# Then test with Claude Code

# 5. Submit PR
git add . && git commit -m "feat: add my-skill" && git push -u origin feat/my-contribution
```

---

## スキルのコントリビューション

スキルはコンテキストに基づいてClaude Codeが読み込むナレッジモジュールです。

### ディレクトリ構造

```
skills/
└── your-skill-name/
    └── SKILL.md
```

### SKILL.mdテンプレート

```markdown
---
name: your-skill-name
description: スキルリストに表示される簡潔な説明
origin: ECC
---

# スキルタイトル

このスキルがカバーする内容の概要。

## コアコンセプト

主要なパターンとガイドラインを説明。

## コード例

\`\`\`typescript
// Include practical, tested examples
function example() {
  // Well-commented code
}
\`\`\`

## ベストプラクティス

- 実行可能なガイドライン
- すべきこととすべきでないこと
- 避けるべき一般的な落とし穴

## 使用場面

このスキルが適用されるシナリオを説明。
```

### スキルチェックリスト

- [ ] 1つのドメイン/技術に焦点を当てている
- [ ] 実用的なコード例が含まれている
- [ ] 500行未満
- [ ] 明確なセクションヘッダーを使用している
- [ ] Claude Codeでテスト済み

### スキルの例

| スキル | 目的 |
|-------|---------|
| `coding-standards/` | TypeScript/JavaScriptのパターン |
| `frontend-patterns/` | ReactとNext.jsのベストプラクティス |
| `backend-patterns/` | APIとデータベースのパターン |
| `security-review/` | セキュリティチェックリスト |

---

## エージェントのコントリビューション

エージェントはTaskツールを介して呼び出される特化型アシスタントです。

### ファイルの配置場所

```
agents/your-agent-name.md
```

### エージェントテンプレート

```markdown
---
name: your-agent-name
description: このエージェントが何をし、いつClaude がそれを呼び出すべきか。具体的に！
tools: ["Read", "Write", "Edit", "Bash", "Grep", "Glob"]
model: sonnet
---

あなたは[役割]のスペシャリストです。

## あなたの役割

- 主要な責任
- 副次的な責任
- やらないこと（境界）

## ワークフロー

### ステップ1：理解
タスクへのアプローチ方法。

### ステップ2：実行
作業の遂行方法。

### ステップ3：検証
結果のバリデーション方法。

## 出力フォーマット

ユーザーに返す内容。

## 例

### 例：[シナリオ]
入力：[ユーザーが提供するもの]
アクション：[実行すること]
出力：[返すもの]
```

### エージェントフィールド

| フィールド | 説明 | オプション |
|-------|-------------|---------|
| `name` | 小文字、ハイフン区切り | `code-reviewer` |
| `description` | いつ呼び出すかの判断に使用 | 具体的に！ |
| `tools` | 必要なものだけ | `Read, Write, Edit, Bash, Grep, Glob, WebFetch, Task`、またはエージェントがMCPを使用する場合はMCPツール名（例：`mcp__context7__resolve-library-id`、`mcp__context7__query-docs`） |
| `model` | 複雑度レベル | `haiku`（シンプル）、`sonnet`（コーディング）、`opus`（複雑） |

### エージェントの例

| エージェント | 目的 |
|-------|---------|
| `tdd-guide.md` | テスト駆動開発 |
| `code-reviewer.md` | コードレビュー |
| `security-reviewer.md` | セキュリティスキャン |
| `build-error-resolver.md` | ビルドエラーの修正 |

---

## フックのコントリビューション

フックはClaude Codeのイベントでトリガーされる自動的な動作です。

### ファイルの配置場所

```
hooks/hooks.json
```

### フックタイプ

| タイプ | トリガー | ユースケース |
|------|---------|----------|
| `PreToolUse` | ツール実行前 | バリデーション、警告、ブロック |
| `PostToolUse` | ツール実行後 | フォーマット、チェック、通知 |
| `SessionStart` | セッション開始時 | コンテキストの読み込み |
| `Stop` | セッション終了時 | クリーンアップ、監査 |

### フックフォーマット

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "tool == \"Bash\" && tool_input.command matches \"rm -rf /\"",
        "hooks": [
          {
            "type": "command",
            "command": "echo '[Hook] BLOCKED: Dangerous command' && exit 1"
          }
        ],
        "description": "Block dangerous rm commands"
      }
    ]
  }
}
```

### マッチャー構文

```javascript
// Match specific tools
tool == "Bash"
tool == "Edit"
tool == "Write"

// Match input patterns
tool_input.command matches "npm install"
tool_input.file_path matches "\\.tsx?$"

// Combine conditions
tool == "Bash" && tool_input.command matches "git push"
```

### フックの例

```json
// Block dev servers outside tmux
{
  "matcher": "tool == \"Bash\" && tool_input.command matches \"npm run dev\"",
  "hooks": [{"type": "command", "command": "echo 'Use tmux for dev servers' && exit 1"}],
  "description": "Ensure dev servers run in tmux"
}

// Auto-format after editing TypeScript
{
  "matcher": "tool == \"Edit\" && tool_input.file_path matches \"\\.tsx?$\"",
  "hooks": [{"type": "command", "command": "npx prettier --write \"$file_path\""}],
  "description": "Format TypeScript files after edit"
}

// Warn before git push
{
  "matcher": "tool == \"Bash\" && tool_input.command matches \"git push\"",
  "hooks": [{"type": "command", "command": "echo '[Hook] Review changes before pushing'"}],
  "description": "Reminder to review before push"
}
```

### フックチェックリスト

- [ ] マッチャーが具体的である（過度に広くない）
- [ ] 明確なエラー/情報メッセージが含まれている
- [ ] 正しい終了コードを使用している（`exit 1`はブロック、`exit 0`は許可）
- [ ] 十分にテスト済み
- [ ] descriptionがある

---

## コマンドのコントリビューション

コマンドは`/command-name`でユーザーが呼び出すアクションです。

### ファイルの配置場所

```
commands/your-command.md
```

### コマンドテンプレート

```markdown
---
description: /helpに表示される簡潔な説明
---

# コマンド名

## 目的

このコマンドが何をするか。

## 使用方法

\`\`\`
/your-command [args]
\`\`\`

## ワークフロー

1. 最初のステップ
2. 次のステップ
3. 最終ステップ

## 出力

ユーザーが受け取るもの。
```

### コマンドの例

| コマンド | 目的 |
|---------|---------|
| `commit.md` | gitコミットの作成 |
| `code-review.md` | コード変更のレビュー |
| `tdd.md` | TDDワークフロー |
| `e2e.md` | E2Eテスト |

---

## MCPとドキュメント（例：Context7）

スキルとエージェントは、トレーニングデータだけに依存せず、**MCP（Model Context Protocol）**ツールを使用して最新のデータを取り込むことができます。これはドキュメントに特に便利です。

- **Context7**は`resolve-library-id`と`query-docs`を公開するMCPサーバーです。ユーザーがライブラリ、フレームワーク、またはAPIについて質問した際に使用すると、回答に最新のドキュメントとコード例が反映されます。
- ライブドキュメントに依存する**スキル**を提供する場合（例：セットアップ、API使用法）、関連するMCPツールの使用方法（例：ライブラリIDの解決、次にドキュメントのクエリ）を説明し、`documentation-lookup`スキルまたはContext7をパターンとして示してください。
- ドキュメント/APIの質問に回答する**エージェント**を提供する場合、エージェントのツールにContext7 MCPツール名（例：`mcp__context7__resolve-library-id`、`mcp__context7__query-docs`）を含め、resolve → queryワークフローを文書化してください。
- **mcp-configs/mcp-servers.json**にはContext7エントリが含まれています。ユーザーは自分のハーネス（例：Claude Code、Cursor）で有効にすると、documentation-lookupスキル（`skills/documentation-lookup/`内）と`/docs`コマンドを使用できます。

---

## クロスハーネスと翻訳

### スキルサブセット（CodexとCursor）

ECCは他のハーネス用にスキルサブセットを同梱しています：

- **Codex：** `.agents/skills/` — `agents/openai.yaml`に記載されたスキルがCodexにより読み込まれます。
- **Cursor：** `.cursor/skills/` — スキルのサブセットがCursor用にバンドルされています。

**新しいスキルを追加**してCodexまたはCursorで利用可能にする場合：

1. 通常どおり`skills/your-skill-name/`配下にスキルを追加します。
2. **Codex**で利用可能にする場合、`.agents/skills/`に追加し（スキルディレクトリをコピーまたは参照を追加）、必要に応じて`agents/openai.yaml`で参照されていることを確認してください。
3. **Cursor**で利用可能にする場合、Cursorのレイアウトに従って`.cursor/skills/`に追加してください。

期待される構造については、これらのディレクトリ内の既存スキルを確認してください。これらのサブセットの同期は手動です。更新した場合はPRに記載してください。

### 翻訳

翻訳は`docs/`配下にあります（例：`docs/zh-CN`、`docs/zh-TW`、`docs/ja-JP`）。翻訳されているエージェント、コマンド、またはスキルを変更する場合、対応する翻訳ファイルの更新を検討するか、メンテナーや翻訳者が更新できるようにIssueを作成してください。

---

## プルリクエストプロセス

### 1. PRタイトル形式

```
feat(skills): add rust-patterns skill
feat(agents): add api-designer agent
feat(hooks): add auto-format hook
fix(skills): update React patterns
docs: improve contributing guide
```

### 2. PR説明

```markdown
## 概要
追加する内容とその理由。

## タイプ
- [ ] スキル
- [ ] エージェント
- [ ] フック
- [ ] コマンド

## テスト
テスト方法。

## チェックリスト
- [ ] フォーマットガイドラインに従っている
- [ ] Claude Codeでテスト済み
- [ ] 機密情報なし（APIキー、パス）
- [ ] 明確な説明
```

### 3. レビュープロセス

1. メンテナーが48時間以内にレビュー
2. フィードバックがあれば対応
3. 承認されたらmainにマージ

---

## ガイドライン

### すべきこと
- コントリビューションをフォーカスしてモジュラーに保つ
- 明確な説明を含める
- 提出前にテストする
- 既存のパターンに従う
- 依存関係をドキュメント化する

### すべきでないこと
- 機密データ（APIキー、トークン、パス）を含める
- 過度に複雑またはニッチな設定を追加する
- テストされていないコントリビューションを提出する
- 既存機能の重複を作成する

---

## ファイル命名規則

- 小文字のハイフン区切りを使用：`python-reviewer.md`
- 記述的に：`tdd-workflow.md`であって`workflow.md`ではない
- 名前とファイル名を一致させる

---

## 質問は？

- **Issues：** [github.com/affaan-m/everything-claude-code/issues](https://github.com/affaan-m/everything-claude-code/issues)
- **X/Twitter：** [@affaanmustafa](https://x.com/affaanmustafa)

---

コントリビューションありがとうございます！一緒に素晴らしいリソースを作りましょう。
