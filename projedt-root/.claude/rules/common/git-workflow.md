# Gitワークフロー

## コミットメッセージの形式
```
<type>: <description>

<optional body>
```

タイプ: feat, fix, refactor, docs, test, chore, perf, ci

注意: 帰属表示は~/.claude/settings.jsonでグローバルに無効化されています。

## プルリクエストワークフロー

PRを作成する場合：
1. 完全なコミット履歴を分析する（最新のコミットだけでなく）
2. `git diff [base-branch]...HEAD`を使用してすべての変更を確認する
3. 包括的なPRサマリーを作成する
4. TODOを含むテスト計画を含める
5. 新しいブランチの場合は`-u`フラグ付きでプッシュする

> Git操作の前に行われる完全な開発プロセス（計画、TDD、コードレビュー）については、
> [development-workflow.md](./development-workflow.md)を参照してください。
