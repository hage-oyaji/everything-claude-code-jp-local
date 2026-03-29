---
name: nuxt4-patterns
description: Nuxt 4アプリパターン — ハイドレーション安全性、パフォーマンス、ルートルール、遅延読み込み、useFetchとuseAsyncDataによるSSR安全なデータフェッチ。
origin: ECC
---

# Nuxt 4パターン

SSR、ハイブリッドレンダリング、ルートルール、またはページレベルのデータフェッチを伴うNuxt 4アプリの構築やデバッグ時に使用する。

## 有効化するタイミング

- サーバーHTMLとクライアントステート間のハイドレーションミスマッチ
- プリレンダー、SWR、ISR、クライアント専用セクションなどのルートレベルのレンダリング決定
- 遅延読み込み、遅延ハイドレーション、ペイロードサイズに関するパフォーマンスワーク
- `useFetch`、`useAsyncData`、`$fetch` によるページまたはコンポーネントのデータフェッチ
- ルートパラメータ、ミドルウェア、SSR/クライアントの差異に関連するNuxtルーティングの問題

## ハイドレーション安全性

- 最初のレンダリングを決定論的に保つ。`Date.now()`、`Math.random()`、ブラウザ専用API、ストレージの読み取りをSSRレンダリングされたテンプレートステートに直接配置しない。
- サーバーが同じマークアップを生成できない場合、ブラウザ専用ロジックを `onMounted()`、`import.meta.client`、`ClientOnly`、または `.client.vue` コンポーネントの背後に移動する。
- `vue-router` のものではなく、Nuxtの `useRoute()` コンポーザブルを使用する。
- SSRレンダリングされたマークアップを駆動するために `route.fullPath` を使用しない。URLフラグメントはクライアント専用であり、ハイドレーションミスマッチを引き起こす可能性がある。
- `ssr: false` はミスマッチのデフォルト修正ではなく、真にブラウザ専用の領域のためのエスケープハッチとして扱う。

## データフェッチ

- ページやコンポーネントでのSSR安全なAPIリードには `await useFetch()` を優先する。サーバーでフェッチしたデータをNuxtペイロードに転送し、ハイドレーション時の2回目のフェッチを回避する。
- フェッチャーが単純な `$fetch()` 呼び出しでない場合、カスタムキーが必要な場合、または複数の非同期ソースを組み合わせる場合は `useAsyncData()` を使用する。
- キャッシュの再利用と予測可能なリフレッシュ動作のために `useAsyncData()` に安定したキーを与える。
- `useAsyncData()` ハンドラーは副作用のない状態に保つ。SSRとハイドレーション中に実行される可能性がある。
- ユーザートリガーの書き込みやクライアント専用アクションには `$fetch()` を使用する。SSRからハイドレーションされるべきトップレベルのページデータには使用しない。
- ナビゲーションをブロックすべきでない非クリティカルデータには `lazy: true`、`useLazyFetch()`、または `useLazyAsyncData()` を使用する。UIで `status === 'pending'` を処理する。
- SEOや初回ペイントに不要なデータにのみ `server: false` を使用する。
- `pick` でペイロードサイズを削減し、深いリアクティビティが不要な場合はより浅いペイロードを優先する。

```ts
const route = useRoute()

const { data: article, status, error, refresh } = await useAsyncData(
  () => `article:${route.params.slug}`,
  () => $fetch(`/api/articles/${route.params.slug}`),
)

const { data: comments } = await useFetch(`/api/articles/${route.params.slug}/comments`, {
  lazy: true,
  server: false,
})
```

## ルートルール

レンダリングとキャッシュ戦略には `nuxt.config.ts` の `routeRules` を優先する:

```ts
export default defineNuxtConfig({
  routeRules: {
    '/': { prerender: true },
    '/products/**': { swr: 3600 },
    '/blog/**': { isr: true },
    '/admin/**': { ssr: false },
    '/api/**': { cache: { maxAge: 60 * 60 } },
  },
})
```

- `prerender`: ビルド時の静的HTML
- `swr`: キャッシュされたコンテンツを提供し、バックグラウンドで再検証
- `isr`: サポートされるプラットフォームでのインクリメンタル静的再生成
- `ssr: false`: クライアントレンダリングされるルート
- `cache` または `redirect`: Nitroレベルのレスポンス動作

ルートルールはグローバルではなく、ルートグループごとに選択する。マーケティングページ、カタログ、ダッシュボード、APIは通常異なる戦略が必要。

## 遅延読み込みとパフォーマンス

- Nuxtはすでにルートごとにページをコード分割している。コンポーネント分割をマイクロ最適化する前に、ルート境界を意味のあるものに保つ。
- 非クリティカルなコンポーネントの動的インポートには `Lazy` プレフィックスを使用する。
- UIが実際に必要とするまでチャンクがロードされないよう、遅延コンポーネントを `v-if` で条件付きレンダリングする。
- ファーストビュー以下や非クリティカルなインタラクティブUIには遅延ハイドレーションを使用する。

```vue
<template>
  <LazyRecommendations v-if="showRecommendations" />
  <LazyProductGallery hydrate-on-visible />
</template>
```

- カスタム戦略には、ビジビリティまたはアイドル戦略で `defineLazyHydrationComponent()` を使用する。
- Nuxtの遅延ハイドレーションはシングルファイルコンポーネントで動作する。遅延ハイドレーションされたコンポーネントに新しいpropsを渡すと、即座にハイドレーションがトリガーされる。
- 内部ナビゲーションには `NuxtLink` を使用して、Nuxtがルートコンポーネントと生成されたペイロードをプリフェッチできるようにする。

## レビューチェックリスト

- 最初のSSRレンダリングとハイドレーションされたクライアントレンダリングが同じマークアップを生成する
- ページデータがトップレベルの `$fetch` ではなく `useFetch` または `useAsyncData` を使用している
- 非クリティカルデータは遅延で、明示的なローディングUIがある
- ルートルールがページのSEOと鮮度の要件に一致している
- 重いインタラクティブアイランドが遅延読み込みまたは遅延ハイドレーションされている
