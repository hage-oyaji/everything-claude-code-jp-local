# フック

フックは、Claude Codeのツール実行前後に発火するイベント駆動型の自動化です。コード品質の強制、ミスの早期発見、反復的なチェックの自動化を実現します。

> **注意:** 以前はフック定義を `hooks.json` に記述していましたが、現在は [`.claude/settings.json`](../settings.json) の `"hooks"` キーに統合されています。このディレクトリの `README.md` は人間向けのドキュメントです。

## フックの仕組み

```
ユーザーリクエスト → Claudeがツールを選択 → PreToolUseフックが実行 → ツール実行 → PostToolUseフックが実行
```

- **PreToolUse**フックはツール実行前に実行されます。**ブロック**（終了コード2）または**警告**（stderrへの出力、ブロックなし）が可能です。
- **PostToolUse**フックはツール完了後に実行されます。出力の分析は可能ですが、ブロックはできません。
- **Stop**フックはClaudeの各レスポンス後に実行されます。
- **SessionStart/SessionEnd**フックはセッションのライフサイクル境界で実行されます。
- **PreCompact**フックはコンテキスト圧縮前に実行され、状態の保存に便利です。

## このプラグインのフック

### PreToolUseフック

| フック | マッチャー | 動作 | 終了コード |
|------|---------|----------|-----------|
| **開発サーバーブロッカー** | `Bash` | tmux外での`npm run dev`等をブロック — ログアクセスを確保 | 2（ブロック） |
| **tmuxリマインダー** | `Bash` | 長時間実行コマンド（npm test、cargo build、docker）にtmuxの使用を提案 | 0（警告） |
| **Git pushリマインダー** | `Bash` | `git push`前に変更のレビューをリマインド | 0（警告） |
| **ドキュメントファイル警告** | `Write` | 非標準の`.md`/`.txt`ファイルについて警告（README、CLAUDE、CONTRIBUTING、CHANGELOG、LICENSE、SKILL、docs/、skills/は許可）；クロスプラットフォームパス処理対応 | 0（警告） |
| **戦略的コンパクト** | `Edit\|Write` | 論理的な間隔（約50ツール呼び出しごと）で手動`/compact`を提案 | 0（警告） |
| **InsAItsセキュリティモニター（オプトイン）** | `Bash\|Write\|Edit\|MultiEdit` | 高信頼性ツール入力に対するオプションのセキュリティスキャン。`ECC_ENABLE_INSAITS=1`が設定されていない限り無効。クリティカルな検出でブロック、非クリティカルで警告し、監査ログを`.insaits_audit_session.jsonl`に書き込み。`pip install insa-its`が必要。[詳細](../scripts/hooks/insaits-security-monitor.py) | 2（クリティカルをブロック） / 0（警告） |

### PostToolUseフック

| フック | マッチャー | 動作内容 |
|------|---------|-------------|
| **PRロガー** | `Bash` | `gh pr create`後にPR URLとレビューコマンドをログ |
| **ビルド分析** | `Bash` | ビルドコマンド後にバックグラウンド分析（非同期、ノンブロッキング） |
| **品質ゲート** | `Edit\|Write\|MultiEdit` | 編集後に高速な品質チェックを実行 |
| **Prettierフォーマット** | `Edit` | 編集後にPrettierでJS/TSファイルを自動フォーマット |
| **TypeScriptチェック** | `Edit` | `.ts`/`.tsx`ファイル編集後に`tsc --noEmit`を実行 |
| **console.log警告** | `Edit` | 編集されたファイル内の`console.log`文について警告 |

### ライフサイクルフック

| フック | イベント | 動作内容 |
|------|-------|-------------|
| **セッション開始** | `SessionStart` | 前回のコンテキストを読み込み、パッケージマネージャーを検出 |
| **プリコンパクト** | `PreCompact` | コンテキスト圧縮前に状態を保存 |
| **Console.log監査** | `Stop` | 各レスポンス後にすべての変更ファイルの`console.log`をチェック |
| **セッションサマリー** | `Stop` | トランスクリプトパスが利用可能な場合にセッション状態を永続化 |
| **パターン抽出** | `Stop` | 抽出可能なパターンについてセッションを評価（継続的学習） |
| **コストトラッカー** | `Stop` | 軽量な実行コストテレメトリマーカーを出力 |
| **デスクトップ通知** | `Stop` | タスクサマリーのmacOSデスクトップ通知を送信（standard+） |
| **セッション終了マーカー** | `SessionEnd` | ライフサイクルマーカーとクリーンアップログ |

## フックのカスタマイズ

### フックの無効化

`.claude/settings.json`の`"hooks"`セクション内のフックエントリを削除またはコメントアウトします。プラグインとしてインストールされている場合は、`~/.claude/settings.json`でオーバーライドできます：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [],
        "description": "Override: allow all .md file creation"
      }
    ]
  }
}
```

### ランタイムフック制御（推奨）

`settings.json`を編集せずに、環境変数でフックの動作を制御できます：

```bash
# minimal | standard | strict (default: standard)
export ECC_HOOK_PROFILE=standard

