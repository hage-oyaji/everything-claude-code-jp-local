---
name: data-scraper-agent
description: あらゆる公開ソース向けの完全自動化AI駆動データ収集エージェントを構築 — 求人、価格、ニュース、GitHub、スポーツなどあらゆるデータに対応。スケジュールでスクレイピングし、無料LLM（Gemini Flash）でデータを強化し、Notion / Sheets / Supabaseに結果を保存し、ユーザーフィードバックから学習。GitHub Actionsで100%無料で稼働。ユーザーが自動的にデータを監視、収集、追跡したい場合に使用。
origin: community
---

# データスクレイパーエージェント

あらゆる公開データソース向けのプロダクションレディなAI駆動データ収集エージェントを構築する。
スケジュールで実行し、無料LLMで結果を強化し、データベースに保存し、時間とともに改善する。

**スタック: Python · Gemini Flash（無料） · GitHub Actions（無料） · Notion / Sheets / Supabase**

## 発動条件

- ユーザーがあらゆる公開ウェブサイトやAPIをスクレイピングまたは監視したいとき
- ユーザーが「...をチェックするボットを作って」「Xを監視して」「...からデータを収集して」と言うとき
- ユーザーが求人、価格、ニュース、リポジトリ、スポーツスコア、イベント、リスティングを追跡したいとき
- ユーザーがホスティング費用なしでデータ収集を自動化する方法を尋ねるとき
- ユーザーが自分の判断に基づいて時間とともに賢くなるエージェントを求めるとき

## コアコンセプト

### 3つのレイヤー

すべてのデータスクレイパーエージェントには3つのレイヤーがある:

```
COLLECT → ENRICH → STORE
  │           │        │
Scraper    AI (LLM)  Database
runs on    scores/   Notion /
schedule   summarises Sheets /
           & classifies Supabase
```

### 無料スタック

| レイヤー | ツール | 理由 |
|---|---|---|
| **スクレイピング** | `requests` + `BeautifulSoup` | コストなし、公開サイトの80%をカバー |
| **JS描画サイト** | `playwright`（無料） | HTMLスクレイピングが失敗した場合 |
| **AI強化** | Gemini Flash（REST API経由） | 500リクエスト/日、100万トークン/日 — 無料 |
| **ストレージ** | Notion API | 無料枠、レビューに優れたUI |
| **スケジュール** | GitHub Actions cron | 公開リポジトリは無料 |
| **学習** | リポジトリ内のJSONフィードバックファイル | インフラゼロ、gitに永続化 |

### AIモデルフォールバックチェーン

クォータ消費時に自動フォールバックするエージェントを構築:

```
gemini-2.0-flash-lite (30 RPM) →
gemini-2.0-flash (15 RPM) →
gemini-2.5-flash (10 RPM) →
gemini-flash-lite-latest (fallback)
```

### 効率化のためのバッチAPI呼び出し

アイテムごとに1回LLMを呼び出さない。常にバッチ処理:

```python
# BAD: 33 API calls for 33 items
for item in items:
    result = call_ai(item)  # 33 calls → hits rate limit

# GOOD: 7 API calls for 33 items (batch size 5)
for batch in chunks(items, size=5):
    results = call_ai(batch)  # 7 calls → stays within free tier
```

---

## ワークフロー

### ステップ1: 目的の理解

ユーザーに問い合わせる:

1. **何を収集するか:** 「どのデータソース？ URL / API / RSS / 公開エンドポイント？」
2. **何を抽出するか:** 「どのフィールドが重要？ タイトル、価格、URL、日付、スコア？」
3. **どこに保存するか:** 「結果をどこに保存？ Notion、Google Sheets、Supabase、ローカルファイル？」
4. **どう強化するか:** 「AIに各アイテムをスコアリング、要約、分類、マッチングさせたいか？」
5. **頻度:** 「どのくらいの頻度で実行？ 毎時、毎日、毎週？」

促すための一般的な例:
- 求人ボード → 履歴書への関連性をスコアリング
- 製品価格 → 値下げ時にアラート
- GitHubリポジトリ → 新しいリリースを要約
- ニュースフィード → トピックとセンチメントで分類
- スポーツ結果 → トラッカーに統計を抽出
- イベントカレンダー → 興味でフィルター

---

### ステップ2: エージェントアーキテクチャの設計

ユーザー向けにこのディレクトリ構造を生成する:

```
my-agent/
├── config.yaml              # User customises this (keywords, filters, preferences)
├── profile/
│   └── context.md           # User context the AI uses (resume, interests, criteria)
├── scraper/
│   ├── __init__.py
│   ├── main.py              # Orchestrator: scrape → enrich → store
│   ├── filters.py           # Rule-based pre-filter (fast, before AI)
│   └── sources/
│       ├── __init__.py
│       └── source_name.py   # One file per data source
├── ai/
│   ├── __init__.py
│   ├── client.py            # Gemini REST client with model fallback
│   ├── pipeline.py          # Batch AI analysis
│   ├── jd_fetcher.py        # Fetch full content from URLs (optional)
│   └── memory.py            # Learn from user feedback
├── storage/
│   ├── __init__.py
│   └── notion_sync.py       # Or sheets_sync.py / supabase_sync.py
├── data/
│   └── feedback.json        # User decision history (auto-updated)
├── .env.example
├── setup.py                 # One-time DB/schema creation
├── enrich_existing.py       # Backfill AI scores on old rows
├── requirements.txt
└── .github/
    └── workflows/
        └── scraper.yml      # GitHub Actions schedule
```

---

### ステップ3-10: 実装

