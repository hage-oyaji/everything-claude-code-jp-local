---
name: skill-health
description: スキルポートフォリオのヘルスダッシュボードをチャートと分析とともに表示します
command: true
---

# スキルヘルスダッシュボード

成功率のスパークライン、障害パターンのクラスタリング、保留中の修正提案、バージョン履歴を含む、ポートフォリオ内のすべてのスキルに対する包括的なヘルスダッシュボードを表示します。

## 実装

ダッシュボードモードでスキルヘルスCLIを実行します：

```bash
ECC_ROOT="${CLAUDE_PLUGIN_ROOT:-$(node -e "var p=require('path'),f=require('fs'),h=require('os').homedir(),d=p.join(h,'.claude'),q=p.join('scripts','lib','utils.js');if(!f.existsSync(p.join(d,q))){try{var b=p.join(d,'plugins','cache','everything-claude-code');for(var o of f.readdirSync(b))for(var v of f.readdirSync(p.join(b,o))){var c=p.join(b,o,v);if(f.existsSync(p.join(c,q))){d=c;break}}}catch(x){}}console.log(d)")}"
node "$ECC_ROOT/scripts/skills-health.js" --dashboard
```

特定のパネルのみの場合：

```bash
ECC_ROOT="${CLAUDE_PLUGIN_ROOT:-$(node -e "var p=require('path'),f=require('fs'),h=require('os').homedir(),d=p.join(h,'.claude'),q=p.join('scripts','lib','utils.js');if(!f.existsSync(p.join(d,q))){try{var b=p.join(d,'plugins','cache','everything-claude-code');for(var o of f.readdirSync(b))for(var v of f.readdirSync(p.join(b,o))){var c=p.join(b,o,v);if(f.existsSync(p.join(c,q))){d=c;break}}}catch(x){}}console.log(d)")}"
node "$ECC_ROOT/scripts/skills-health.js" --dashboard --panel failures
```

機械可読な出力の場合：

```bash
ECC_ROOT="${CLAUDE_PLUGIN_ROOT:-$(node -e "var p=require('path'),f=require('fs'),h=require('os').homedir(),d=p.join(h,'.claude'),q=p.join('scripts','lib','utils.js');if(!f.existsSync(p.join(d,q))){try{var b=p.join(d,'plugins','cache','everything-claude-code');for(var o of f.readdirSync(b))for(var v of f.readdirSync(p.join(b,o))){var c=p.join(b,o,v);if(f.existsSync(p.join(c,q))){d=c;break}}}catch(x){}}console.log(d)")}"
node "$ECC_ROOT/scripts/skills-health.js" --dashboard --json
```

## 使い方

```
/skill-health                    # Full dashboard view
/skill-health --panel failures   # Only failure clustering panel
/skill-health --json             # Machine-readable JSON output
```

## 実行手順

1. --dashboardフラグ付きでskills-health.jsスクリプトを実行
2. 出力をユーザーに表示
3. 低下しているスキルがある場合、ハイライトして/evolveの実行を提案
4. 保留中の修正提案がある場合、レビューを提案

## パネル

- **成功率（30日間）** — スキルごとの日次成功率のスパークラインチャート
- **障害パターン** — クラスタリングされた障害理由の横棒グラフ
- **保留中の修正提案** — レビュー待ちの修正提案
- **バージョン履歴** — スキルごとのバージョンスナップショットのタイムライン
