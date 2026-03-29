---
name: nextjs-turbopack
description: Next.js 16+とTurbopack — インクリメンタルバンドル、FSキャッシュ、開発速度、TurbopackとWebpackの使い分け。
origin: ECC
---

# Next.jsとTurbopack

Next.js 16+はローカル開発にデフォルトでTurbopackを使用する。Rustで書かれたインクリメンタルバンドラーで、開発の起動とホットアップデートを大幅に高速化する。

## 使用するタイミング

- **Turbopack（デフォルト開発）**: 日常の開発に使用。特に大規模アプリでコールドスタートとHMRが高速。
- **Webpack（レガシー開発）**: Turbopackのバグに遭遇した場合、またはWebpack専用プラグインに依存している場合のみ使用。`--webpack`（またはNext.jsバージョンによっては `--no-turbopack`; リリースのドキュメントを確認）で無効化。
- **本番**: 本番ビルドの動作（`next build`）はNext.jsバージョンによってTurbopackまたはWebpackを使用する場合がある。バージョンの公式Next.jsドキュメントを確認すること。

Next.js 16+アプリの開発やデバッグ、開発起動やHMRの遅さの診断、本番バンドルの最適化時に使用する。

## 仕組み

- **Turbopack**: Next.js開発用のインクリメンタルバンドラー。ファイルシステムキャッシュを使用するため、再起動が大幅に高速化される（例: 大規模プロジェクトで5〜14倍）。
- **開発でのデフォルト**: Next.js 16から、`next dev` は無効化しない限りTurbopackで実行される。
- **ファイルシステムキャッシュ**: 再起動時に以前の作業を再利用。キャッシュは通常 `.next` 配下にあり、基本的な使用には追加設定不要。
- **Bundle Analyzer（Next.js 16.1+）**: 出力を検査し、重い依存関係を見つけるための実験的なBundle Analyzer。設定または実験的フラグで有効化（バージョンのNext.jsドキュメントを参照）。

## 使用例

### コマンド

```bash
next dev
next build
next start
```

### 使い方

ローカル開発にはTurbopack付きの `next dev` を実行する。コード分割の最適化や大きな依存関係の削減にはBundle Analyzer（Next.jsドキュメントを参照）を使用する。可能な限りApp Routerとサーバーコンポーネントを優先する。

## ベストプラクティス

- 安定したTurbopackとキャッシュ動作のために、最新のNext.js 16.xを使用する。
- 開発が遅い場合、Turbopack（デフォルト）を使用していて、キャッシュが不必要にクリアされていないことを確認する。
- 本番バンドルサイズの問題には、バージョンの公式Next.jsバンドル分析ツールを使用する。
