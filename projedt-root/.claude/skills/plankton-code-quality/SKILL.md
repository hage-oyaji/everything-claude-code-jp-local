---
name: plankton-code-quality
description: "Planktonによる書き込み時コード品質強制 — ファイル編集ごとにフックを介した自動フォーマット、リンティング、Claude駆動の修正。"
origin: community
---

# Planktonコード品質スキル

Plankton（クレジット: @alxfazio）のインテグレーションリファレンス。Claude Codeの書き込み時コード品質強制システム。PlanktonはPostToolUseフックを介してファイル編集ごとにフォーマッターとリンターを実行し、エージェントが見落とした違反を修正するためにClaudeサブプロセスを起動する。

## 使用するタイミング

- ファイル編集ごと（コミット時だけでなく）に自動フォーマットとリンティングが必要な場合
- エージェントがコードを修正する代わりにリンター設定を変更してパスさせることへの防御が必要な場合
- 修正のための段階的モデルルーティング（スタイルにはHaiku、ロジックにはSonnet、型にはOpus）が必要な場合
- 複数言語（Python、TypeScript、Shell、YAML、JSON、TOML、Markdown、Dockerfile）で作業する場合

## 仕組み

### 3フェーズアーキテクチャ

Claude Codeがファイルを編集・書き込みするたびに、Planktonの `multi_linter.sh` PostToolUseフックが実行される:

```
Phase 1: Auto-Format (Silent)
├─ Runs formatters (ruff format, biome, shfmt, taplo, markdownlint)
├─ Fixes 40-50% of issues silently
└─ No output to main agent

Phase 2: Collect Violations (JSON)
├─ Runs linters and collects unfixable violations
├─ Returns structured JSON: {line, column, code, message, linter}
└─ Still no output to main agent

Phase 3: Delegate + Verify
├─ Spawns claude -p subprocess with violations JSON
├─ Routes to model tier based on violation complexity:
│   ├─ Haiku: formatting, imports, style (E/W/F codes) — 120s timeout
│   ├─ Sonnet: complexity, refactoring (C901, PLR codes) — 300s timeout
│   └─ Opus: type system, deep reasoning (unresolved-attribute) — 600s timeout
├─ Re-runs Phase 1+2 to verify fixes
└─ Exit 0 if clean, Exit 2 if violations remain (reported to main agent)
```

### メインエージェントが見るもの

| シナリオ | エージェントが見る内容 | フック終了コード |
|----------|-----------|-----------|
| 違反なし | なし | 0 |
| サブプロセスですべて修正済み | なし | 0 |
| サブプロセス後も違反が残る | `[hook] N violation(s) remain` | 2 |
| アドバイザリー（重複、古いツール） | `[hook:advisory] ...` | 0 |

メインエージェントはサブプロセスが修正できなかった問題のみを確認する。ほとんどの品質問題は透過的に解決される。

### 設定保護（ルール回避への防御）

LLMはコードを修正する代わりに `.ruff.toml` や `biome.json` を変更してルールを無効化する。Planktonは3層でこれをブロックする:

1. **PreToolUseフック** — `protect_linter_configs.sh` がすべてのリンター設定への編集を発生前にブロック
2. **Stopフック** — `stop_config_guardian.sh` がセッション終了時に `git diff` で設定変更を検出
3. **保護ファイルリスト** — `.ruff.toml`、`biome.json`、`.shellcheckrc`、`.yamllint`、`.hadolint.yaml` など

### パッケージマネージャー強制

BashのPreToolUseフックがレガシーパッケージマネージャーをブロック:
- `pip`、`pip3`、`poetry`、`pipenv` → ブロック（`uv` を使用）
- `npm`、`yarn`、`pnpm` → ブロック（`bun` を使用）
- 許可される例外: `npm audit`、`npm view`、`npm publish`

## セットアップ

### クイックスタート

> **注意:** Planktonはリポジトリからの手動インストールが必要。インストール前にコードを確認すること。

```bash
# Install core dependencies
brew install jaq ruff uv

# Install Python linters
uv sync --all-extras

# Start Claude Code — hooks activate automatically
claude
```

インストールコマンド、プラグイン設定は不要。Planktonディレクトリ内でClaude Codeを実行すると、`.claude/settings.json` のフックが自動的に認識される。

### プロジェクト単位の統合

自分のプロジェクトでPlanktonフックを使用するには:

1. `.claude/hooks/` ディレクトリをプロジェクトにコピー
2. `.claude/settings.json` のフック設定をコピー
3. リンター設定ファイル（`.ruff.toml`、`biome.json` など）をコピー
4. 使用する言語のリンターをインストール

### 言語別依存関係

