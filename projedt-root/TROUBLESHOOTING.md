# トラブルシューティングガイド

Everything Claude Code（ECC）プラグインの一般的な問題と解決策です。

## 目次

- [メモリとコンテキストの問題](#メモリとコンテキストの問題)
- [エージェントハーネスの障害](#エージェントハーネスの障害)
- [フックとワークフローのエラー](#フックとワークフローのエラー)
- [インストールとセットアップ](#インストールとセットアップ)
- [パフォーマンスの問題](#パフォーマンスの問題)
- [一般的なエラーメッセージ](#一般的なエラーメッセージ)
- [ヘルプを得る](#ヘルプを得る)

---

## メモリとコンテキストの問題

### コンテキストウィンドウのオーバーフロー

**症状：** 「Context too long」エラーまたは不完全なレスポンス

**原因：**
- トークン制限を超える大きなファイルのアップロード
- 蓄積された会話履歴
- 単一セッション内での複数の大きなツール出力

**解決策：**
```bash
# 1. Clear conversation history and start fresh
# Use Claude Code: "New Chat" or Cmd/Ctrl+Shift+N

# 2. Reduce file size before analysis
head -n 100 large-file.log > sample.log

# 3. Use streaming for large outputs
head -n 50 large-file.txt

# 4. Split tasks into smaller chunks
# Instead of: "Analyze all 50 files"
# Use: "Analyze files in src/components/ directory"
```

### メモリ永続化の失敗

**症状：** エージェントが以前のコンテキストや観察を記憶しない

**原因：**
- continuous-learningフックが無効化されている
- 破損した観察ファイル
- プロジェクト検出の失敗

**解決策：**
```bash
# Check if observations are being recorded
ls ~/.claude/homunculus/projects/*/observations.jsonl

# Find the current project's hash id
python3 - <<'PY'
import json, os
registry_path = os.path.expanduser("~/.claude/homunculus/projects.json")
with open(registry_path) as f:
    registry = json.load(f)
for project_id, meta in registry.items():
    if meta.get("root") == os.getcwd():
        print(project_id)
        break
else:
    raise SystemExit("Project hash not found in ~/.claude/homunculus/projects.json")
PY

# View recent observations for that project
tail -20 ~/.claude/homunculus/projects/<project-hash>/observations.jsonl

# Back up a corrupted observations file before recreating it
mv ~/.claude/homunculus/projects/<project-hash>/observations.jsonl \
  ~/.claude/homunculus/projects/<project-hash>/observations.jsonl.bak.$(date +%Y%m%d-%H%M%S)

# Verify hooks are enabled
grep -r "observe" ~/.claude/settings.json
```

---

## エージェントハーネスの障害

### エージェントが見つからない

**症状：** 「Agent not loaded」または「Unknown agent」エラー

**原因：**
- プラグインが正しくインストールされていない
- エージェントパスの設定ミス
- MarketplaceインストールとマニュアルインストールのQ不一致

**解決策：**
```bash
# Check plugin installation
ls ~/.claude/plugins/cache/

# Verify agent exists (marketplace install)
ls ~/.claude/plugins/cache/*/agents/

# For manual install, agents should be in:
ls ~/.claude/agents/  # Custom agents only

# Reload plugin
# Claude Code → Settings → Extensions → Reload
```

### ワークフロー実行のハング

**症状：** エージェントが開始されるが完了しない

**原因：**
- エージェントロジック内の無限ループ
- ユーザー入力待ちでブロックされている
- API待ちのネットワークタイムアウト

**解決策：**
```bash
# 1. Check for stuck processes
ps aux | grep claude

# 2. Enable debug mode
export CLAUDE_DEBUG=1

# 3. Set shorter timeouts
export CLAUDE_TIMEOUT=30

# 4. Check network connectivity
curl -I https://api.anthropic.com
```

### ツール使用エラー

**症状：** 「Tool execution failed」またはアクセス拒否

**原因：**
- 依存関係の不足（npm、pythonなど）
- ファイルパーミッションの不足
- パスが見つからない

**解決策：**
```bash
# Verify required tools are installed
which node python3 npm git

# Fix permissions on hook scripts
chmod +x ~/.claude/plugins/cache/*/hooks/*.sh
chmod +x ~/.claude/plugins/cache/*/skills/*/hooks/*.sh

# Check PATH includes necessary binaries
echo $PATH
```

---

## フックとワークフローのエラー

### フックが発火しない

**症状：** Pre/Postフックが実行されない

**原因：**
- settings.jsonにフックが登録されていない
- 無効なフック構文
- フックスクリプトが実行可能でない

**解決策：**
```bash
# Check hooks are registered
grep -A 10 '"hooks"' ~/.claude/settings.json

# Verify hook files exist and are executable
ls -la ~/.claude/plugins/cache/*/hooks/

# Test hook manually
bash ~/.claude/plugins/cache/*/hooks/pre-bash.sh <<< '{"command":"echo test"}'

# Re-register hooks (if using plugin)
# Disable and re-enable plugin in Claude Code settings
```

### Python/Nodeバージョンの不一致

**症状：** 「python3 not found」または「node: command not found」

**原因：**
- Python/Nodeがインストールされていない
- PATHが設定されていない
- 誤ったPythonバージョン（Windows）

**解決策：**
```bash
# Install Python 3 (if missing)
# macOS: brew install python3
# Ubuntu: sudo apt install python3
# Windows: Download from python.org

# Install Node.js (if missing)
# macOS: brew install node
# Ubuntu: sudo apt install nodejs npm
# Windows: Download from nodejs.org

# Verify installations
python3 --version
node --version
npm --version

# Windows: Ensure python (not python3) works
python --version
```

### 開発サーバーブロッカーの誤検出

**症状：** フックが「dev」を含む正当なコマンドをブロックする

**原因：**
- ヒアドキュメントの内容がパターンマッチをトリガーしている
- 引数に「dev」が含まれる非devコマンド

**解決策：**
```bash
# This is fixed in v1.8.0+ (PR #371)
# Upgrade plugin to latest version

# Workaround: Wrap dev servers in tmux
tmux new-session -d -s dev "npm run dev"
tmux attach -t dev

# Disable hook temporarily if needed
# Edit ~/.claude/settings.json and remove pre-bash hook
```

---

## インストールとセットアップ

### プラグインが読み込まれない

**症状：** インストール後にプラグイン機能が利用できない

**原因：**
- Marketplaceキャッシュが更新されていない
- Claude Codeバージョンの非互換性
- 破損したプラグインファイル

**解決策：**
```bash
# Inspect the plugin cache before changing it
ls -la ~/.claude/plugins/cache/

# Back up the plugin cache instead of deleting it in place
mv ~/.claude/plugins/cache ~/.claude/plugins/cache.backup.$(date +%Y%m%d-%H%M%S)
mkdir -p ~/.claude/plugins/cache

# Reinstall from marketplace
# Claude Code → Extensions → Everything Claude Code → Uninstall
# Then reinstall from marketplace

# Check Claude Code version
claude --version
# Requires Claude Code 2.0+

# Manual install (if marketplace fails)
git clone https://github.com/affaan-m/everything-claude-code.git
cp -r everything-claude-code ~/.claude/plugins/ecc
```

### パッケージマネージャー検出の失敗

**症状：** 誤ったパッケージマネージャーが使用される（pnpmの代わりにnpm）

**原因：**
- ロックファイルが存在しない
- CLAUDE_PACKAGE_MANAGERが設定されていない
- 複数のロックファイルが検出を混乱させている

**解決策：**
```bash
# Set preferred package manager globally
export CLAUDE_PACKAGE_MANAGER=pnpm
# Add to ~/.bashrc or ~/.zshrc

# Or set per-project
echo '{"packageManager": "pnpm"}' > .claude/package-manager.json

# Or use package.json field
npm pkg set packageManager="pnpm@8.15.0"

# Warning: removing lock files can change installed dependency versions.
# Commit or back up the lock file first, then run a fresh install and re-run CI.
# Only do this when intentionally switching package managers.
rm package-lock.json  # If using pnpm/yarn/bun
```

---

## パフォーマンスの問題

### レスポンス時間が遅い

**症状：** エージェントの応答に30秒以上かかる

**原因：**
- 大きな観察ファイル
- アクティブなフックが多すぎる
- APIへのネットワーク遅延

**解決策：**
```bash
# Archive large observations instead of deleting them
archive_dir="$HOME/.claude/homunculus/archive/$(date +%Y%m%d)"
mkdir -p "$archive_dir"
find ~/.claude/homunculus/projects -name "observations.jsonl" -size +10M -exec sh -c '
  for file do
    base=$(basename "$(dirname "$file")")
    gzip -c "$file" > "'"$archive_dir"'/${base}-observations.jsonl.gz"
    : > "$file"
  done
' sh {} +

# Disable unused hooks temporarily
# Edit ~/.claude/settings.json

# Keep active observation files small
# Large archives should live under ~/.claude/homunculus/archive/
```

### 高CPU使用率

**症状：** Claude Codeが100%のCPUを消費する

**原因：**
- 無限の観察ループ
- 大きなディレクトリでのファイル監視
- フックのメモリリーク

**解決策：**
```bash
# Check for runaway processes
top -o cpu | grep claude

# Disable continuous learning temporarily
touch ~/.claude/homunculus/disabled

# Restart Claude Code
# Cmd/Ctrl+Q then reopen

# Check observation file size
du -sh ~/.claude/homunculus/*/
```

---

## 一般的なエラーメッセージ

### "EACCES: permission denied"

```bash
# Fix hook permissions
find ~/.claude/plugins -name "*.sh" -exec chmod +x {} \;

# Fix observation directory permissions
chmod -R u+rwX,go+rX ~/.claude/homunculus
```

### "MODULE_NOT_FOUND"

```bash
# Install plugin dependencies
cd ~/.claude/plugins/cache/everything-claude-code
npm install

# Or for manual install
cd ~/.claude/plugins/ecc
npm install
```

### "spawn UNKNOWN"

```bash
# Windows-specific: Ensure scripts use correct line endings
# Convert CRLF to LF
find ~/.claude/plugins -name "*.sh" -exec dos2unix {} \;

# Or install dos2unix
# macOS: brew install dos2unix
# Ubuntu: sudo apt install dos2unix
```

---

## ヘルプを得る

問題が解決しない場合：

1. **GitHub Issuesを確認**：[github.com/affaan-m/everything-claude-code/issues](https://github.com/affaan-m/everything-claude-code/issues)
2. **デバッグログを有効にする**：
   ```bash
   export CLAUDE_DEBUG=1
   export CLAUDE_LOG_LEVEL=debug
   ```
3. **診断情報を収集する**：
   ```bash
   claude --version
   node --version
   python3 --version
   echo $CLAUDE_PACKAGE_MANAGER
   ls -la ~/.claude/plugins/cache/
   ```
4. **Issueを作成する**：デバッグログ、エラーメッセージ、診断情報を含めてください

---

## 関連ドキュメント

- [README.md](./README.md) - インストールと機能
- [CONTRIBUTING.md](./CONTRIBUTING.md) - 開発ガイドライン
- [docs/](./docs/) - 詳細なドキュメント
- [examples/](./examples/) - 使用例
