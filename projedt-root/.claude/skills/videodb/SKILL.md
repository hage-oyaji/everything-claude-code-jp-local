---
name: videodb
description: 動画と音声の認識・理解・操作。認識 - ローカルファイル、URL、RTSP/ライブフィード、またはデスクトップのライブ録画からの取り込み; リアルタイムコンテキストと再生可能なストリームリンクを返す。理解 - フレームを抽出し、視覚的/意味的/時間的インデックスを構築し、タイムスタンプとオートクリップで瞬間を検索。操作 - トランスコードと正規化（コーデック、fps、解像度、アスペクト比）、タイムライン編集（字幕、テキスト/画像オーバーレイ、ブランディング、オーディオオーバーレイ、吹き替え、翻訳）、メディアアセット生成（画像、音声、動画）、ライブストリームまたはデスクトップキャプチャからのイベントのリアルタイムアラートの作成。
origin: ECC
allowed-tools: Read Grep Glob Bash(python:*)
argument-hint: "[task description]"
---

# VideoDBスキル

**動画、ライブストリーム、デスクトップセッションのための認識 + 記憶 + アクション。**

## 使用タイミング

### デスクトップ認識
- **スクリーン、マイク、システム音声**をキャプチャする**デスクトップセッション**の開始/停止
- **ライブコンテキスト**のストリーミングと**エピソードセッション記憶**の保存
- 画面上で話されていることと起きていることに対する**リアルタイムアラート/トリガー**の実行
- **セッションサマリー**、検索可能なタイムライン、**再生可能な証拠リンク**の生成

### 動画取り込み + ストリーム
- **ファイルまたはURL**の取り込みと**再生可能なWebストリームリンク**の返却
- トランスコード/正規化: **コーデック、ビットレート、fps、解像度、アスペクト比**

### インデックス + 検索（タイムスタンプ + 証拠）
- **視覚的**、**音声**、**キーワード**インデックスの構築
- 正確な瞬間を**タイムスタンプ**と**再生可能な証拠**で検索して返却
- 検索結果から**クリップ**を自動作成

### タイムライン編集 + 生成
- 字幕: **生成**、**翻訳**、**バーンイン**
- オーバーレイ: **テキスト/画像/ブランディング**、モーションキャプション
- 音声: **バックグラウンドミュージック**、**ナレーション**、**吹き替え**
- **タイムライン操作**によるプログラム的なコンポジションとエクスポート

### ライブストリーム (RTSP) + モニタリング
- **RTSP/ライブフィード**への接続
- **リアルタイムの視覚・音声理解**を実行し、モニタリングワークフロー用の**イベント/アラート**を発行

## 仕組み

### 一般的な入力
- ローカル**ファイルパス**、公開**URL**、または**RTSP URL**
- デスクトップキャプチャリクエスト: **セッションの開始 / 停止 / サマリー**
- 必要な操作: 理解のためのコンテキスト取得、トランスコード仕様、インデックス仕様、検索クエリ、クリップ範囲、タイムライン編集、アラートルール

### 一般的な出力
- **ストリームURL**
- **タイムスタンプ**と**証拠リンク**付きの検索結果
- 生成されたアセット: 字幕、音声、画像、クリップ
- ライブストリーム用の**イベント/アラートペイロード**
- デスクトップ**セッションサマリー**と記憶エントリ

### Pythonコードの実行

VideoDBコードを実行する前に、プロジェクトディレクトリに移動して環境変数を読み込みます:

```python
from dotenv import load_dotenv
load_dotenv(".env")

import videodb
conn = videodb.connect()
```

これは以下から`VIDEO_DB_API_KEY`を読み取ります:
1. 環境（既にエクスポートされている場合）
2. カレントディレクトリのプロジェクトの`.env`ファイル

キーが見つからない場合、`videodb.connect()`は自動的に`AuthenticationError`を発生させます。

短いインラインコマンドで済む場合はスクリプトファイルを書かないでください。