詳細なコード例とパターンについてはオリジナルの英語版スキルドキュメントを参照してください。コードブロック内のソースコード、設定ファイル、GitHub Actionsワークフローはすべてそのまま使用できます。

---

## 一般的なスクレイピングパターン

### パターン1: REST API（最も簡単）
```python
resp = requests.get(url, params={"q": query}, headers=HEADERS, timeout=15)
items = resp.json().get("results", [])
```

### パターン2: HTMLスクレイピング
```python
soup = BeautifulSoup(resp.text, "lxml")
for card in soup.select(".listing-card"):
    title = card.select_one("h2").get_text(strip=True)
    href = card.select_one("a")["href"]
```

### パターン3: RSSフィード
```python
import xml.etree.ElementTree as ET
root = ET.fromstring(resp.text)
for item in root.findall(".//item"):
    title = item.findtext("title", "")
    link = item.findtext("link", "")
    pub_date = item.findtext("pubDate", "")
```

### パターン4: ページネーション付きAPI
```python
page = 1
while True:
    resp = requests.get(url, params={"page": page, "limit": 50}, timeout=15)
    data = resp.json()
    items = data.get("results", [])
    if not items:
        break
    for item in items:
        results.append(_normalise(item))
    if not data.get("has_more"):
        break
    page += 1
```

### パターン5: JS描画ページ（Playwright）
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page()
    page.goto(url)
    page.wait_for_selector(".listing")
    html = page.content()
    browser.close()

soup = BeautifulSoup(html, "lxml")
```

---

## 避けるべきアンチパターン

| アンチパターン | 問題 | 修正 |
|---|---|---|
| アイテムごとに1回のLLM呼び出し | レートリミットに即座に到達 | 1回の呼び出しで5アイテムをバッチ処理 |
| コード内のハードコードキーワード | 再利用不可 | すべての設定を`config.yaml`に移動 |
| レートリミットなしのスクレイピング | IPバン | リクエスト間に`time.sleep(1)`を追加 |
| コード内にシークレットを保存 | セキュリティリスク | 常に`.env` + GitHub Secretsを使用 |
| 重複排除なし | 重複行が蓄積 | プッシュ前に常にURLを確認 |
| `robots.txt`を無視 | 法的/倫理的リスク | クロールルールを尊重; 利用可能な場合は公開APIを使用 |
| `requests`でのJS描画サイト | 空レスポンス | Playwrightを使用するか、基盤となるAPIを探す |
| `maxOutputTokens`が低すぎる | JSON切り捨て、パースエラー | バッチレスポンスには2048+を使用 |

---

## 無料枠制限リファレンス

| サービス | 無料制限 | 典型的な使用量 |
|---|---|---|
| Gemini Flash Lite | 30 RPM、1500 RPD | 3時間間隔で約56リクエスト/日 |
| Gemini 2.0 Flash | 15 RPM、1500 RPD | 良いフォールバック |
| Gemini 2.5 Flash | 10 RPM、500 RPD | 控えめに使用 |
| GitHub Actions | 無制限（公開リポジトリ） | 約20分/日 |
| Notion API | 無制限 | 約200書き込み/日 |
| Supabase | 500MB DB、2GB転送 | ほとんどのエージェントに十分 |
| Google Sheets API | 300リクエスト/分 | 小規模エージェント向け |

---

## 要件テンプレート

```
requests==2.31.0
beautifulsoup4==4.12.3
lxml==5.1.0
python-dotenv==1.0.1
pyyaml==6.0.2
notion-client==2.2.1   # if using Notion
# playwright==1.40.0   # uncomment for JS-rendered sites
```

---

## 品質チェックリスト

エージェントの完了をマークする前に:

- [ ] `config.yaml`がすべてのユーザー向け設定を制御 — ハードコード値なし
- [ ] `profile/context.md`がAIマッチングのためのユーザー固有コンテキストを保持
- [ ] ストレージプッシュ前にURL重複排除
- [ ] Geminiクライアントにモデルフォールバックチェーン（4モデル）
- [ ] バッチサイズ ≤ API呼び出しあたり5アイテム
- [ ] `maxOutputTokens` ≥ 2048
- [ ] `.env`が`.gitignore`に含まれている
- [ ] オンボーディング用の`.env.example`が提供されている
- [ ] `setup.py`が初回実行時にDBスキーマを作成
- [ ] `enrich_existing.py`が古い行にAIスコアをバックフィル
- [ ] GitHub Actionsワークフローが実行ごとに`feedback.json`をコミット
- [ ] READMEがカバー: 5分以内のセットアップ、必要なシークレット、カスタマイズ

---

## 実世界の例

```
"Build me an agent that monitors Hacker News for AI startup funding news"
"Scrape product prices from 3 e-commerce sites and alert when they drop"
"Track new GitHub repos tagged with 'llm' or 'agents' — summarise each one"
"Collect Chief of Staff job listings from LinkedIn and Cutshort into Notion"
"Monitor a subreddit for posts mentioning my company — classify sentiment"
"Scrape new academic papers from arXiv on a topic I care about daily"
"Track sports fixture results and keep a running table in Google Sheets"
"Build a real estate listing watcher — alert on new properties under ₹1 Cr"
```

---

## リファレンス実装

このアーキテクチャで構築された完全に動作するエージェントは、4以上のソースをスクレイピングし、Gemini呼び出しをバッチ処理し、Notionに保存されたApplied/Rejectedの判断から学習し、GitHub Actionsで100%無料で実行される。上記のステップ1-9に従って独自のエージェントを構築する。
