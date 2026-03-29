---
name: click-path-audit
description: "すべてのユーザー向けボタン/タッチポイントを完全なステート変更シーケンスを通してトレースし、個々の関数は正常に動作するが互いに打ち消し合ったり、誤った最終ステートを生成したり、UI を不整合な状態にするバグを発見する。使用タイミング：体系的なデバッグでバグが見つからないがユーザーがボタンの不具合を報告している場合、または共有ステートストアに触れる大規模リファクタリング後。"
origin: community
---

# /click-path-audit — 動作フロー監査

静的なコード読解では見逃すバグを発見する：ステート間の相互作用による副作用、連続呼び出し間のレースコンディション、互いのハンドラが暗黙的に取り消し合う問題。

## このスキルが解決する問題

従来のデバッグで確認すること：
- 関数は存在するか？（接続漏れ）
- クラッシュするか？（ランタイムエラー）
- 正しい型を返すか？（データフロー）

しかし確認しないこと：
- **最終的な UI ステートはボタンラベルが約束する内容と一致するか？**
- **関数 B が関数 A の実行結果を暗黙的に取り消していないか？**
- **共有ステート（Zustand/Redux/context）に意図したアクションを打ち消す副作用がないか？**

実例：「新規メール」ボタンが `setComposeMode(true)` を呼び出し、次に `selectThread(null)` を呼び出していた。両方とも個別には正常に動作した。しかし `selectThread` には `composeMode: false` にリセットする副作用があった。ボタンは何もしなかった。体系的デバッグで 54 件のバグが見つかったが、この 1 件は見逃された。

---

## 仕組み

対象領域のすべてのインタラクティブなタッチポイントに対して：

```
1. IDENTIFY the handler (onClick, onSubmit, onChange, etc.)
2. TRACE every function call in the handler, IN ORDER
3. For EACH function call:
   a. What state does it READ?
   b. What state does it WRITE?
   c. Does it have SIDE EFFECTS on shared state?
   d. Does it reset/clear any state as a side effect?
4. CHECK: Does any later call UNDO a state change from an earlier call?
5. CHECK: Is the FINAL state what the user expects from the button label?
6. CHECK: Are there race conditions (async calls that resolve in wrong order)?
```

---

## 実行ステップ

### ステップ 1：ステートストアのマッピング

タッチポイントを監査する前に、すべてのステートストアアクションの副作用マップを構築する：

```
For each Zustand store / React context in scope:
  For each action/setter:
    - What fields does it set?
    - Does it RESET other fields as a side effect?
    - Document: actionName → {sets: [...], resets: [...]}
```

これが重要なリファレンスとなる。「新規メール」のバグは `selectThread` が `composeMode` をリセットすることを知らなければ見えなかった。

**出力形式：**
```
STORE: emailStore
  setComposeMode(bool) → sets: {composeMode}
  selectThread(thread|null) → sets: {selectedThread, selectedThreadId, messages, drafts, selectedDraft, summary} RESETS: {composeMode: false, composeData: null, redraftOpen: false}
  setDraftGenerating(bool) → sets: {draftGenerating}
  ...

DANGEROUS RESETS (actions that clear state they don't own):
  selectThread → resets composeMode (owned by setComposeMode)
  reset → resets everything
```

### ステップ 2：各タッチポイントの監査

対象領域の各ボタン/トグル/フォーム送信に対して：

```
TOUCHPOINT: [Button label] in [Component:line]
  HANDLER: onClick → {
    call 1: functionA() → sets {X: true}
    call 2: functionB() → sets {Y: null} RESETS {X: false}  ← CONFLICT
  }
  EXPECTED: User sees [description of what button label promises]
  ACTUAL: X is false because functionB reset it
  VERDICT: BUG — [description]
```

**以下の各バグパターンを確認：**

#### パターン 1：連続的な取り消し
```
handler() {
  setState_A(true)     // sets X = true
  setState_B(null)     // side effect: resets X = false
}
// Result: X is false. First call was pointless.
```

#### パターン 2：非同期レース
```
handler() {
  fetchA().then(() => setState({ loading: false }))
  fetchB().then(() => setState({ loading: true }))
}
// Result: final loading state depends on which resolves first
```

#### パターン 3：ステールクロージャ
```
const [count, setCount] = useState(0)
const handler = useCallback(() => {
  setCount(count + 1)  // captures stale count
  setCount(count + 1)  // same stale count — increments by 1, not 2
}, [count])
```