インラインPython（`python -c "..."`）を書く場合、常に適切にフォーマットされたコードを使用してください — セミコロンでステートメントを区切り、読みやすく保ちます。3ステートメントより長い場合は、代わりにヒアドキュメントを使用:

```bash
python << 'EOF'
from dotenv import load_dotenv
load_dotenv(".env")

import videodb
conn = videodb.connect()
coll = conn.get_collection()
print(f"Videos: {len(coll.get_videos())}")
EOF
```

### セットアップ

ユーザーが「setup videodb」または同様のリクエストをした場合:

### 1. SDKのインストール

```bash
pip install "videodb[capture]" python-dotenv
```

Linuxで`videodb[capture]`が失敗する場合、captureエクストラなしでインストール:

```bash
pip install videodb python-dotenv
```

### 2. APIキーの設定

ユーザーは**いずれかの**方法で`VIDEO_DB_API_KEY`を設定する必要があります:

- **ターミナルでエクスポート**（Claude起動前）: `export VIDEO_DB_API_KEY=your-key`
- **プロジェクトの`.env`ファイル**: プロジェクトの`.env`ファイルに`VIDEO_DB_API_KEY=your-key`を保存

無料のAPIキーは[console.videodb.io](https://console.videodb.io)で取得できます（50回の無料アップロード、クレジットカード不要）。

APIキーを自分で読み取り、書き込み、処理**しないでください**。常にユーザーに設定させてください。

### クイックリファレンス

### メディアのアップロード

```python
# URL
video = coll.upload(url="https://example.com/video.mp4")

# YouTube
video = coll.upload(url="https://www.youtube.com/watch?v=VIDEO_ID")

# Local file
video = coll.upload(file_path="/path/to/video.mp4")
```

### トランスクリプト + 字幕

```python
# force=True skips the error if the video is already indexed
video.index_spoken_words(force=True)
text = video.get_transcript_text()
stream_url = video.add_subtitle()
```

### 動画内検索

```python
from videodb.exceptions import InvalidRequestError

video.index_spoken_words(force=True)

# search() raises InvalidRequestError when no results are found.
# Always wrap in try/except and treat "No results found" as empty.
try:
    results = video.search("product demo")
    shots = results.get_shots()
    stream_url = results.compile()
except InvalidRequestError as e:
    if "No results found" in str(e):
        shots = []
    else:
        raise
```

### シーン検索

```python
import re
from videodb import SearchType, IndexType, SceneExtractionType
from videodb.exceptions import InvalidRequestError

# index_scenes() has no force parameter — it raises an error if a scene
# index already exists. Extract the existing index ID from the error.
try:
    scene_index_id = video.index_scenes(
        extraction_type=SceneExtractionType.shot_based,
        prompt="Describe the visual content in this scene.",
    )
except Exception as e:
    match = re.search(r"id\s+([a-f0-9]+)", str(e))
    if match:
        scene_index_id = match.group(1)
    else:
        raise

# Use score_threshold to filter low-relevance noise (recommended: 0.3+)
try:
    results = video.search(
        query="person writing on a whiteboard",
        search_type=SearchType.semantic,
        index_type=IndexType.scene,
        scene_index_id=scene_index_id,
        score_threshold=0.3,
    )
    shots = results.get_shots()
    stream_url = results.compile()
except InvalidRequestError as e:
    if "No results found" in str(e):
        shots = []
    else:
        raise
```

### タイムライン編集

**重要:** タイムラインを構築する前に常にタイムスタンプを検証:
- `start`は >= 0 でなければならない（負の値は黙って受け入れられるが壊れた出力を生成）
- `start`は`end`より小さくなければならない
- `end`は`video.length`以下でなければならない

```python
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

timeline = Timeline(conn)
timeline.add_inline(VideoAsset(asset_id=video.id, start=10, end=30))
timeline.add_overlay(0, TextAsset(text="The End", duration=3, style=TextStyle(fontsize=36)))
stream_url = timeline.generate_stream()
```

### 動画のトランスコード（解像度/品質変更）

```python
from videodb import TranscodeMode, VideoConfig, AudioConfig

# Change resolution, quality, or aspect ratio server-side
job_id = conn.transcode(
    source="https://example.com/video.mp4",
    callback_url="https://example.com/webhook",
    mode=TranscodeMode.economy,
    video_config=VideoConfig(resolution=720, quality=23, aspect_ratio="16:9"),
    audio_config=AudioConfig(mute=False),
)
```

### アスペクト比のリフレーミング（ソーシャルプラットフォーム向け）

**警告:** `reframe()`はサーバーサイドの遅い操作です。長い動画では数分かかりタイムアウトする場合があります。ベストプラクティス:
- 可能な場合は常に`start`/`end`を使用して短いセグメントに限定
- フルレングス動画には非同期処理のために`callback_url`を使用
- まず`Timeline`で動画をトリミングし、短い結果をリフレーミング

```python
from videodb import ReframeMode

# Always prefer reframing a short segment:
reframed = video.reframe(start=0, end=60, target="vertical", mode=ReframeMode.smart)

# Async reframe for full-length videos (returns None, result via webhook):
video.reframe(target="vertical", callback_url="https://example.com/webhook")

# Presets: "vertical" (9:16), "square" (1:1), "landscape" (16:9)
reframed = video.reframe(start=0, end=60, target="square")

# Custom dimensions
reframed = video.reframe(start=0, end=60, target={"width": 1280, "height": 720})
```

### 生成メディア

```python
image = coll.generate_image(
    prompt="a sunset over mountains",
    aspect_ratio="16:9",
)
```

## エラーハンドリング

```python
from videodb.exceptions import AuthenticationError, InvalidRequestError

try:
    conn = videodb.connect()
except AuthenticationError:
    print("Check your VIDEO_DB_API_KEY")

try:
    video = coll.upload(url="https://example.com/video.mp4")
except InvalidRequestError as e:
    print(f"Upload failed: {e}")
```

### よくある落とし穴

| シナリオ | エラーメッセージ | 解決策 |
|----------|--------------|----------|
| 既にインデックス済みの動画をインデックス | `Spoken word index for video already exists` | `video.index_spoken_words(force=True)`で既存の場合スキップ |
| シーンインデックスが既に存在 | `Scene index with id XXXX already exists` | `re.search(r"id\s+([a-f0-9]+)", str(e))`でエラーから既存の`scene_index_id`を抽出 |
| 検索で一致なし | `InvalidRequestError: No results found` | 例外をキャッチして空の結果として処理（`shots = []`） |
| リフレームがタイムアウト | 長い動画で無期限にブロック | `start`/`end`でセグメントを限定、または非同期処理に`callback_url`を渡す |
| Timelineでの負のタイムスタンプ | 黙って壊れたストリームを生成 | `VideoAsset`作成前に常に`start >= 0`を検証 |
| `generate_video()` / `create_collection()`が失敗 | `Operation not allowed`または`maximum limit` | プランゲートされた機能 — プラン制限についてユーザーに通知 |

## 例

### 標準的なプロンプト
- 「Start desktop capture and alert when a password field appears.」
- 「Record my session and produce an actionable summary when it ends.」
- 「Ingest this file and return a playable stream link.」
- 「Index this folder and find every scene with people, return timestamps.」
- 「Generate subtitles, burn them in, and add light background music.」
- 「Connect this RTSP URL and alert when a person enters the zone.」

### スクリーン録画（デスクトップキャプチャ）

`ws_listener.py`を使用して録画セッション中のWebSocketイベントをキャプチャ。デスクトップキャプチャは**macOS**のみをサポート。

#### クイックスタート

1. **stateディレクトリの選択**: `STATE_DIR="${VIDEODB_EVENTS_DIR:-$HOME/.local/state/videodb}"`
2. **リスナーの開始**: `VIDEODB_EVENTS_DIR="$STATE_DIR" python scripts/ws_listener.py --clear "$STATE_DIR" &`
3. **WebSocket IDの取得**: `cat "$STATE_DIR/videodb_ws_id"`
4. **キャプチャコードの実行**（フルワークフローはreference/capture.mdを参照）
5. **イベントの書き込み先**: `$STATE_DIR/videodb_events.jsonl`

新しいキャプチャランを開始するたびに`--clear`を使用し、古いトランスクリプトやビジュアルイベントが新しいセッションに漏れないようにします。

#### イベントのクエリ

```python
import json
import os
import time
from pathlib import Path

events_dir = Path(os.environ.get("VIDEODB_EVENTS_DIR", Path.home() / ".local" / "state" / "videodb"))
events_file = events_dir / "videodb_events.jsonl"
events = []

if events_file.exists():
    with events_file.open(encoding="utf-8") as handle:
        for line in handle:
            try:
                events.append(json.loads(line))
            except json.JSONDecodeError:
                continue

transcripts = [e["data"]["text"] for e in events if e.get("channel") == "transcript"]
cutoff = time.time() - 300
recent_visual = [
    e for e in events
    if e.get("channel") == "visual_index" and e["unix_ts"] > cutoff
]
```

## 追加ドキュメント

リファレンスドキュメントはこのSKILL.mdファイルに隣接する`reference/`ディレクトリにあります。必要に応じてGlobツールで場所を特定してください。

- [reference/api-reference.md](reference/api-reference.md) - 完全なVideoDB Python SDK APIリファレンス
- [reference/search.md](reference/search.md) - 動画検索（音声・シーンベース）の詳細ガイド
- [reference/editor.md](reference/editor.md) - タイムライン編集、アセット、コンポジション
- [reference/streaming.md](reference/streaming.md) - HLSストリーミングとインスタント再生
- [reference/generative.md](reference/generative.md) - AI搭載メディア生成（画像、動画、音声）
- [reference/rtstream.md](reference/rtstream.md) - ライブストリーム取り込みワークフロー（RTSP/RTMP）
- [reference/rtstream-reference.md](reference/rtstream-reference.md) - RTStream SDKメソッドとAIパイプライン
- [reference/capture.md](reference/capture.md) - デスクトップキャプチャワークフロー
- [reference/capture-reference.md](reference/capture-reference.md) - Capture SDKとWebSocketイベント
- [reference/use-cases.md](reference/use-cases.md) - 一般的な動画処理パターンと例

VideoDBが操作をサポートしている場合、**ffmpeg、moviepy、またはローカルエンコーディングツールを使用しないでください**。以下はすべてVideoDBによりサーバーサイドで処理されます — トリミング、クリップの結合、オーディオまたはミュージックのオーバーレイ、字幕の追加、テキスト/画像オーバーレイ、トランスコード、解像度変更、アスペクト比変換、プラットフォーム要件に合わせたリサイズ、書き起こし、メディア生成。reference/editor.mdのLimitationsに記載されている操作（トランジション、速度変更、クロップ/ズーム、カラーグレーディング、ボリュームミキシング）のみローカルツールにフォールバックしてください。

### 用途別の使い分け

| 問題 | VideoDBソリューション |
|---------|-----------------|
| プラットフォームが動画のアスペクト比や解像度を拒否 | `video.reframe()`または`conn.transcode()`と`VideoConfig` |
| Twitter/Instagram/TikTok向けに動画をリサイズ | `video.reframe(target="vertical")`または`target="square"` |
| 解像度の変更（例: 1080p → 720p） | `conn.transcode()`と`VideoConfig(resolution=720)` |
| 動画にオーディオ/ミュージックをオーバーレイ | `Timeline`上の`AudioAsset` |
| 字幕の追加 | `video.add_subtitle()`または`CaptionAsset` |
| クリップの結合/トリミング | `Timeline`上の`VideoAsset` |
| ナレーション、ミュージック、SFXの生成 | `coll.generate_voice()`、`generate_music()`、`generate_sound_effect()` |

## 出自

このスキルのリファレンス資料は`skills/videodb/reference/`にローカルにベンダリングされています。
実行時に外部リポジトリリンクを辿るのではなく、上記のローカルコピーを使用してください。
