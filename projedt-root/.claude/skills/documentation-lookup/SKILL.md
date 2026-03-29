---
name: documentation-lookup
description: トレーニングデータの代わりにContext7 MCPを使用して最新のライブラリやフレームワークのドキュメントを取得。セットアップの質問、APIリファレンス、コード例、またはユーザーがフレームワーク名（React、Next.js、Prismaなど）を指定した場合にアクティベート。
origin: ECC
---

# ドキュメント検索（Context7）

ユーザーがライブラリ、フレームワーク、またはAPIについて質問した場合、トレーニングデータに頼らず、Context7 MCP（ツール`resolve-library-id`および`query-docs`）を通じて最新のドキュメントを取得します。

## コアコンセプト

- **Context7**: ライブドキュメントを公開するMCPサーバー。ライブラリやAPIについてはトレーニングデータの代わりにこれを使用。
- **resolve-library-id**: ライブラリ名とクエリからContext7互換のライブラリID（例: `/vercel/next.js`）を返す。
- **query-docs**: 指定されたライブラリIDと質問に対するドキュメントとコードスニペットを取得。有効なライブラリIDを取得するために、必ず先にresolve-library-idを呼び出す。

## 使用するタイミング

以下の場合にアクティベート：

- セットアップや設定に関する質問（例: 「Next.jsのミドルウェアはどう設定する？」）
- ライブラリに依存するコードのリクエスト（「Prismaでクエリを書いて...」）
- APIやリファレンス情報の要求（「Supabaseの認証メソッドは？」）
- 特定のフレームワークやライブラリの言及（React、Vue、Svelte、Express、Tailwind、Prisma、Supabaseなど）

ライブラリ、フレームワーク、またはAPIの正確で最新の動作に依存するリクエストの場合にこのスキルを使用します。Context7 MCPが設定されているハーネス（Claude Code、Cursor、Codexなど）全体で適用されます。

## 仕組み

### ステップ1: ライブラリIDの解決

**resolve-library-id** MCPツールを以下で呼び出します：

- **libraryName**: ユーザーの質問から取得したライブラリまたは製品名（例: `Next.js`、`Prisma`、`Supabase`）。
- **query**: ユーザーの完全な質問。これにより結果の関連性ランキングが向上します。

ドキュメントをクエリする前に、Context7互換のライブラリID（フォーマット`/org/project`または`/org/project/version`）を取得する必要があります。このステップからの有効なライブラリIDなしにquery-docsを呼び出さないでください。

### ステップ2: 最適な一致の選択

解決結果から、以下を使用して1つの結果を選択：

- **名前の一致**: ユーザーが求めたものに最も近い一致を優先。
- **ベンチマークスコア**: 高いスコアはドキュメント品質が高いことを示す（100が最高）。
- **ソースの信頼性**: 利用可能な場合、HighまたはMediumの信頼性を優先。
- **バージョン**: ユーザーがバージョンを指定した場合（例: 「React 19」「Next.js 15」）、リストされていればバージョン固有のライブラリID（例: `/org/project/v1.2.0`）を優先。

### ステップ3: ドキュメントの取得

**query-docs** MCPツールを以下で呼び出します：

- **libraryId**: ステップ2で選択したContext7ライブラリID（例: `/vercel/next.js`）。
- **query**: ユーザーの具体的な質問やタスク。関連するスニペットを得るために具体的に。

制限: 1つの質問につきquery-docs（またはresolve-library-id）を3回以上呼び出さないこと。3回呼び出しても回答が不明確な場合は、不確実性を述べて推測するのではなく、持っている最良の情報を使用してください。

### ステップ4: ドキュメントの使用

- 取得した最新の情報を使用してユーザーの質問に回答。
- 有用な場合、ドキュメントからの関連コード例を含める。
- 重要な場合はライブラリやバージョンを引用（例: 「Next.js 15では...」）。

## 使用例

### 例: Next.jsミドルウェア

1. **resolve-library-id**を`libraryName: "Next.js"`、`query: "How do I set up Next.js middleware?"`で呼び出す。
2. 結果から、名前とベンチマークスコアで最適な一致（例: `/vercel/next.js`）を選択。
3. **query-docs**を`libraryId: "/vercel/next.js"`、`query: "How do I set up Next.js middleware?"`で呼び出す。
4. 返されたスニペットとテキストを使用して回答。関連する場合はドキュメントからの最小限の`middleware.ts`例を含める。

### 例: Prismaクエリ

1. **resolve-library-id**を`libraryName: "Prisma"`、`query: "How do I query with relations?"`で呼び出す。
2. 公式PrismaライブラリID（例: `/prisma/prisma`）を選択。
3. そのライブラリIDとクエリで**query-docs**を呼び出す。
4. ドキュメントからの短いコードスニペットとともにPrisma Clientパターン（例: `include`または`select`）を返す。

### 例: Supabase認証メソッド

1. **resolve-library-id**を`libraryName: "Supabase"`、`query: "What are the auth methods?"`で呼び出す。
2. SupabaseドキュメントのライブラリIDを選択。
3. **query-docs**を呼び出し、認証メソッドを要約して取得したドキュメントから最小限の例を表示。

## ベストプラクティス

- **具体的に**: より良い関連性のために、可能な限りユーザーの完全な質問をクエリとして使用。
- **バージョン認識**: ユーザーがバージョンに言及する場合、解決ステップから利用可能な場合はバージョン固有のライブラリIDを使用。
- **公式ソースを優先**: 複数の一致が存在する場合、コミュニティフォークよりも公式またはプライマリパッケージを優先。
- **機密データなし**: Context7に送信するクエリからAPIキー、パスワード、トークン、その他のシークレットを削除。resolve-library-idやquery-docsに渡す前に、ユーザーの質問にシークレットが含まれている可能性があるものとして扱う。
