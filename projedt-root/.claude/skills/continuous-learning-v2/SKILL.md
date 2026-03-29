---
name: continuous-learning-v2
description: フックを介してセッションを観察し、信頼度スコアリング付きのアトミックなインスティンクトを作成し、それらをスキル/コマンド/エージェントへ進化させるインスティンクトベースの学習システム。v2.1ではプロジェクトスコープのインスティンクトを追加し、プロジェクト間の汚染を防止します。
origin: ECC
version: 2.1.0
---

# Continuous Learning v2.1 - インスティンクトベースアーキテクチャ

Claude Codeのセッションを、アトミックな「インスティンクト」（信頼度スコアリング付きの小さな学習済み行動）を通じて再利用可能な知識に変換する高度な学習システムです。

**v2.1** では**プロジェクトスコープのインスティンクト**を追加しました — Reactのパターンはあなたのreactプロジェクトに、Pythonの慣例はPythonプロジェクトに留まり、ユニバーサルなパターン（「常に入力をバリデーションする」など）はグローバルに共有されます。

## 有効化するタイミング

- Claude Codeセッションからの自動学習をセットアップする場合
- フック経由でインスティンクトベースの行動抽出を設定する場合
- 学習済み行動の信頼度しきい値を調整する場合
- インスティンクトライブラリの確認、エクスポート、またはインポートを行う場合
- インスティンクトを完全なスキル、コマンド、またはエージェントへ進化させる場合
- プロジェクトスコープとグローバルのインスティンクトを管理する場合
- インスティンクトをプロジェクトからグローバルスコープに昇格させる場合

## v2.1の新機能

| 機能 | v2.0 | v2.1 |
|------|------|------|
| ストレージ | グローバル (~/.claude/homunculus/) | プロジェクトスコープ (projects/<hash>/) |
| スコープ | すべてのインスティンクトがどこでも適用 | プロジェクトスコープ + グローバル |
| 検出 | なし | git remote URL / リポジトリパス |
| 昇格 | N/A | 2つ以上のプロジェクトで確認された場合にプロジェクト → グローバル |
| コマンド | 4つ (status/evolve/export/import) | 6つ (+promote/projects) |
| プロジェクト間 | 汚染リスクあり | デフォルトで隔離 |

## v2の新機能（v1との比較）

| 機能 | v1 | v2 |
|------|----|----|
| 観察 | Stopフック（セッション終了） | PreToolUse/PostToolUse（100%信頼性） |
| 分析 | メインコンテキスト | バックグラウンドエージェント（Haiku） |
| 粒度 | 完全なスキル | アトミックな「インスティンクト」 |
| 信頼度 | なし | 0.3-0.9の加重 |
| 進化 | スキルへ直接 | インスティンクト -> クラスタ -> スキル/コマンド/エージェント |
| 共有 | なし | インスティンクトのエクスポート/インポート |

## インスティンクトモデル

インスティンクトとは小さな学習済み行動です：

```yaml
---
id: prefer-functional-style
trigger: "when writing new functions"
confidence: 0.7
domain: "code-style"
source: "session-observation"
scope: project
project_id: "a1b2c3d4e5f6"
project_name: "my-react-app"
---

# Prefer Functional Style

## Action
Use functional patterns over classes when appropriate.

## Evidence
- Observed 5 instances of functional pattern preference
- User corrected class-based approach to functional on 2025-01-15
```

**プロパティ：**
- **アトミック** -- 1つのトリガー、1つのアクション
- **信頼度加重** -- 0.3 = 暫定的、0.9 = ほぼ確実
- **ドメインタグ付き** -- code-style、testing、git、debugging、workflowなど
- **エビデンスベース** -- どの観察が作成したかを追跡
- **スコープ対応** -- `project`（デフォルト）または `global`

## 仕組み

```
Session Activity (in a git repo)
      |
      | Hooks capture prompts + tool use (100% reliable)
      | + detect project context (git remote / repo path)
      v
+---------------------------------------------+
|  projects/<project-hash>/observations.jsonl  |
|   (prompts, tool calls, outcomes, project)   |
+---------------------------------------------+
      |
      | Observer agent reads (background, Haiku)
      v
+---------------------------------------------+
|          PATTERN DETECTION                   |
|   * User corrections -> instinct             |
|   * Error resolutions -> instinct            |
|   * Repeated workflows -> instinct           |
|   * Scope decision: project or global?       |
+---------------------------------------------+
      |
      | Creates/updates
      v
+---------------------------------------------+
|  projects/<project-hash>/instincts/personal/ |
|   * prefer-functional.yaml (0.7) [project]   |
|   * use-react-hooks.yaml (0.9) [project]     |
+---------------------------------------------+
|  instincts/personal/  (GLOBAL)               |
|   * always-validate-input.yaml (0.85) [global]|
|   * grep-before-edit.yaml (0.6) [global]     |
+---------------------------------------------+
      |
      | /evolve clusters + /promote
      v
+---------------------------------------------+
|  projects/<hash>/evolved/ (project-scoped)   |
|  evolved/ (global)                           |
|   * commands/new-feature.md                  |
|   * skills/testing-workflow.md               |
|   * agents/refactor-specialist.md            |
+---------------------------------------------+
```

