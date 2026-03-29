---
description: 現在のセッション状態を~/.claude/session-data/内の日付付きファイルに保存し、将来のセッションで完全なコンテキストとともに作業を再開できるようにします。
---

# セッション保存コマンド

このセッションで起きたことすべて — 何が構築されたか、何がうまくいったか、何が失敗したか、何が残っているか — をキャプチャし、次のセッションがこのセッションの終了箇所から正確に再開できるように日付付きファイルに書き込みます。

## 使用するタイミング

- Claude Codeを閉じる前の作業セッション終了時
- コンテキスト制限に達する前（まずこれを実行してから、新しいセッションを開始）
- 記憶しておきたい複雑な問題を解決した後
- 将来のセッションにコンテキストを引き継ぐ必要があるとき

## プロセス

### ステップ1：コンテキストの収集

ファイルを書き込む前に、以下を収集します：

- このセッション中に変更されたすべてのファイルを読み取る（git diffを使用するか会話から思い出す）
- 議論、試行、決定されたことをレビュー
- 発生したエラーとその解決方法（または未解決の場合）を記録
- 関連する場合、現在のテスト/ビルドステータスを確認

### ステップ2：セッションフォルダの作成（存在しない場合）

ユーザーのClaudeホームディレクトリに正規のセッションフォルダを作成します：

```bash
mkdir -p ~/.claude/session-data
```

### ステップ3：セッションファイルの書き込み

`~/.claude/session-data/YYYY-MM-DD-<short-id>-session.tmp`を作成します。今日の実際の日付と、`session-manager.js`の`SESSION_FILENAME_REGEX`で定められたルールを満たすショートIDを使用します：

- 互換性文字：英字`a-z` / `A-Z`、数字`0-9`、ハイフン`-`、アンダースコア`_`
- 互換性最小長：1文字
- 新規ファイルの推奨スタイル：小文字英字、数字、ハイフンで8文字以上（衝突回避のため）

有効な例：`abc123de`、`a1b2c3d4`、`frontend-worktree-1`、`ChezMoi_2`
新規ファイルでは避ける：`A`、`test_id1`、`ABC123de`

完全な有効ファイル名の例：`2024-01-15-abc123de-session.tmp`

レガシーファイル名`YYYY-MM-DD-session.tmp`も有効ですが、新しいセッションファイルでは同日の衝突を避けるためにショートID形式を推奨します。

### ステップ4：以下のすべてのセクションを記入

すべてのセクションを正直に記入してください。セクションをスキップしない — 本当に内容がない場合は「特になし」または「N/A」と記述してください。不完全なファイルは、正直な空セクションよりも悪いです。

### ステップ5：ユーザーにファイルを表示

書き込み後、完全な内容を表示して以下を尋ねます：

```
Session saved to [actual resolved path to the session file]

Does this look accurate? Anything to correct or add before we close?
```

確認を待ちます。リクエストがあれば編集します。

---

## セッションファイルフォーマット

```markdown
# Session: YYYY-MM-DD

**Started:** [approximate time if known]
**Last Updated:** [current time]
**Project:** [project name or path]
**Topic:** [one-line summary of what this session was about]

---

## What We Are Building

[1-3 paragraphs describing the feature, bug fix, or task. Include enough
context that someone with zero memory of this session can understand the goal.
Include: what it does, why it's needed, how it fits into the larger system.]

---

## What WORKED (with evidence)

[List only things that are confirmed working. For each item include WHY you
know it works — test passed, ran in browser, Postman returned 200, etc.
Without evidence, move it to "Not Tried Yet" instead.]

- **[thing that works]** — confirmed by: [specific evidence]
- **[thing that works]** — confirmed by: [specific evidence]

If nothing is confirmed working yet: "Nothing confirmed working yet — all approaches still in progress or untested."

---

## What Did NOT Work (and why)

[This is the most important section. List every approach tried that failed.
For each failure write the EXACT reason so the next session doesn't retry it.
Be specific: "threw X error because Y" is useful. "didn't work" is not.]

- **[approach tried]** — failed because: [exact reason / error message]
- **[approach tried]** — failed because: [exact reason / error message]

If nothing failed: "No failed approaches yet."

---

## What Has NOT Been Tried Yet

[Approaches that seem promising but haven't been attempted. Ideas from the
conversation. Alternative solutions worth exploring. Be specific enough that
the next session knows exactly what to try.]

- [approach / idea]
- [approach / idea]

If nothing is queued: "No specific untried approaches identified."

---

## Current State of Files

[Every file touched this session. Be precise about what state each file is in.]

| File              | Status         | Notes                      |
| ----------------- | -------------- | -------------------------- |
| `path/to/file.ts` | ✅ Complete    | [what it does]             |
| `path/to/file.ts` | 🔄 In Progress | [what's done, what's left] |
| `path/to/file.ts` | ❌ Broken      | [what's wrong]             |
| `path/to/file.ts` | 🗒️ Not Started | [planned but not touched]  |

If no files were touched: "No files modified this session."

---

## Decisions Made

[Architecture choices, tradeoffs accepted, approaches chosen and why.
These prevent the next session from relitigating settled decisions.]

- **[decision]** — reason: [why this was chosen over alternatives]

If no significant decisions: "No major decisions made this session."

---

## Blockers & Open Questions

[Anything unresolved that the next session needs to address or investigate.
Questions that came up but weren't answered. External dependencies waiting on.]

- [blocker / open question]

If none: "No active blockers."

---

## Exact Next Step

[If known: The single most important thing to do when resuming. Be precise
enough that resuming requires zero thinking about where to start.]

[If not known: "Next step not determined — review 'What Has NOT Been Tried Yet'
and 'Blockers' sections to decide on direction before starting."]

---

## Environment & Setup Notes

[Only fill this if relevant — commands needed to run the project, env vars
required, services that need to be running, etc. Skip if standard setup.]

[If none: omit this section entirely.]
```

