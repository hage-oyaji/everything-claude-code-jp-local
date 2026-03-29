---
description: ~/.claude/session-data/から最新のセッションファイルをロードし、前回のセッションが終了した箇所から完全なコンテキストで作業を再開します。
---

# セッション再開コマンド

保存された最新のセッション状態をロードし、作業を開始する前に完全に把握します。
このコマンドは`/save-session`の対になるコマンドです。

## 使用するタイミング

- 前日からの作業を継続するために新しいセッションを開始するとき
- コンテキスト制限により新しいセッションを開始した後
- 別のソースからセッションファイルを引き継ぐとき（ファイルパスを指定するだけ）
- セッションファイルがあり、Claudeに作業を進める前に完全に理解させたいとき

## 使い方

```
/resume-session                                                      # loads most recent file in ~/.claude/session-data/
/resume-session 2024-01-15                                           # loads most recent session for that date
/resume-session ~/.claude/session-data/2024-01-15-abc123de-session.tmp  # loads a current short-id session file
/resume-session ~/.claude/sessions/2024-01-15-session.tmp               # loads a specific legacy-format file
```

## プロセス

### ステップ1：セッションファイルを見つける

引数が指定されていない場合：

1. `~/.claude/session-data/`を確認
2. 最新の`*-session.tmp`ファイルを選択
3. フォルダが存在しないか一致するファイルがない場合、ユーザーに通知：
   ```
   No session files found in ~/.claude/session-data/
   Run /save-session at the end of a session to create one.
   ```
   その後停止。

引数が指定されている場合：

- 日付形式（`YYYY-MM-DD`）の場合、まず`~/.claude/session-data/`を検索し、次にレガシーの`~/.claude/sessions/`で、`YYYY-MM-DD-session.tmp`（レガシー形式）または`YYYY-MM-DD-<shortid>-session.tmp`（現在の形式）に一致するファイルを探し、その日付の最新ファイルをロード
- ファイルパスの場合、そのファイルを直接読み取る
- 見つからない場合、明確に報告して停止

### ステップ2：セッションファイル全体を読み取る

ファイル全体を読み取ります。まだ要約しないでください。

### ステップ3：理解の確認

以下の正確なフォーマットで構造化されたブリーフィングを応答します：

```
SESSION LOADED: [actual resolved path to the file]
════════════════════════════════════════════════

PROJECT: [project name / topic from file]

WHAT WE'RE BUILDING:
[2-3 sentence summary in your own words]

CURRENT STATE:
✅ Working: [count] items confirmed
🔄 In Progress: [list files that are in progress]
🗒️ Not Started: [list planned but untouched]

WHAT NOT TO RETRY:
[list every failed approach with its reason — this is critical]

OPEN QUESTIONS / BLOCKERS:
[list any blockers or unanswered questions]

NEXT STEP:
[exact next step if defined in the file]
[if not defined: "No next step defined — recommend reviewing 'What Has NOT Been Tried Yet' together before starting"]

════════════════════════════════════════════════
Ready to continue. What would you like to do?
```

### ステップ4：ユーザーを待つ

自動的に作業を開始しないでください。ファイルに触れないでください。ユーザーの指示を待ってください。

次のステップがセッションファイルに明確に定義されており、ユーザーが「続けて」「はい」などと言った場合 — その正確な次のステップに進みます。

次のステップが定義されていない場合 — ユーザーにどこから始めるか尋ね、オプションで「まだ試していないこと」セクションからアプローチを提案します。

---

## エッジケース

**同じ日付に複数のセッション**（`2024-01-15-session.tmp`、`2024-01-15-abc123de-session.tmp`）：
レガシーのIDなし形式か現在のショートID形式かに関係なく、その日付の最新のファイルをロードします。

**セッションファイルが存在しないファイルを参照している場合：**
ブリーフィング中に記載 — 「⚠️ `path/to/file.ts`がセッションで参照されていますが、ディスク上に見つかりません。」

**セッションファイルが7日以上前の場合：**
差分を記載 — 「⚠️ このセッションはN日前のものです（しきい値：7日）。状況が変わっている可能性があります。」 — その後通常通り進行。

**ユーザーがファイルパスを直接指定した場合（例：チームメイトから転送）：**
読み取って同じブリーフィングプロセスに従います — ソースに関係なくフォーマットは同じです。

**セッションファイルが空または不正な形式の場合：**
報告：「セッションファイルが見つかりましたが、空または読み取り不能のようです。`/save-session`で新しいファイルを作成する必要があるかもしれません。」

---

## 出力例

```
SESSION LOADED: /Users/you/.claude/session-data/2024-01-15-abc123de-session.tmp
════════════════════════════════════════════════

PROJECT: my-app — JWT Authentication

WHAT WE'RE BUILDING:
User authentication with JWT tokens stored in httpOnly cookies.
Register and login endpoints are partially done. Route protection
via middleware hasn't been started yet.

CURRENT STATE:
✅ Working: 3 items (register endpoint, JWT generation, password hashing)
🔄 In Progress: app/api/auth/login/route.ts (token works, cookie not set yet)
🗒️ Not Started: middleware.ts, app/login/page.tsx

WHAT NOT TO RETRY:
❌ Next-Auth — conflicts with custom Prisma adapter, threw adapter error on every request
❌ localStorage for JWT — causes SSR hydration mismatch, incompatible with Next.js

OPEN QUESTIONS / BLOCKERS:
- Does cookies().set() work inside a Route Handler or only Server Actions?

NEXT STEP:
In app/api/auth/login/route.ts — set the JWT as an httpOnly cookie using
cookies().set('token', jwt, { httpOnly: true, secure: true, sameSite: 'strict' })
then test with Postman for a Set-Cookie header in the response.

════════════════════════════════════════════════
Ready to continue. What would you like to do?
```

---

## 備考

- セッションファイルをロードする際に変更しないこと — 読み取り専用の履歴記録です
- ブリーフィングのフォーマットは固定 — 空でもセクションをスキップしないこと
- 「再試行しないこと」は常に表示する必要があります。「なし」と書くだけでも — 見落とすには重要すぎます
- 再開後、ユーザーは新しいセッションの終了時に再度`/save-session`を実行して、新しい日付付きファイルを作成したい場合があります