## プロジェクト検出

システムは現在のプロジェクトを自動検出します：

1. **`CLAUDE_PROJECT_DIR` 環境変数**（最優先）
2. **`git remote get-url origin`** -- ハッシュ化してポータブルなプロジェクトIDを作成（異なるマシン上の同じリポジトリは同じIDを取得）
3. **`git rev-parse --show-toplevel`** -- リポジトリパスを使用するフォールバック（マシン固有）
4. **グローバルフォールバック** -- プロジェクトが検出されない場合、インスティンクトはグローバルスコープへ

各プロジェクトは12文字のハッシュID（例：`a1b2c3d4e5f6`）を取得します。`~/.claude/homunculus/projects.json` のレジストリファイルがIDを人間が読める名前にマッピングします。

## クイックスタート

### 1. 観察フックを有効化

`~/.claude/settings.json` に追加します。

**プラグインとしてインストールした場合**（推奨）：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/hooks/observe.sh"
      }]
    }],
    "PostToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/hooks/observe.sh"
      }]
    }]
  }
}
```

**手動で** `~/.claude/skills` にインストールした場合：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning-v2/hooks/observe.sh"
      }]
    }],
    "PostToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning-v2/hooks/observe.sh"
      }]
    }]
  }
}
```

### 2. ディレクトリ構造の初期化

システムは初回使用時に自動でディレクトリを作成しますが、手動で作成することもできます：

```bash
# グローバルディレクトリ
mkdir -p ~/.claude/homunculus/{instincts/{personal,inherited},evolved/{agents,skills,commands},projects}

# プロジェクトディレクトリはフックがgitリポジトリ内で初めて実行された時に自動作成されます
```

### 3. インスティンクトコマンドを使用

```bash
/instinct-status     # 学習済みインスティンクトを表示（プロジェクト + グローバル）
/evolve              # 関連するインスティンクトをスキル/コマンドにクラスタリング
/instinct-export     # インスティンクトをファイルにエクスポート
/instinct-import     # 他者からインスティンクトをインポート
/promote             # プロジェクトのインスティンクトをグローバルスコープに昇格
/projects            # 既知のすべてのプロジェクトとそのインスティンクト数を一覧表示
```

## コマンド

| コマンド | 説明 |
|---------|------|
| `/instinct-status` | すべてのインスティンクト（プロジェクトスコープ + グローバル）を信頼度付きで表示 |
| `/evolve` | 関連するインスティンクトをスキル/コマンドにクラスタリングし、昇格を提案 |
| `/instinct-export` | インスティンクトをエクスポート（スコープ/ドメインでフィルタ可能） |
| `/instinct-import <file>` | スコープ制御付きでインスティンクトをインポート |
| `/promote [id]` | プロジェクトのインスティンクトをグローバルスコープに昇格 |
| `/projects` | 既知のすべてのプロジェクトとそのインスティンクト数を一覧表示 |

## 設定

`config.json` を編集してバックグラウンドオブザーバーを制御：

```json
{
  "version": "2.1",
  "observer": {
    "enabled": false,
    "run_interval_minutes": 5,
    "min_observations_to_analyze": 20
  }
}
```

| キー | デフォルト | 説明 |
|-----|---------|------|
| `observer.enabled` | `false` | バックグラウンドオブザーバーエージェントを有効化 |
| `observer.run_interval_minutes` | `5` | オブザーバーが観察を分析する頻度 |
| `observer.min_observations_to_analyze` | `20` | 分析が実行される前の最小観察数 |

その他の動作（観察キャプチャ、インスティンクトしきい値、プロジェクトスコーピング、昇格基準）は `instinct-cli.py` と `observe.sh` のコードデフォルトで設定されます。

## ファイル構造

