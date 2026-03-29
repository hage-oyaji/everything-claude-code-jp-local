---
name: bun-runtime
description: ランタイム、パッケージマネージャー、バンドラー、テストランナーとしてのBun。Bun vs Nodeの選択基準、マイグレーションノート、Vercelサポート。
origin: ECC
---

# Bunランタイム

Bunは高速なオールインワンJavaScriptランタイムおよびツールキットです：ランタイム、パッケージマネージャー、バンドラー、テストランナー。

## 使用タイミング

- **Bunを選ぶ場合**: 新しいJS/TSプロジェクト、インストール/実行速度が重要なスクリプト、Bunランタイムを使用するVercelデプロイ、単一のツールチェーン（実行 + インストール + テスト + ビルド）が必要な場合。
- **Nodeを選ぶ場合**: 最大限のエコシステム互換性、Nodeを前提としたレガシーツール、依存関係にBunの既知の問題がある場合。

Bunの導入、Nodeからのマイグレーション、Bunスクリプト/テストの作成やデバッグ、Vercelやその他のプラットフォームでのBun設定を行う際に使用してください。

## 動作の仕組み

- **ランタイム**: Node互換のドロップインランタイム（JavaScriptCore上に構築、Zigで実装）。
- **パッケージマネージャー**: `bun install`はnpm/yarnより大幅に高速。ロックファイルは現在のBunではデフォルトで`bun.lock`（テキスト）、古いバージョンでは`bun.lockb`（バイナリ）。
- **バンドラー**: アプリとライブラリ向けの組み込みバンドラーおよびトランスパイラー。
- **テストランナー**: Jest風APIを持つ組み込み`bun test`。

**Nodeからのマイグレーション**: `node script.js`を`bun run script.js`または`bun script.js`に置き換えます。`npm install`の代わりに`bun install`を実行、ほとんどのパッケージは動作します。npmスクリプトには`bun run`、npxスタイルのワンオフ実行には`bun x`を使用します。Node組み込みモジュールはサポートされています。パフォーマンス向上のため、存在する場合はBun APIを優先してください。

**Vercel**: プロジェクト設定でランタイムをBunに設定します。ビルド：`bun run build`または`bun build ./src/index.ts --outdir=dist`。インストール：再現可能なデプロイのため`bun install --frozen-lockfile`。

## 使用例

### 実行とインストール

```bash
# Install dependencies (creates/updates bun.lock or bun.lockb)
bun install

# Run a script or file
bun run dev
bun run src/index.ts
bun src/index.ts
```

### スクリプトと環境変数

```bash
bun run --env-file=.env dev
FOO=bar bun run script.ts
```

### テスト

```bash
bun test
bun test --watch
```

```typescript
// test/example.test.ts
import { expect, test } from "bun:test";

test("add", () => {
  expect(1 + 2).toBe(3);
});
```

### ランタイムAPI

```typescript
const file = Bun.file("package.json");
const json = await file.json();

Bun.serve({
  port: 3000,
  fetch(req) {
    return new Response("Hello");
  },
});
```

## ベストプラクティス

- 再現可能なインストールのためロックファイル（`bun.lock`または`bun.lockb`）をコミットする。
- スクリプトには`bun run`を優先する。TypeScriptの場合、Bunは`.ts`をネイティブに実行する。
- 依存関係を最新に保つ。Bunとエコシステムは急速に進化している。
