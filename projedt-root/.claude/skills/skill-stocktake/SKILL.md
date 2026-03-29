---
description: "Claudeスキルとコマンドの品質監査に使用。クイックスキャン（変更されたスキルのみ）とフルストックテイクモードをサポートし、シーケンシャルなサブエージェントバッチ評価を実施。"
origin: ECC
---

# skill-stocktake

品質チェックリスト＋AIホリスティック判定を使用して、すべてのClaudeスキルとコマンドを監査するスラッシュコマンド（`/skill-stocktake`）。2つのモードをサポート：最近変更されたスキルのクイックスキャンと、完全なレビューのフルストックテイク。

## スコープ

コマンドは**呼び出されたディレクトリを基準に**以下のパスを対象とします：

| パス | 説明 |
|------|-------------|
| `~/.claude/skills/` | グローバルスキル（全プロジェクト） |
| `{cwd}/.claude/skills/` | プロジェクトレベルスキル（ディレクトリが存在する場合） |

**フェーズ1の開始時に、コマンドはどのパスが見つかりスキャンされたかを明示的にリストします。**

### 特定のプロジェクトを対象にする

プロジェクトレベルスキルを含めるには、そのプロジェクトのルートディレクトリから実行します：

```bash
cd ~/path/to/my-project
/skill-stocktake
```

プロジェクトに`.claude/skills/`ディレクトリがない場合、グローバルスキルとコマンドのみが評価されます。

## モード

| モード | トリガー | 所要時間 |
|------|---------|---------|
| クイックスキャン | `results.json`が存在（デフォルト） | 5〜10分 |
| フルストックテイク | `results.json`が存在しない、または`/skill-stocktake full` | 20〜30分 |

**結果キャッシュ:** `~/.claude/skills/skill-stocktake/results.json`

## クイックスキャンフロー

前回の実行以降に変更されたスキルのみを再評価します（5〜10分）。

1. `~/.claude/skills/skill-stocktake/results.json`を読み込み
2. 実行: `bash ~/.claude/skills/skill-stocktake/scripts/quick-diff.sh \
         ~/.claude/skills/skill-stocktake/results.json`
   （プロジェクトディレクトリは`$PWD/.claude/skills`から自動検出；必要な場合のみ明示的に指定）
3. 出力が`[]`の場合：「前回の実行以降に変更はありません。」と報告して停止
4. 変更されたファイルのみを同じフェーズ2の基準で再評価
5. 前回の結果から変更されていないスキルを引き継ぎ
6. 差分のみを出力
7. 実行: `bash ~/.claude/skills/skill-stocktake/scripts/save-results.sh \
         ~/.claude/skills/skill-stocktake/results.json <<< "$EVAL_RESULTS"`

## フルストックテイクフロー

### フェーズ 1 — インベントリ

実行: `bash ~/.claude/skills/skill-stocktake/scripts/scan.sh`

スクリプトはスキルファイルを列挙し、フロントマターを抽出し、UTCのmtimeを収集します。
プロジェクトディレクトリは`$PWD/.claude/skills`から自動検出；必要な場合のみ明示的に指定。
スクリプト出力からスキャンサマリーとインベントリテーブルを表示：

```
Scanning:
  ✓ ~/.claude/skills/         (17 files)
  ✗ {cwd}/.claude/skills/    (not found — global skills only)
```

| スキル | 7日使用 | 30日使用 | 説明 |
|-------|--------|---------|-------------|

### フェーズ 2 — 品質評価

Agentツールサブエージェント（**汎用エージェント**）に完全なインベントリとチェックリストを渡して起動：

```text
Agent(
  subagent_type="general-purpose",
  prompt="
Evaluate the following skill inventory against the checklist.

[INVENTORY]

[CHECKLIST]

Return JSON for each skill:
{ \"verdict\": \"Keep\"|\"Improve\"|\"Update\"|\"Retire\"|\"Merge into [X]\", \"reason\": \"...\" }
"
)
```

サブエージェントは各スキルを読み取り、チェックリストを適用し、スキルごとのJSONを返します：

`{ "verdict": "Keep"|"Improve"|"Update"|"Retire"|"Merge into [X]", "reason": "..." }`

**チャンクガイダンス:** サブエージェント呼び出しごとに約20スキルを処理し、コンテキストを管理可能に保ちます。各チャンクの後に中間結果を`results.json`（`status: "in_progress"`）に保存します。

すべてのスキルが評価された後：`status: "completed"`に設定し、フェーズ3に進みます。

**再開検出:** 起動時に`status: "in_progress"`が見つかった場合、最初の未評価スキルから再開します。