#### パターン 4：ステート遷移の欠落
```
// Button says "Save" but handler only validates, never actually saves
// Button says "Delete" but handler sets a flag without calling the API
// Button says "Send" but the API endpoint is removed/broken
```

#### パターン 5：条件付きデッドパス
```
handler() {
  if (someState) {        // someState is ALWAYS false at this point
    doTheActualThing()    // never reached
  }
}
```

#### パターン 6：useEffect 干渉
```
// Button sets stateX = true
// A useEffect watches stateX and resets it to false
// User sees nothing happen
```

### ステップ 3：レポート

発見された各バグに対して：

```
CLICK-PATH-NNN: [severity: CRITICAL/HIGH/MEDIUM/LOW]
  Touchpoint: [Button label] in [file:line]
  Pattern: [Sequential Undo / Async Race / Stale Closure / Missing Transition / Dead Path / useEffect Interference]
  Handler: [function name or inline]
  Trace:
    1. [call] → sets {field: value}
    2. [call] → RESETS {field: value}  ← CONFLICT
  Expected: [what user expects]
  Actual: [what actually happens]
  Fix: [specific fix]
```

---

## スコープ制御

この監査はコストが高い。適切にスコープすること：

- **アプリ全体の監査：** ローンチ時または大規模リファクタリング後に使用。ページごとに並列エージェントを起動。
- **単一ページの監査：** 新しいページの構築後、またはユーザーがボタンの不具合を報告した後に使用。
- **ストア集中型の監査：** Zustand ストアを変更した後に使用 — 変更されたアクションのすべてのコンシューマーを監査。

### アプリ全体の推奨エージェント分割：

```
Agent 1: Map ALL state stores (Step 1) — this is shared context for all other agents
Agent 2: Dashboard (Tasks, Notes, Journal, Ideas)
Agent 3: Chat (DanteChatColumn, JustChatPage)
Agent 4: Emails (ThreadList, DraftArea, EmailsPage)
Agent 5: Projects (ProjectsPage, ProjectOverviewTab, NewProjectWizard)
Agent 6: CRM (all sub-tabs)
Agent 7: Profile, Settings, Vault, Notifications
Agent 8: Management Suite (all pages)
```

Agent 1 が最初に完了しなければならない。その出力は他のすべてのエージェントの入力となる。

---

## いつ使うか

- 体系的なデバッグで「バグなし」となったが、ユーザーが UI の不具合を報告している場合
- Zustand ストアのアクションを変更した後（すべての呼び出し元を確認）
- 共有ステートに触れるリファクタリング後
- リリース前の重要なユーザーフロー
- ボタンが「何も起きない」場合 — これがまさにそのためのツール

## いつ使わないか

- API レベルのバグ（不正なレスポンス形式、エンドポイントの欠落）— systematic-debugging を使用
- スタイリング/レイアウトの問題 — 目視確認
- パフォーマンスの問題 — プロファイリングツール

---

## 他のスキルとの連携

- `/superpowers:systematic-debugging`（他の 54 種類のバグを見つける）の後に実行
- `/superpowers:verification-before-completion`（修正が機能することを検証する）の前に実行
- `/superpowers:test-driven-development` にフィードバック — ここで見つかったすべてのバグにテストを追加すべき

---

## 例：このスキルのきっかけとなったバグ

**ThreadList.tsx の「新規メール」ボタン：**
```
onClick={() => {
  useEmailStore.getState().setComposeMode(true)   // ✓ sets composeMode = true
  useEmailStore.getState().selectThread(null)      // ✗ RESETS composeMode = false
}}
```

ストア定義：
```
selectThread: (thread) => set({
  selectedThread: thread,
  selectedThreadId: thread?.id ?? null,
  messages: [],
  drafts: [],
  selectedDraft: null,
  summary: null,
  composeMode: false,     // ← THIS silent reset killed the button
  composeData: null,
  redraftOpen: false,
})
```

**体系的デバッグが見逃した理由：**
- ボタンには onClick ハンドラがある（デッドではない）
- 両方の関数が存在する（接続漏れなし）
- どちらの関数もクラッシュしない（ランタイムエラーなし）
- データ型は正しい（型の不一致なし）

**クリックパス監査が検出する理由：**
- ステップ 1 で `selectThread` が `composeMode` をリセットすることをマッピング
- ステップ 2 でハンドラをトレース：呼び出し 1 で true を設定、呼び出し 2 で false にリセット
- 判定：連続的な取り消し — 最終ステートがボタンの意図と矛盾