# Disable specific hook IDs (comma-separated)
export ECC_DISABLED_HOOKS="pre:bash:tmux-reminder,post:edit:typecheck"
```

プロファイル：
- `minimal` — 必須のライフサイクルフックと安全フックのみを維持。
- `standard` — デフォルト；品質チェックと安全チェックのバランス型。
- `strict` — 追加のリマインダーとより厳格なガードレールを有効化。

### 独自のフックを作成する

フックはstdinでJSON形式のツール入力を受け取り、stdoutにJSONを出力するシェルコマンドです。

**基本構造：**

```javascript
// my-hook.js
let data = '';
process.stdin.on('data', chunk => data += chunk);
process.stdin.on('end', () => {
  const input = JSON.parse(data);

  // Access tool info
  const toolName = input.tool_name;        // "Edit", "Bash", "Write", etc.
  const toolInput = input.tool_input;      // Tool-specific parameters
  const toolOutput = input.tool_output;    // Only available in PostToolUse

  // Warn (non-blocking): write to stderr
  console.error('[Hook] Warning message shown to Claude');

  // Block (PreToolUse only): exit with code 2
  // process.exit(2);

  // Always output the original data to stdout
  console.log(data);
});
```

**終了コード：**
- `0` — 成功（実行を続行）
- `2` — ツール呼び出しをブロック（PreToolUseのみ）
- その他の非ゼロ — エラー（ログに記録されるがブロックしない）

### フック入力スキーマ

```typescript
interface HookInput {
  tool_name: string;          // "Bash", "Edit", "Write", "Read", etc.
  tool_input: {
    command?: string;         // Bash: the command being run
    file_path?: string;       // Edit/Write/Read: target file
    old_string?: string;      // Edit: text being replaced
    new_string?: string;      // Edit: replacement text
    content?: string;         // Write: file content
  };
  tool_output?: {             // PostToolUse only
    output?: string;          // Command/tool output
  };
}
```

### 非同期フック

メインフローをブロックすべきでないフック（例：バックグラウンド分析）の場合：

```json
{
  "type": "command",
  "command": "node my-slow-hook.js",
  "async": true,
  "timeout": 30
}
```

非同期フックはバックグラウンドで実行されます。ツール実行をブロックすることはできません。

## よくあるフックレシピ

### TODOコメントについて警告

```json
{
  "matcher": "Edit",
  "hooks": [{
    "type": "command",
    "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const ns=i.tool_input?.new_string||'';if(/TODO|FIXME|HACK/.test(ns)){console.error('[Hook] New TODO/FIXME added - consider creating an issue')}console.log(d)})\""
  }],
  "description": "Warn when adding TODO/FIXME comments"
}
```

### 大きなファイルの作成をブロック

```json
{
  "matcher": "Write",
  "hooks": [{
    "type": "command",
    "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const c=i.tool_input?.content||'';const lines=c.split('\\n').length;if(lines>800){console.error('[Hook] BLOCKED: File exceeds 800 lines ('+lines+' lines)');console.error('[Hook] Split into smaller, focused modules');process.exit(2)}console.log(d)})\""
  }],
  "description": "Block creation of files larger than 800 lines"
}
```

### ruffでPythonファイルを自動フォーマット

```json
{
  "matcher": "Edit",
  "hooks": [{
    "type": "command",
    "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const p=i.tool_input?.file_path||'';if(/\\.py$/.test(p)){const{execFileSync}=require('child_process');try{execFileSync('ruff',['format',p],{stdio:'pipe'})}catch(e){}}console.log(d)})\""
  }],
  "description": "Auto-format Python files with ruff after edits"
}
```

### 新しいソースファイルにテストファイルを要求

```json
{
  "matcher": "Write",
  "hooks": [{
    "type": "command",
    "command": "node -e \"const fs=require('fs');let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const p=i.tool_input?.file_path||'';if(/src\\/.*\\.(ts|js)$/.test(p)&&!/\\.test\\.|\\.spec\\./.test(p)){const testPath=p.replace(/\\.(ts|js)$/,'.test.$1');if(!fs.existsSync(testPath)){console.error('[Hook] No test file found for: '+p);console.error('[Hook] Expected: '+testPath);console.error('[Hook] Consider writing tests first (/tdd)')}}console.log(d)})\""
  }],
  "description": "Remind to create tests when adding new source files"
}
```

## クロスプラットフォームに関する注意事項

フックロジックはWindows、macOS、Linuxでのクロスプラットフォームな動作のためにNode.jsスクリプトで実装されています。少数のシェルラッパーが継続的学習オブザーバーフック用に保持されています。これらのラッパーはプロファイルによるゲートがかけられており、Windows向けの安全なフォールバック動作を備えています。

## 関連

- [rules/common/hooks.md](../rules/common/hooks.md) — フックアーキテクチャガイドライン
- [skills/strategic-compact/](../skills/strategic-compact/) — 戦略的コンパクションスキル
- [scripts/hooks/](../scripts/hooks/) — フックスクリプトの実装
