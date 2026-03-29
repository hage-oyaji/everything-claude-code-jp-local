---
name: crosspost
description: X、LinkedIn、Threads、Blueskyにわたるマルチプラットフォームコンテンツ配信。content-engineパターンを使用してプラットフォームごとにコンテンツを適応。同一コンテンツのクロスプラットフォーム投稿は行わない。ソーシャルプラットフォーム間でコンテンツを配信したい場合に使用。
origin: ECC
---

# クロスポスト

プラットフォームネイティブな適応を伴う、複数ソーシャルプラットフォーム間でのコンテンツ配信。

## 発動条件

- ユーザーが複数のプラットフォームにコンテンツを投稿したいとき
- ソーシャルメディア全体でアナウンス、ローンチ、アップデートを公開するとき
- 1つのプラットフォームからの投稿を他のプラットフォームにリパーパスするとき
- ユーザーが「crosspost」「post everywhere」「share on all platforms」「distribute this」と発言したとき

## コアルール

1. **同一コンテンツをクロスプラットフォームで投稿しない。** 各プラットフォームにネイティブな適応を行う。
2. **プライマリープラットフォームを先に。** メインプラットフォームに投稿してから、他のプラットフォームに適応する。
3. **プラットフォーム規約を尊重する。** 文字数制限、フォーマット、リンク処理はすべて異なる。
4. **投稿ごとに1つのアイデア。** ソースコンテンツに複数のアイデアがある場合、投稿を分割する。
5. **帰属表示は重要。** 他者のコンテンツをクロスポストする場合、ソースをクレジットする。

## プラットフォーム仕様

| プラットフォーム | 最大長 | リンク処理 | ハッシュタグ | メディア |
|----------|-----------|---------------|----------|-------|
| X | 280文字（Premiumは4000） | 文字数にカウント | 最小限（最大1-2） | 画像、動画、GIF |
| LinkedIn | 3000文字 | 文字数にカウントされない | 3-5個の関連タグ | 画像、動画、ドキュメント、カルーセル |
| Threads | 500文字 | 別リンク添付 | 一般的に使用しない | 画像、動画 |
| Bluesky | 300文字 | ファセット経由（リッチテキスト） | なし（フィードを使用） | 画像 |

## ワークフロー

### ステップ1: ソースコンテンツの作成

コアアイデアから始める。高品質なドラフトには`content-engine`スキルを使用:
- 単一のコアメッセージを特定する
- プライマリープラットフォームを決定する（オーディエンスが最も多い場所）
- プライマリープラットフォームバージョンを最初にドラフトする

### ステップ2: ターゲットプラットフォームの特定

ユーザーに問い合わせるかコンテキストから判断する:
- ターゲットとするプラットフォーム
- 優先順位（プライマリーが最高のバージョンを得る）
- プラットフォーム固有の要件（例: LinkedInはプロフェッショナルなトーンが必要）

### ステップ3: プラットフォームごとの適応

各ターゲットプラットフォームに対してコンテンツを変換する:

**X適応:**
- 要約ではなくフックでオープンする
- コアインサイトに素早く切り込む
- 可能な限りリンクは本文の外に置く
- 長いコンテンツにはスレッドフォーマットを使用

**LinkedIn適応:**
- 強い最初の1行（「see more」の前に見える）
- 改行を含む短い段落
- レッスン、結果、プロフェッショナルなテイクアウェイのフレーミング
- Xよりも明確なコンテキスト（LinkedInのオーディエンスにはフレーミングが必要）

**Threads適応:**
- 会話的、カジュアルなトーン
- LinkedInより短く、Xより圧縮されていない
- 可能であればビジュアルファースト

**Bluesky適応:**
- 直接的で簡潔（300文字制限）
- コミュニティ志向のトーン
- ハッシュタグの代わりにフィード/リストを使用してトピックをターゲティング

### ステップ4: プライマリープラットフォームへの投稿

プライマリープラットフォームに最初に投稿:
- Xには`x-api`スキルを使用
- 他のプラットフォームにはプラットフォーム固有のAPIやツールを使用
- クロスリファレンス用に投稿URLをキャプチャ

### ステップ5: セカンダリープラットフォームへの投稿

適応されたバージョンを残りのプラットフォームに投稿:
- タイミングをずらす（一度にすべてではなく — 30-60分の間隔）
- 適切な場合はクロスプラットフォーム参照を含める（「Xに詳細なスレッドあり」など）

## コンテンツ適応の例

### ソース: プロダクトローンチ

**Xバージョン:**
```
We just shipped [feature].

[One specific thing it does that's impressive]

[Link]
```

**LinkedInバージョン:**
```
Excited to share: we just launched [feature] at [Company].

Here's why it matters:

[2-3 short paragraphs with context]

[Takeaway for the audience]

[Link]
```

**Threadsバージョン:**
```
just shipped something cool — [feature]

[casual explanation of what it does]

link in bio
```

### ソース: 技術的インサイト

**Xバージョン:**
```
TIL: [specific technical insight]

[Why it matters in one sentence]
```

**LinkedInバージョン:**
```
A pattern I've been using that's made a real difference:

[Technical insight with professional framing]

[How it applies to teams/orgs]

#relevantHashtag
```

## API統合

### バッチクロスポストサービス（パターン例）
クロスポストサービス（例: Postbridge、Buffer、カスタムAPI）を使用する場合のパターン:

```python
import os
import requests

resp = requests.post(
    "https://your-crosspost-service.example/api/posts",
    headers={"Authorization": f"Bearer {os.environ['POSTBRIDGE_API_KEY']}"},
    json={
        "platforms": ["twitter", "linkedin", "threads"],
        "content": {
            "twitter": {"text": x_version},
            "linkedin": {"text": linkedin_version},
            "threads": {"text": threads_version}
        }
    },
    timeout=30,
)
resp.raise_for_status()
```

### 手動投稿
Postbridgeなしで各プラットフォームのネイティブAPIを使用して投稿:
- X: `x-api`スキルパターンを使用
- LinkedIn: OAuth 2.0のLinkedIn API v2
- Threads: Threads API (Meta)
- Bluesky: AT Protocol API

## 品質ゲート

投稿前に:
- [ ] 各プラットフォームバージョンがそのプラットフォームで自然に読める
- [ ] プラットフォーム間で同一コンテンツがない
- [ ] 文字数制限を遵守している
- [ ] リンクが機能し適切に配置されている
- [ ] トーンがプラットフォームの規約に一致している
- [ ] メディアが各プラットフォームに適したサイズになっている

## 関連スキル

- `content-engine` — プラットフォームネイティブコンテンツの生成
- `x-api` — X/Twitter API統合