```
~/.claude/homunculus/
+-- identity.json           # あなたのプロフィール、技術レベル
+-- projects.json           # レジストリ：プロジェクトハッシュ -> 名前/パス/リモート
+-- observations.jsonl      # グローバル観察（フォールバック）
+-- instincts/
|   +-- personal/           # グローバル自動学習インスティンクト
|   +-- inherited/          # グローバルインポート済みインスティンクト
+-- evolved/
|   +-- agents/             # グローバル生成エージェント
|   +-- skills/             # グローバル生成スキル
|   +-- commands/           # グローバル生成コマンド
+-- projects/
    +-- a1b2c3d4e5f6/       # プロジェクトハッシュ（git remote URLから）
    |   +-- project.json    # プロジェクトごとのメタデータミラー（id/name/root/remote）
    |   +-- observations.jsonl
    |   +-- observations.archive/
    |   +-- instincts/
    |   |   +-- personal/   # プロジェクト固有の自動学習
    |   |   +-- inherited/  # プロジェクト固有のインポート済み
    |   +-- evolved/
    |       +-- skills/
    |       +-- commands/
    |       +-- agents/
    +-- f6e5d4c3b2a1/       # 別のプロジェクト
        +-- ...
```

## スコープ判定ガイド

| パターンの種類 | スコープ | 例 |
|-------------|---------|---|
| 言語/フレームワークの慣例 | **project** | 「React Hooksを使用」、「Django RESTパターンに従う」 |
| ファイル構造の好み | **project** | 「テストは `__tests__`/ に」、「コンポーネントは src/components/ に」 |
| コードスタイル | **project** | 「関数型スタイルを使用」、「dataclassesを優先」 |
| エラーハンドリング戦略 | **project** | 「エラーにResult型を使用」 |
| セキュリティプラクティス | **global** | 「ユーザー入力をバリデーション」、「SQLをサニタイズ」 |
| 一般的なベストプラクティス | **global** | 「テストを先に書く」、「常にエラーをハンドリング」 |
| ツールワークフローの好み | **global** | 「編集前にGrep」、「書き込み前にRead」 |
| Gitプラクティス | **global** | 「コンベンショナルコミット」、「小さく焦点を絞ったコミット」 |

## インスティンクトの昇格（プロジェクト -> グローバル）

同じインスティンクトが複数のプロジェクトで高い信頼度で出現する場合、グローバルスコープへの昇格候補となります。

**自動昇格基準：**
- 同じインスティンクトIDが2つ以上のプロジェクトに存在
- 平均信頼度 >= 0.8

**昇格方法：**

```bash
# 特定のインスティンクトを昇格
python3 instinct-cli.py promote prefer-explicit-errors

# 資格のあるすべてのインスティンクトを自動昇格
python3 instinct-cli.py promote

# 変更なしでプレビュー
python3 instinct-cli.py promote --dry-run
```

`/evolve` コマンドも昇格候補を提案します。

## 信頼度スコアリング

信頼度は時間とともに変化します：

| スコア | 意味 | 動作 |
|-------|------|------|
| 0.3 | 暫定的 | 提案されるが強制されない |
| 0.5 | 中程度 | 関連する場合に適用 |
| 0.7 | 強い | 適用が自動承認 |
| 0.9 | ほぼ確実 | コア動作 |

**信頼度が上がる場合：**
- パターンが繰り返し観察される
- ユーザーが提案された行動を修正しない
- 他のソースからの類似インスティンクトが一致する

**信頼度が下がる場合：**
- ユーザーが明示的に行動を修正する
- パターンが長期間観察されない
- 矛盾するエビデンスが出現する

## 観察にスキルではなくフックを使う理由

> 「v1はスキルに依存して観察していました。スキルは確率的で、Claudeの判断に基づいて約50-80%の確率で発動します。」

フックは**100%の確率**で、決定論的に発動します。これは以下を意味します：
- すべてのツールコールが観察される
- パターンが見逃されない
- 学習が包括的である

## 後方互換性

v2.1はv2.0およびv1と完全に互換性があります：
- `~/.claude/homunculus/instincts/` の既存のグローバルインスティンクトはグローバルインスティンクトとして引き続き機能
- v1の `~/.claude/skills/learned/` の既存スキルも引き続き機能
- Stopフックも引き続き実行（ただしv2にもフィード）
- 段階的移行：両方を並行して実行可能

## プライバシー

- 観察はあなたのマシンに**ローカルに**留まります
- プロジェクトスコープのインスティンクトはプロジェクトごとに隔離されます
- **インスティンクト**（パターン）のみがエクスポート可能 — 生の観察データではありません
- 実際のコードや会話の内容は共有されません
- エクスポートと昇格はあなたが制御します

## 関連

- [ECC-Tools GitHub App](https://github.com/apps/ecc-tools) - リポジトリ履歴からインスティンクトを生成
- Homunculus - v2のインスティンクトベースアーキテクチャに影響を与えたコミュニティプロジェクト（アトミック観察、信頼度スコアリング、インスティンクト進化パイプライン）
- [The Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352) - 継続的学習セクション

---

*インスティンクトベースの学習：Claudeにあなたのパターンを1プロジェクトずつ教えます。*
