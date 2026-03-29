---
name: continuous-learning
description: Claude Codeセッションから再利用可能なパターンを自動的に抽出し、学習済みスキルとして保存。
origin: ECC
---

# 継続的学習スキル

セッション終了時にClaude Codeセッションを自動評価し、学習済みスキルとして保存可能な再利用可能パターンを抽出する。

## 発動条件

- Claude Codeセッションからの自動パターン抽出を設定するとき
- セッション評価のためのStopフックを設定するとき
- `~/.claude/skills/learned/`の学習済みスキルをレビューまたはキュレーションするとき
- 抽出しきい値やパターンカテゴリを調整するとき
- v1（本スキル）とv2（インスティンクトベース）のアプローチを比較するとき

## 動作方法

このスキルはセッション終了時の**Stopフック**として実行される:

1. **セッション評価**: セッションに十分なメッセージがあるか確認（デフォルト: 10以上）
2. **パターン検出**: セッションから抽出可能なパターンを特定
3. **スキル抽出**: 有用なパターンを`~/.claude/skills/learned/`に保存

## 設定

`config.json`を編集してカスタマイズ:

```json
{
  "min_session_length": 10,
  "extraction_threshold": "medium",
  "auto_approve": false,
  "learned_skills_path": "~/.claude/skills/learned/",
  "patterns_to_detect": [
    "error_resolution",
    "user_corrections",
    "workarounds",
    "debugging_techniques",
    "project_specific"
  ],
  "ignore_patterns": [
    "simple_typos",
    "one_time_fixes",
    "external_api_issues"
  ]
}
```

## パターンタイプ

| パターン | 説明 |
|---------|-------------|
| `error_resolution` | 特定のエラーがどのように解決されたか |
| `user_corrections` | ユーザーの修正から得られたパターン |
| `workarounds` | フレームワーク/ライブラリの癖に対する解決策 |
| `debugging_techniques` | 効果的なデバッグアプローチ |
| `project_specific` | プロジェクト固有の規約 |

## フックのセットアップ

`~/.claude/settings.json`に追加:

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning/evaluate-session.sh"
      }]
    }]
  }
}
```

## Stopフックを選ぶ理由

- **軽量**: セッション終了時に1回だけ実行
- **ノンブロッキング**: すべてのメッセージにレイテンシーを追加しない
- **完全なコンテキスト**: セッション全体のトランスクリプトにアクセス可能

## 関連

- [ロングフォームガイド](https://x.com/affaanmustafa/status/2014040193557471352) - 継続的学習のセクション
- `/learn`コマンド - セッション中の手動パターン抽出

---

## 比較メモ（リサーチ: 2025年1月）

### vs Homunculus

Homunculus v2はより洗練されたアプローチを取る:

| 機能 | 本アプローチ | Homunculus v2 |
|---------|--------------|---------------|
| 観察 | Stopフック（セッション終了時） | PreToolUse/PostToolUseフック（100%信頼性） |
| 分析 | メインコンテキスト | バックグラウンドエージェント（Haiku） |
| 粒度 | 完全なスキル | アトミックな「インスティンクト」 |
| 信頼度 | なし | 0.3-0.9の重み付け |
| 進化 | 直接スキルへ | インスティンクト → クラスター → スキル/コマンド/エージェント |
| 共有 | なし | インスティンクトのエクスポート/インポート |

**Homunculusからの主要な洞察:**
> 「v1はスキルに観察を頼っていた。スキルは確率的で、約50-80%の確率で発火する。v2は観察にフック（100%信頼性）を使用し、学習された行動のアトミックな単位としてインスティンクトを使用する。」

### v2の潜在的な拡張

1. **インスティンクトベースの学習** - 信頼度スコアリングを持つ、より小さくアトミックな行動
2. **バックグラウンドオブザーバー** - 並列で分析するHaikuエージェント
3. **信頼度の減衰** - 矛盾した場合にインスティンクトの信頼度が低下
4. **ドメインタグ付け** - code-style、testing、git、debuggingなど
5. **進化パス** - 関連するインスティンクトをスキル/コマンドにクラスタリング

詳細は`docs/continuous-learning-v2-spec.md`を参照。