---

## 出力例

```markdown
# Session: 2024-01-15

**Started:** ~2pm
**Last Updated:** 5:30pm
**Project:** my-app
**Topic:** Building JWT authentication with httpOnly cookies

---

## What We Are Building

User authentication system for the Next.js app. Users register with email/password,
receive a JWT stored in an httpOnly cookie (not localStorage), and protected routes
check for a valid token via middleware. The goal is session persistence across browser
refreshes without exposing the token to JavaScript.

---

## What WORKED (with evidence)

- **`/api/auth/register` endpoint** — confirmed by: Postman POST returns 200 with user
  object, row visible in Supabase dashboard, bcrypt hash stored correctly
- **JWT generation in `lib/auth.ts`** — confirmed by: unit test passes
  (`npm test -- auth.test.ts`), decoded token at jwt.io shows correct payload
- **Password hashing** — confirmed by: `bcrypt.compare()` returns true in test

---

## What Did NOT Work (and why)

- **Next-Auth library** — failed because: conflicts with our custom Prisma adapter,
  threw "Cannot use adapter with credentials provider in this configuration" on every
  request. Not worth debugging — too opinionated for our setup.
- **Storing JWT in localStorage** — failed because: SSR renders happen before
  localStorage is available, caused React hydration mismatch error on every page load.
  This approach is fundamentally incompatible with Next.js SSR.

---

## What Has NOT Been Tried Yet

- Store JWT as httpOnly cookie in the login route response (most likely solution)
- Use `cookies()` from `next/headers` to read token in server components
- Write middleware.ts to protect routes by checking cookie existence

---

## Current State of Files

| File                             | Status         | Notes                                           |
| -------------------------------- | -------------- | ----------------------------------------------- |
| `app/api/auth/register/route.ts` | ✅ Complete    | Works, tested                                   |
| `app/api/auth/login/route.ts`    | 🔄 In Progress | Token generates but not setting cookie yet      |
| `lib/auth.ts`                    | ✅ Complete    | JWT helpers, all tested                         |
| `middleware.ts`                  | 🗒️ Not Started | Route protection, needs cookie read logic first |
| `app/login/page.tsx`             | 🗒️ Not Started | UI not started                                  |

---

## Decisions Made

- **httpOnly cookie over localStorage** — reason: prevents XSS token theft, works with SSR
- **Custom auth over Next-Auth** — reason: Next-Auth conflicts with our Prisma setup, not worth the fight

---

## Blockers & Open Questions

- Does `cookies().set()` work inside a Route Handler or only in Server Actions? Need to verify.

---

## Exact Next Step

In `app/api/auth/login/route.ts`, after generating the JWT, set it as an httpOnly
cookie using `cookies().set('token', jwt, { httpOnly: true, secure: true, sameSite: 'strict' })`.
Then test with Postman — the response should include a `Set-Cookie` header.
```

---

## 備考

- 各セッションは独自のファイルを持つ — 以前のセッションファイルには追記しない
- 「うまくいかなかったこと」セクションが最も重要 — これがないと将来のセッションが失敗したアプローチを盲目的に再試行する
- ユーザーがセッション途中（終了時だけでなく）に保存を依頼した場合、現時点で分かっていることを保存し、進行中のアイテムを明確にマーク
- このファイルは次のセッションの開始時に`/resume-session`経由でClaudeに読み取られることを意図している
- 正規のグローバルセッションストアを使用：`~/.claude/session-data/`
- 新しいセッションファイルにはショートIDファイル名形式（`YYYY-MM-DD-<short-id>-session.tmp`）を推奨