| 言語 | 必須 | オプション |
|----------|----------|----------|
| Python | `ruff`、`uv` | `ty`（型）、`vulture`（デッドコード）、`bandit`（セキュリティ） |
| TypeScript/JS | `biome` | `oxlint`、`semgrep`、`knip`（デッドエクスポート） |
| Shell | `shellcheck`、`shfmt` | — |
| YAML | `yamllint` | — |
| Markdown | `markdownlint-cli2` | — |
| Dockerfile | `hadolint`（>= 2.12.0） | — |
| TOML | `taplo` | — |
| JSON | `jaq` | — |

## ECCとの併用

### 補完的で重複しない

| 関心事 | ECC | Plankton |
|---------|-----|----------|
| コード品質強制 | PostToolUseフック（Prettier、tsc） | PostToolUseフック（20以上のリンター + サブプロセス修正） |
| セキュリティスキャン | AgentShield、security-reviewerエージェント | Bandit（Python）、Semgrep（TypeScript） |
| 設定保護 | — | PreToolUseブロック + Stopフック検出 |
| パッケージマネージャー | 検出 + セットアップ | 強制（レガシーPMをブロック） |
| CI統合 | — | gitのpre-commitフック |
| モデルルーティング | 手動（`/model opus`） | 自動（違反の複雑さ → ティア） |

### 推奨される組み合わせ

1. ECCをプラグインとしてインストール（エージェント、スキル、コマンド、ルール）
2. 書き込み時品質強制にPlanktonフックを追加
3. セキュリティ監査にAgentShieldを使用
4. PR前の最終ゲートとしてECCのverification-loopを使用

### フックの競合回避

ECCとPlanktonの両方のフックを実行する場合:
- ECCのPrettierフックとPlanktonのbiomeフォーマッターがJS/TSファイルで競合する可能性がある
- 解決策: Plankton使用時はECCのPrettier PostToolUseフックを無効にする（Planktonのbiomeの方が包括的）
- 異なるファイルタイプでは両方が共存可能（ECCがPlanktonがカバーしないものを処理）

## 設定リファレンス

Planktonの `.claude/hooks/config.json` がすべての動作を制御する:

```json
{
  "languages": {
    "python": true,
    "shell": true,
    "yaml": true,
    "json": true,
    "toml": true,
    "dockerfile": true,
    "markdown": true,
    "typescript": {
      "enabled": true,
      "js_runtime": "auto",
      "biome_nursery": "warn",
      "semgrep": true
    }
  },
  "phases": {
    "auto_format": true,
    "subprocess_delegation": true
  },
  "subprocess": {
    "tiers": {
      "haiku":  { "timeout": 120, "max_turns": 10 },
      "sonnet": { "timeout": 300, "max_turns": 10 },
      "opus":   { "timeout": 600, "max_turns": 15 }
    },
    "volume_threshold": 5
  }
}
```

**主な設定:**
- 使用しない言語を無効にしてフックを高速化する
- `volume_threshold` — この数を超える違反は自動的に上位モデルティアにエスカレーション
- `subprocess_delegation: false` — フェーズ3を完全にスキップ（違反のレポートのみ）

## 環境変数オーバーライド

| 変数 | 用途 |
|----------|---------|
| `HOOK_SKIP_SUBPROCESS=1` | フェーズ3をスキップし、違反を直接レポート |
| `HOOK_SUBPROCESS_TIMEOUT=N` | ティアタイムアウトをオーバーライド |
| `HOOK_DEBUG_MODEL=1` | モデル選択の決定をログ出力 |
| `HOOK_SKIP_PM=1` | パッケージマネージャー強制をバイパス |

## リファレンス

- Plankton（クレジット: @alxfazio）
- Plankton REFERENCE.md — 完全なアーキテクチャドキュメント（クレジット: @alxfazio）
- Plankton SETUP.md — 詳細なインストールガイド（クレジット: @alxfazio）

## ECC v1.8追加機能

### コピー可能なフックプロファイル

厳格な品質動作を設定:

```bash
export ECC_HOOK_PROFILE=strict
export ECC_QUALITY_GATE_FIX=true
export ECC_QUALITY_GATE_STRICT=true
```

### 言語ゲートテーブル

- TypeScript/JavaScript: Biome優先、Prettierフォールバック
- Python: Ruff format/check
- Go: gofmt

### 設定改ざんガード

品質強制中、同じイテレーションでの設定ファイルの変更をフラグ:

- `biome.json`、`.eslintrc*`、`prettier.config*`、`tsconfig.json`、`pyproject.toml`

違反を抑制するために設定が変更された場合、マージ前に明示的なレビューを要求する。

### CI統合パターン

ローカルフックと同じコマンドをCIで使用:

1. フォーマッターチェックを実行
2. リント/型チェックを実行
3. strictモードでフェイルファスト
4. 修正サマリーを公開

### ヘルスメトリクス

追跡:
- ゲートによるフラグが付いた編集
- 平均修正時間
- カテゴリ別の繰り返し違反
- ゲート失敗によるマージブロック
