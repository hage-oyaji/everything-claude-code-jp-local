---
name: prune
description: 昇格されなかった30日以上前の保留中のインスティンクトを削除します
command: true
---

# 保留中インスティンクトの削除

自動生成されたが、レビューまたは昇格されなかった期限切れの保留中インスティンクトを削除します。

## 実装

プラグインのルートパスを使用してインスティンクトCLIを実行します：

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" prune
```

`CLAUDE_PLUGIN_ROOT`が設定されていない場合（手動インストール）：

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py prune
```

## 使い方

```
/prune                    # 30日以上前のインスティンクトを削除
/prune --max-age 60      # カスタム日数しきい値
/prune --dry-run         # 削除せずにプレビュー
```
