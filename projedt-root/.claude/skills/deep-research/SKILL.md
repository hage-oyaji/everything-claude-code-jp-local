---
name: deep-research
description: firecrawlとexa MCPを使用したマルチソースディープリサーチ。Webを検索し、調査結果を統合し、ソース帰属付きの引用レポートを提供。トピックについてエビデンスと引用付きの徹底的なリサーチが必要な場合に使用。
origin: ECC
---

# ディープリサーチ

firecrawlとexa MCPツールを使用して、複数のWebソースから徹底的で引用付きのリサーチレポートを作成する。

## 発動条件

- ユーザーがあるトピックについて深く調査するよう依頼したとき
- 競合分析、技術評価、市場規模の推定
- 企業、投資家、テクノロジーのデューデリジェンス
- 複数のソースからの統合を必要とする質問
- ユーザーが「research」「deep dive」「investigate」「what's the current state of」と発言したとき

## MCP要件

以下の少なくとも1つ:
- **firecrawl** — `firecrawl_search`、`firecrawl_scrape`、`firecrawl_crawl`
- **exa** — `web_search_exa`、`web_search_advanced_exa`、`crawling_exa`

両方を併用すると最高のカバレッジが得られる。`~/.claude.json`または`~/.codex/config.toml`で設定する。

## ワークフロー

### ステップ1: 目的の理解

1-2個の簡単な明確化質問をする:
- 「目的は何ですか — 学習、意思決定、何かを書くこと？」
- 「特定の角度や深さは必要ですか？」

ユーザーが「とにかく調べて」と言う場合 — 合理的なデフォルトで先に進む。

### ステップ2: リサーチの計画

トピックを3-5個のリサーチサブ質問に分解する。例:
- トピック: 「AIの医療への影響」
  - 今日の医療における主なAIアプリケーションは何か？
  - どのような臨床アウトカムが測定されているか？
  - 規制上の課題は何か？
  - この分野をリードしている企業は？
  - 市場規模と成長軌道は？

### ステップ3: マルチソース検索の実行

各サブ質問について、利用可能なMCPツールで検索する:

**firecrawlの場合:**
```
firecrawl_search(query: "<sub-question keywords>", limit: 8)
```

**exaの場合:**
```
web_search_exa(query: "<sub-question keywords>", numResults: 8)
web_search_advanced_exa(query: "<keywords>", numResults: 5, startPublishedDate: "2025-01-01")
```

**検索戦略:**
- サブ質問ごとに2-3種類のキーワードバリエーションを使用
- 一般的なクエリとニュースに焦点を当てたクエリを混在
- 合計15-30のユニークなソースを目指す
- 優先順位: 学術、公式、信頼できるニュース > ブログ > フォーラム

### ステップ4: 主要ソースの深い読み込み

最も有望なURLについて、フルコンテンツを取得:

**firecrawlの場合:**
```
firecrawl_scrape(url: "<url>")
```

**exaの場合:**
```
crawling_exa(url: "<url>", tokensNum: 5000)
```

深さのために3-5個の主要ソースをフルで読む。検索スニペットだけに頼らない。

### ステップ5: レポートの統合と作成

レポートの構造:

```markdown
# [Topic]: Research Report
*Generated: [date] | Sources: [N] | Confidence: [High/Medium/Low]*

## Executive Summary
[3-5 sentence overview of key findings]

## 1. [First Major Theme]
[Findings with inline citations]
- Key point ([Source Name](url))
- Supporting data ([Source Name](url))

## 2. [Second Major Theme]
...

## 3. [Third Major Theme]
...

## Key Takeaways
- [Actionable insight 1]
- [Actionable insight 2]
- [Actionable insight 3]

## Sources
1. [Title](url) — [one-line summary]
2. ...

## Methodology
Searched [N] queries across web and news. Analyzed [M] sources.
Sub-questions investigated: [list]
```

### ステップ6: 配信

- **短いトピック**: チャットにフルレポートを投稿
- **長いレポート**: エグゼクティブサマリー + 主要テイクアウェイを投稿し、フルレポートをファイルに保存

## サブエージェントによる並列リサーチ

広いトピックの場合、Claude CodeのTaskツールを使用して並列化:

```
Launch 3 research agents in parallel:
1. Agent 1: Research sub-questions 1-2
2. Agent 2: Research sub-questions 3-4
3. Agent 3: Research sub-question 5 + cross-cutting themes
```

各エージェントが検索し、ソースを読み、調査結果を返す。メインセッションが最終レポートに統合する。

## 品質ルール

1. **すべての主張にソースが必要。** ソースなしの主張を行わない。
2. **クロスリファレンス。** 1つのソースだけが述べている場合、未検証としてフラグを立てる。
3. **最新性が重要。** 過去12ヶ月のソースを優先する。
4. **ギャップを認める。** サブ質問について良い情報が見つからなかった場合、そう述べる。
5. **ハルシネーションなし。** 分からない場合は「十分なデータが見つかりませんでした」と述べる。
6. **事実と推論を分離する。** 推定、予測、意見を明確にラベル付けする。

## 例

```
"Research the current state of nuclear fusion energy"
"Deep dive into Rust vs Go for backend services in 2026"
"Research the best strategies for bootstrapping a SaaS business"
"What's happening with the US housing market right now?"
"Investigate the competitive landscape for AI code editors"
```
