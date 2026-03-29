# ルール
## 構成

ルールは**共通**レイヤーと**言語固有**のディレクトリに整理されています：

```
rules/
├── common/          # 言語非依存の原則（常にインストール）
│   ├── coding-style.md
│   ├── git-workflow.md
│   ├── testing.md
│   ├── performance.md
│   ├── patterns.md
│   ├── hooks.md
│   ├── agents.md
│   └── security.md
├── typescript/      # TypeScript/JavaScript固有
├── python/          # Python固有
├── golang/          # Go固有
├── swift/           # Swift固有
└── php/             # PHP固有
```

- **common/**には普遍的な原則が含まれています — 言語固有のコード例はありません。
- **言語ディレクトリ**は、フレームワーク固有のパターン、ツール、コード例で共通ルールを拡張します。各ファイルは対応する共通ルールを参照しています。

## インストール

### オプション1：インストールスクリプト（推奨）

```bash
# Install common + one or more language-specific rule sets
./install.sh typescript
./install.sh python
./install.sh golang
./install.sh swift
./install.sh php

# Install multiple languages at once
./install.sh typescript python
```

### オプション2：手動インストール

> **重要：** ディレクトリ全体をコピーしてください — `/*`でフラット化しないでください。
> 共通ディレクトリと言語固有ディレクトリには同名のファイルが含まれています。
> 1つのディレクトリにフラット化すると、言語固有ファイルが共通ルールを上書きし、
> 言語固有ファイルが使用する相対`../common/`参照が壊れます。

```bash
# Install common rules (required for all projects)
cp -r rules/common ~/.claude/rules/common

# Install language-specific rules based on your project's tech stack
cp -r rules/typescript ~/.claude/rules/typescript
cp -r rules/python ~/.claude/rules/python
cp -r rules/golang ~/.claude/rules/golang
cp -r rules/swift ~/.claude/rules/swift
cp -r rules/php ~/.claude/rules/php

# Attention ! ! ! Configure according to your actual project requirements; the configuration here is for reference only.
```

## ルールとスキルの違い

- **ルール**は、広く適用される基準、規約、チェックリストを定義します（例：「テストカバレッジ80%」、「ハードコードされたシークレット禁止」）。
- **スキル**（`skills/`ディレクトリ）は、特定のタスクに対する詳細で実行可能な参考資料を提供します（例：`python-patterns`、`golang-testing`）。

言語固有のルールファイルは、適切な場合に関連するスキルを参照します。ルールは*何をすべきか*を示し、スキルは*どのように行うか*を示します。

## 新しい言語の追加

新しい言語のサポートを追加するには（例：`rust/`）：

1. `rules/rust/`ディレクトリを作成します
2. 共通ルールを拡張するファイルを追加します：
   - `coding-style.md` — フォーマットツール、イディオム、エラーハンドリングパターン
   - `testing.md` — テストフレームワーク、カバレッジツール、テスト構成
   - `patterns.md` — 言語固有のデザインパターン
   - `hooks.md` — フォーマッター、リンター、型チェッカー用のPostToolUseフック
   - `security.md` — シークレット管理、セキュリティスキャンツール
3. 各ファイルは以下で始めてください：
   ```
   > This file extends [common/xxx.md](../common/xxx.md) with <Language> specific content.
   ```
4. 利用可能な場合は既存のスキルを参照するか、`skills/`配下に新しいスキルを作成してください。

## ルールの優先順位

言語固有のルールと共通ルールが競合する場合、**言語固有のルールが優先**されます（具体的なものが一般的なものをオーバーライド）。これはCSSの詳細度や`.gitignore`の優先順位と同様の、標準的なレイヤード構成パターンに従っています。

- `rules/common/`は、すべてのプロジェクトに適用される普遍的なデフォルトを定義します。
- `rules/golang/`、`rules/python/`、`rules/swift/`、`rules/php/`、`rules/typescript/`などは、言語のイディオムが異なる場合にそれらのデフォルトをオーバーライドします。

### 例

`common/coding-style.md`はデフォルトの原則としてイミュータビリティを推奨しています。言語固有の`golang/coding-style.md`はこれをオーバーライドできます：

> イディオマティックなGoでは構造体の変更にポインタレシーバを使用します — 一般的な原則については[common/coding-style.md](../common/coding-style.md)を参照してください。ただし、ここではGo固有のイディオマティックな変更が優先されます。

### オーバーライドに関する注記がある共通ルール

`rules/common/`内のルールで、言語固有のファイルによってオーバーライドされる可能性があるものには以下のマークが付いています：

> **言語に関する注記**: このルールは、このパターンがイディオマティックでない言語の言語固有ルールによってオーバーライドされる場合があります。