各スキルは以下のチェックリストに対して評価されます：

```
- [ ] Content overlap with other skills checked
- [ ] Overlap with MEMORY.md / CLAUDE.md checked
- [ ] Freshness of technical references verified (use WebSearch if tool names / CLI flags / APIs are present)
- [ ] Usage frequency considered
```

判定基準：

| 判定 | 意味 |
|---------|---------|
| Keep | 有用で最新 |
| Improve | 保持する価値があるが、特定の改善が必要 |
| Update | 参照している技術が古い（WebSearchで確認） |
| Retire | 低品質、古い、またはコスト非対称 |
| Merge into [X] | 他のスキルと大幅に重複；マージ先を指定 |

評価は**ホリスティックなAI判定**です — 数値ルーブリックではありません。ガイドとなる次元：
- **実行可能性**: 即座に行動できるコード例、コマンド、ステップ
- **スコープの適合性**: 名前、トリガー、コンテンツが整合；広すぎず狭すぎない
- **一意性**: MEMORY.md / CLAUDE.md / 他のスキルで代替不可能な価値
- **最新性**: 技術的参照が現在の環境で機能する

**理由の品質要件** — `reason`フィールドは自己完結型で判断を可能にするものでなければなりません：
- 「unchanged」だけを書かない — 常に核心的な証拠を再記載
- **Retire**の場合：(1) 発見された具体的な欠陥、(2) 代わりに同じニーズをカバーするものを記載
  - 悪い例: `"Superseded"`
  - 良い例: `"disable-model-invocation: true already set; superseded by continuous-learning-v2 which covers all the same patterns plus confidence scoring. No unique content remains."`
- **Merge**の場合：ターゲットを指名し、統合するコンテンツを記述
  - 悪い例: `"Overlaps with X"`
  - 良い例: `"42-line thin content; Step 4 of chatlog-to-article already covers the same workflow. Integrate the 'article angle' tip as a note in that skill."`
- **Improve**の場合：必要な具体的変更を記述（どのセクション、どのアクション、関連する場合は目標サイズ）
  - 悪い例: `"Too long"`
  - 良い例: `"276 lines; Section 'Framework Comparison' (L80–140) duplicates ai-era-architecture-principles; delete it to reach ~150 lines."`
- **Keep**（クイックスキャンでのmtimeのみの変更の場合）：元の判定根拠を再記載、「unchanged」とは書かない
  - 悪い例: `"Unchanged"`
  - 良い例: `"mtime updated but content unchanged. Unique Python reference explicitly imported by rules/python/; no overlap found."`

### フェーズ 3 — サマリーテーブル

| スキル | 7日使用 | 判定 | 理由 |
|-------|--------|---------|--------|

### フェーズ 4 — 統合

1. **Retire / Merge**: ユーザーに確認する前にファイルごとの詳細な根拠を提示：
   - 発見された具体的な問題（重複、古さ、壊れた参照など）
   - 同じ機能をカバーする代替（Retireの場合：どの既存スキル/ルール；Mergeの場合：ターゲットファイルと統合するコンテンツ）
   - 削除の影響（依存スキル、MEMORY.md参照、影響を受けるワークフロー）
2. **Improve**: 根拠付きの具体的な改善提案を提示：
   - 何を変更するか、なぜ（例：「セクションX/Yがpython-patternsと重複するため430→200行にトリム」）
   - ユーザーが実施するかを判断
3. **Update**: ソースを確認した更新コンテンツを提示
4. MEMORY.mdの行数を確認；100行を超える場合は圧縮を提案

## 結果ファイルスキーマ

`~/.claude/skills/skill-stocktake/results.json`:

**`evaluated_at`**: 評価完了の実際のUTC時刻に設定する必要があります。
Bashで取得: `date -u +%Y-%m-%dT%H:%M:%SZ`。`T00:00:00Z`のような日付のみの近似値は使用しないでください。

```json
{
  "evaluated_at": "2026-02-21T10:00:00Z",
  "mode": "full",
  "batch_progress": {
    "total": 80,
    "evaluated": 80,
    "status": "completed"
  },
  "skills": {
    "skill-name": {
      "path": "~/.claude/skills/skill-name/SKILL.md",
      "verdict": "Keep",
      "reason": "Concrete, actionable, unique value for X workflow",
      "mtime": "2026-01-15T08:30:00Z"
    }
  }
}
```

## 注意事項

- 評価はブラインドです：同じチェックリストがオリジン（ECC、自作、自動抽出）に関係なくすべてのスキルに適用されます
- アーカイブ/削除操作には常に明示的なユーザー確認が必要です
- スキルのオリジンによる判定の分岐はありません
