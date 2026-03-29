---
name: mcp-server-patterns
description: Node/TypeScript SDKを使用したMCPサーバーの構築 — ツール、リソース、プロンプト、Zodバリデーション、stdio vs Streamable HTTP。最新APIについてはContext7または公式MCPドキュメントを参照。
origin: ECC
---

# MCPサーバーパターン

Model Context Protocol（MCP）は、AIアシスタントがサーバーからツールの呼び出し、リソースの読み取り、プロンプトの使用を可能にする。MCPサーバーの構築・保守時にこのスキルを使用する。SDK APIは進化するため、現在のメソッド名とシグネチャについてはContext7（「MCP」でquery-docs）または公式MCPドキュメントを確認すること。

## 使用するタイミング

新しいMCPサーバーの実装、ツールやリソースの追加、stdioとHTTPの選択、SDKのアップグレード、MCPの登録やトランスポートの問題のデバッグ時に使用する。

## 仕組み

### コアコンセプト

- **ツール**: モデルが呼び出せるアクション（例: 検索、コマンド実行）。SDKバージョンに応じて `registerTool()` または `tool()` で登録。
- **リソース**: モデルが取得できる読み取り専用データ（例: ファイル内容、APIレスポンス）。`registerResource()` または `resource()` で登録。ハンドラーは通常 `uri` 引数を受け取る。
- **プロンプト**: クライアントが表示できる再利用可能なパラメータ化されたプロンプトテンプレート（例: Claude Desktop内）。`registerPrompt()` または同等のメソッドで登録。
- **トランスポート**: ローカルクライアント（例: Claude Desktop）にはstdio、リモート（Cursor、クラウド）にはStreamable HTTPが推奨。レガシーHTTP/SSEは後方互換性のため。

Node/TypeScript SDKは `tool()` / `resource()` または `registerTool()` / `registerResource()` を公開する場合がある。公式SDKは時間とともに変更されている。常に現在の[MCPドキュメント](https://modelcontextprotocol.io)またはContext7で確認すること。

### stdioでの接続

ローカルクライアントの場合、stdioトランスポートを作成し、サーバーのconnectメソッドに渡す。正確なAPIはSDKバージョンによって異なる（例: コンストラクタ vs ファクトリー）。現在のパターンについては公式MCPドキュメントまたはContext7で「MCP stdio server」をクエリすること。

サーバーロジック（ツール + リソース）をトランスポートから独立させ、エントリーポイントでstdioまたはHTTPを接続できるようにする。

### リモート（Streamable HTTP）

Cursor、クラウド、またはその他のリモートクライアントの場合、**Streamable HTTP**（現在の仕様に基づく単一のMCP HTTPエンドポイント）を使用する。後方互換性が必要な場合のみレガシーHTTP/SSEをサポートする。

## 使用例

### インストールとサーバーセットアップ

```bash
npm install @modelcontextprotocol/sdk zod
```

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const server = new McpServer({ name: "my-server", version: "1.0.0" });
```

SDKバージョンが提供するAPIを使用してツールとリソースを登録する: 一部のバージョンは `server.tool(name, description, schema, handler)`（位置引数）を使用し、他のバージョンは `server.tool({ name, description, inputSchema }, handler)` または `registerTool()` を使用する。リソースも同様 — APIが提供する場合はハンドラーに `uri` を含める。コピー＆ペーストエラーを避けるために、現在の `@modelcontextprotocol/sdk` のシグネチャについて公式MCPドキュメントまたはContext7を確認すること。

入力バリデーションには**Zod**（またはSDKが推奨するスキーマ形式）を使用する。

## ベストプラクティス

- **スキーマファースト**: すべてのツールに入力スキーマを定義し、パラメータと戻り値の形式を文書化する。
- **エラー**: モデルが解釈できる構造化エラーまたはメッセージを返す。生のスタックトレースは避ける。
- **冪等性**: リトライが安全になるよう、可能な限り冪等なツールを優先する。
- **レートとコスト**: 外部APIを呼び出すツールの場合、レート制限とコストを考慮し、ツールの説明に文書化する。
- **バージョニング**: package.jsonでSDKバージョンをピン留めし、アップグレード時にリリースノートを確認する。

## 公式SDKとドキュメント

- **JavaScript/TypeScript**: `@modelcontextprotocol/sdk`（npm）。現在の登録とトランスポートパターンについてはContext7でライブラリ名「MCP」を使用。
- **Go**: GitHubの公式Go SDK（`modelcontextprotocol/go-sdk`）。
- **C#**: .NET用の公式C# SDK。
