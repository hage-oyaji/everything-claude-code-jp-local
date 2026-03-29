---
name: security-scan
description: AgentShieldを使用してClaude Codeの設定（.claude/ディレクトリ）のセキュリティ脆弱性、設定ミス、インジェクションリスクをスキャン。CLAUDE.md、settings.json、MCPサーバー、フック、エージェント定義をチェック。
origin: ECC
---

# セキュリティスキャンスキル

[AgentShield](https://github.com/affaan-m/agentshield)を使用してClaude Codeの設定をセキュリティ問題について監査します。

## 発動タイミング

- 新しいClaude Codeプロジェクトのセットアップ時
- `.claude/settings.json`、`CLAUDE.md`、またはMCP設定の変更後
- 設定変更のコミット前
- 既存のClaude Code設定を持つ新しいリポジトリへのオンボーディング時
- 定期的なセキュリティ衛生チェック

## スキャン対象

| ファイル | チェック内容 |
|------|--------|
| `CLAUDE.md` | ハードコードされたシークレット、自動実行命令、プロンプトインジェクションパターン |
| `settings.json` | 過度に許可的な許可リスト、欠落した拒否リスト、危険なバイパスフラグ |
| `mcp.json` | リスクのあるMCPサーバー、ハードコードされた環境シークレット、npxサプライチェーンリスク |
| `hooks/` | 補間によるコマンドインジェクション、データ持ち出し、サイレントエラー抑制 |
| `agents/*.md` | 制限のないツールアクセス、プロンプトインジェクション面、欠落したモデル仕様 |

## 前提条件

AgentShieldがインストールされている必要があります。必要に応じて確認・インストール：

```bash
# Check if installed
npx ecc-agentshield --version

# Install globally (recommended)
npm install -g ecc-agentshield

# Or run directly via npx (no install needed)
npx ecc-agentshield scan .
```

## 使い方

### 基本スキャン

現在のプロジェクトの`.claude/`ディレクトリに対して実行：

```bash
# Scan current project
npx ecc-agentshield scan

# Scan a specific path
npx ecc-agentshield scan --path /path/to/.claude

# Scan with minimum severity filter
npx ecc-agentshield scan --min-severity medium
```

### 出力フォーマット

```bash
# Terminal output (default) — colored report with grade
npx ecc-agentshield scan

# JSON — for CI/CD integration
npx ecc-agentshield scan --format json

# Markdown — for documentation
npx ecc-agentshield scan --format markdown

# HTML — self-contained dark-theme report
npx ecc-agentshield scan --format html > security-report.html
```

### 自動修正

安全な修正を自動的に適用（自動修正可能とマークされたもののみ）：

```bash
npx ecc-agentshield scan --fix
```

これにより以下が行われます：
- ハードコードされたシークレットを環境変数参照に置換
- ワイルドカードパーミッションをスコープ付き代替に厳格化
- 手動のみの提案は変更しない

### Opus 4.6ディープ分析

より深い分析のために対抗的3エージェントパイプラインを実行：

```bash
# Requires ANTHROPIC_API_KEY
export ANTHROPIC_API_KEY=your-key
npx ecc-agentshield scan --opus --stream
```

以下を実行します：
1. **Attacker（レッドチーム）** — 攻撃ベクトルを発見
2. **Defender（ブルーチーム）** — 強化を推奨
3. **Auditor（最終判定）** — 両方の視点を統合

### 安全な設定の初期化

新しい安全な`.claude/`設定をゼロからスキャフォールド：

```bash
npx ecc-agentshield init
```

作成されるもの：
- スコープ付きパーミッションと拒否リストを持つ`settings.json`
- セキュリティベストプラクティスを含む`CLAUDE.md`
- `mcp.json`プレースホルダー

### GitHub Action

CIパイプラインに追加：

```yaml
- uses: affaan-m/agentshield@v1
  with:
    path: '.'
    min-severity: 'medium'
    fail-on-findings: true
```

## 深刻度レベル

| グレード | スコア | 意味 |
|-------|-------|---------|
| A | 90-100 | 安全な設定 |
| B | 75-89 | 軽微な問題 |
| C | 60-74 | 注意が必要 |
| D | 40-59 | 重大なリスク |
| F | 0-39 | 致命的な脆弱性 |

## 結果の解釈

### 致命的な発見事項（即座に修正）
- 設定ファイルにハードコードされたAPIキーやトークン
- 許可リストに`Bash(*)`（無制限のシェルアクセス）
- `${file}`補間によるフック内のコマンドインジェクション
- シェル実行型MCPサーバー

### 高い発見事項（プロダクション前に修正）
- CLAUDE.mdの自動実行命令（プロンプトインジェクションベクトル）
- パーミッションに欠落した拒否リスト
- 不要なBashアクセスを持つエージェント

### 中程度の発見事項（推奨）
- フック内のサイレントエラー抑制（`2>/dev/null`、`|| true`）
- 欠落したPreToolUseセキュリティフック
- MCP設定での`npx -y`自動インストール

### 情報レベルの発見事項（認識）
- MCPサーバーの欠落した説明
- 適切なプラクティスとして正しくフラグされた禁止命令

## リンク

- **GitHub**: [github.com/affaan-m/agentshield](https://github.com/affaan-m/agentshield)
- **npm**: [npmjs.com/package/ecc-agentshield](https://www.npmjs.com/package/ecc-agentshield)
