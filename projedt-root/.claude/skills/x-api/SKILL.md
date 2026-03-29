---
name: x-api
description: X/Twitter APIインテグレーション。ツイートの投稿、スレッド、タイムラインの読み取り、検索、アナリティクスに対応。OAuth認証パターン、レート制限、プラットフォームネイティブのコンテンツ投稿をカバー。Xとプログラム的にやり取りしたいときに使用。
origin: ECC
---

# X API

投稿、読み取り、検索、アナリティクスのためのX（Twitter）とのプログラム的なインタラクション。

## アクティベーション条件

- ユーザーがプログラム的にツイートやスレッドを投稿したいとき
- Xからタイムライン、メンション、またはユーザーデータを読み取るとき
- コンテンツ、トレンド、会話をXで検索するとき
- Xインテグレーションやボットを構築するとき
- アナリティクスとエンゲージメントの追跡
- ユーザーが「post to X」「tweet」「X API」「Twitter API」と言ったとき

## 認証

### OAuth 2.0 Bearerトークン（アプリ専用）

最適な用途: 読み取り中心の操作、検索、公開データ。

```bash
# Environment setup
export X_BEARER_TOKEN="your-bearer-token"
```

```python
import os
import requests

bearer = os.environ["X_BEARER_TOKEN"]
headers = {"Authorization": f"Bearer {bearer}"}

# Search recent tweets
resp = requests.get(
    "https://api.x.com/2/tweets/search/recent",
    headers=headers,
    params={"query": "claude code", "max_results": 10}
)
tweets = resp.json()
```

### OAuth 1.0a（ユーザーコンテキスト）

必要な場合: ツイートの投稿、アカウント管理、DM。

```bash
# Environment setup — source before use
export X_API_KEY="your-api-key"
export X_API_SECRET="your-api-secret"
export X_ACCESS_TOKEN="your-access-token"
export X_ACCESS_SECRET="your-access-secret"
```

```python
import os
from requests_oauthlib import OAuth1Session

oauth = OAuth1Session(
    os.environ["X_API_KEY"],
    client_secret=os.environ["X_API_SECRET"],
    resource_owner_key=os.environ["X_ACCESS_TOKEN"],
    resource_owner_secret=os.environ["X_ACCESS_SECRET"],
)
```

## コア操作

### ツイートの投稿

```python
resp = oauth.post(
    "https://api.x.com/2/tweets",
    json={"text": "Hello from Claude Code"}
)
resp.raise_for_status()
tweet_id = resp.json()["data"]["id"]
```

### スレッドの投稿

```python
def post_thread(oauth, tweets: list[str]) -> list[str]:
    ids = []
    reply_to = None
    for text in tweets:
        payload = {"text": text}
        if reply_to:
            payload["reply"] = {"in_reply_to_tweet_id": reply_to}
        resp = oauth.post("https://api.x.com/2/tweets", json=payload)
        tweet_id = resp.json()["data"]["id"]
        ids.append(tweet_id)
        reply_to = tweet_id
    return ids
```

### ユーザータイムラインの読み取り

```python
resp = requests.get(
    f"https://api.x.com/2/users/{user_id}/tweets",
    headers=headers,
    params={
        "max_results": 10,
        "tweet.fields": "created_at,public_metrics",
    }
)
```

### ツイートの検索

```python
resp = requests.get(
    "https://api.x.com/2/tweets/search/recent",
    headers=headers,
    params={
        "query": "from:affaanmustafa -is:retweet",
        "max_results": 10,
        "tweet.fields": "public_metrics,created_at",
    }
)
```

### ユーザー名によるユーザー取得

```python
resp = requests.get(
    "https://api.x.com/2/users/by/username/affaanmustafa",
    headers=headers,
    params={"user.fields": "public_metrics,description,created_at"}
)
```

### メディアのアップロードと投稿

```python
# Media upload uses v1.1 endpoint

# Step 1: Upload media
media_resp = oauth.post(
    "https://upload.twitter.com/1.1/media/upload.json",
    files={"media": open("image.png", "rb")}
)
media_id = media_resp.json()["media_id_string"]

# Step 2: Post with media
resp = oauth.post(
    "https://api.x.com/2/tweets",
    json={"text": "Check this out", "media": {"media_ids": [media_id]}}
)
```

## レート制限

X APIのレート制限はエンドポイント、認証方法、アカウントティアにより異なり、時間とともに変更されます。常に:
- 仮定をハードコードする前に現在のX開発者ドキュメントを確認
- ランタイムで`x-rate-limit-remaining`と`x-rate-limit-reset`ヘッダーを読み取る
- コード内の静的テーブルに頼らず自動的にバックオフ

```python
import time

remaining = int(resp.headers.get("x-rate-limit-remaining", 0))
if remaining < 5:
    reset = int(resp.headers.get("x-rate-limit-reset", 0))
    wait = max(0, reset - int(time.time()))
    print(f"Rate limit approaching. Resets in {wait}s")
```

## エラーハンドリング

```python
resp = oauth.post("https://api.x.com/2/tweets", json={"text": content})
if resp.status_code == 201:
    return resp.json()["data"]["id"]
elif resp.status_code == 429:
    reset = int(resp.headers["x-rate-limit-reset"])
    raise Exception(f"Rate limited. Resets at {reset}")
elif resp.status_code == 403:
    raise Exception(f"Forbidden: {resp.json().get('detail', 'check permissions')}")
else:
    raise Exception(f"X API error {resp.status_code}: {resp.text}")
```

## セキュリティ

- **トークンをハードコードしない。** 環境変数または`.env`ファイルを使用。
- **`.env`ファイルをコミットしない。** `.gitignore`に追加。
- **露出した場合はトークンをローテーション。** developer.x.comで再生成。
- **書き込みアクセスが不要な場合は読み取り専用トークンを使用。**
- **OAuthシークレットを安全に保管** — ソースコードやログには記載しない。

## コンテンツエンジンとの統合

`content-engine`スキルを使用してプラットフォームネイティブのコンテンツを生成し、X APIで投稿:
1. content-engineでコンテンツを生成（Xプラットフォーム形式）
2. 長さを検証（単一ツイートは280文字）
3. 上記のパターンを使用してX APIで投稿
4. public_metricsでエンゲージメントを追跡

## 関連スキル

- `content-engine` — Xのプラットフォームネイティブコンテンツを生成
- `crosspost` — X、LinkedIn、その他のプラットフォーム間でコンテンツを配信
