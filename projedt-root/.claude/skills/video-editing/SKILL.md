---
name: video-editing
description: AI支援の動画編集ワークフロー。実際の映像のカット、構造化、拡張を対象とする。FFmpeg、Remotion、ElevenLabs、fal.aiを通じたrawキャプチャからの完全なパイプラインと、DescriptまたはCapCutでの最終仕上げをカバー。動画の編集、フッテージのカット、Vlogの作成、動画コンテンツの制作に使用。
origin: ECC
---

# 動画編集

実際の映像のAI支援編集。プロンプトからの生成ではなく、既存の動画を高速に編集します。

## アクティベーション条件

- ユーザーが動画フッテージの編集、カット、構造化を望むとき
- 長い録画をショートフォームコンテンツに変換するとき
- rawキャプチャからVlog、チュートリアル、デモ動画を作成するとき
- 既存の動画にオーバーレイ、字幕、音楽、ナレーションを追加するとき
- 異なるプラットフォーム（YouTube、TikTok、Instagram）向けに動画をリフレーミングするとき
- ユーザーが「edit video」「cut this footage」「make a vlog」「video workflow」と言ったとき

## コアテーゼ

AI動画編集は、動画全体の作成を求めるのをやめて、実際の映像の圧縮、構造化、拡張に使い始めると有用です。価値は生成ではなく圧縮にあります。

## パイプライン

```
Screen Studio / rawフッテージ
  → Claude / Codex
  → FFmpeg
  → Remotion
  → ElevenLabs / fal.ai
  → Descript or CapCut
```

各レイヤーには特定の役割があります。レイヤーをスキップしないでください。1つのツールにすべてをさせようとしないでください。

## レイヤー1: キャプチャ（Screen Studio / Rawフッテージ）

ソース素材を収集:
- **Screen Studio**: アプリデモ、コーディングセッション、ブラウザワークフロー用の洗練されたスクリーン録画
- **Rawカメラフッテージ**: Vlogフッテージ、インタビュー、イベント録画
- **VideoDBによるデスクトップキャプチャ**: リアルタイムコンテキスト付きセッション録画（`videodb`スキルを参照）

出力: 整理準備完了のrawファイル。

## レイヤー2: 整理（Claude / Codex）

Claude CodeまたはCodexを使用して:
- **書き起こしとラベル付け**: トランスクリプトを生成し、トピックとテーマを特定
- **構造の計画**: 何を残し、何をカットし、どの順序が適切かを決定
- **デッドセクションの特定**: ポーズ、脱線、テイクの繰り返しを見つける
- **編集決定リストの生成**: カットのタイムスタンプ、保持するセグメント
- **FFmpegとRemotionコードの生成**: コマンドとコンポジションを生成

```
プロンプト例:
"Here's the transcript of a 4-hour recording. Identify the 8 strongest segments
for a 24-minute vlog. Give me FFmpeg cut commands for each segment."
```

このレイヤーは構造についてであり、最終的なクリエイティブセンスではありません。

## レイヤー3: 決定論的カット（FFmpeg）

FFmpegは退屈だが重要な作業を処理: 分割、トリミング、結合、前処理。

### タイムスタンプによるセグメント抽出

```bash
ffmpeg -i raw.mp4 -ss 00:12:30 -to 00:15:45 -c copy segment_01.mp4
```

### 編集決定リストからのバッチカット

```bash
#!/bin/bash
# cuts.txt: start,end,label
while IFS=, read -r start end label; do
  ffmpeg -i raw.mp4 -ss "$start" -to "$end" -c copy "segments/${label}.mp4"
done < cuts.txt
```

### セグメントの結合

```bash
# Create file list
for f in segments/*.mp4; do echo "file '$f'"; done > concat.txt
ffmpeg -f concat -safe 0 -i concat.txt -c copy assembled.mp4
```

### より高速な編集のためのプロキシ作成

```bash
ffmpeg -i raw.mp4 -vf "scale=960:-2" -c:v libx264 -preset ultrafast -crf 28 proxy.mp4
```

### 書き起こし用の音声抽出

```bash
ffmpeg -i raw.mp4 -vn -acodec pcm_s16le -ar 16000 audio.wav
```

### 音声レベルの正規化

```bash
ffmpeg -i segment.mp4 -af loudnorm=I=-16:TP=-1.5:LRA=11 -c:v copy normalized.mp4
```

## レイヤー4: プログラム可能なコンポジション（Remotion）

Remotionは編集の問題をコンポーザブルなコードに変えます。従来のエディタでは面倒なことに使用:

### Remotionを使用するタイミング

- オーバーレイ: テキスト、画像、ブランディング、ロワーサード
- データビジュアライゼーション: チャート、統計、アニメーション数値
- モーショングラフィックス: トランジション、説明アニメーション
- コンポーザブルシーン: 動画間で再利用可能なテンプレート
- プロダクトデモ: アノテーション付きスクリーンショット、UIハイライト

### 基本的なRemotionコンポジション

```tsx
import { AbsoluteFill, Sequence, Video, useCurrentFrame } from "remotion";

export const VlogComposition: React.FC = () => {
  const frame = useCurrentFrame();

  return (
    <AbsoluteFill>
      {/* Main footage */}
      <Sequence from={0} durationInFrames={300}>
        <Video src="/segments/intro.mp4" />
      </Sequence>

      {/* Title overlay */}
      <Sequence from={30} durationInFrames={90}>
        <AbsoluteFill style={{
          justifyContent: "center",
          alignItems: "center",
        }}>
          <h1 style={{
            fontSize: 72,
            color: "white",
            textShadow: "2px 2px 8px rgba(0,0,0,0.8)",
          }}>
            The AI Editing Stack
          </h1>
        </AbsoluteFill>
      </Sequence>

      {/* Next segment */}
      <Sequence from={300} durationInFrames={450}>
        <Video src="/segments/demo.mp4" />
      </Sequence>
    </AbsoluteFill>
  );
};
```

### 出力のレンダリング

```bash
npx remotion render src/index.ts VlogComposition output.mp4
```

詳細なパターンとAPIリファレンスは[Remotion docs](https://www.remotion.dev/docs)を参照。

## レイヤー5: 生成アセット（ElevenLabs / fal.ai）

必要なものだけを生成。動画全体を生成しないでください。

### ElevenLabsによるナレーション

```python
import os
import requests

resp = requests.post(
    f"https://api.elevenlabs.io/v1/text-to-speech/{voice_id}",
    headers={
        "xi-api-key": os.environ["ELEVENLABS_API_KEY"],
        "Content-Type": "application/json"
    },
    json={
        "text": "Your narration text here",
        "model_id": "eleven_turbo_v2_5",
        "voice_settings": {"stability": 0.5, "similarity_boost": 0.75}
    }
)
with open("voiceover.mp3", "wb") as f:
    f.write(resp.content)
```

### fal.aiによる音楽とSFX

`fal-ai-media`スキルを使用:
- バックグラウンドミュージックの生成
- サウンドエフェクト（動画から音声へのThinkSoundモデル）
- トランジションサウンド

### fal.aiによる生成ビジュアル

存在しないインサートショット、サムネイル、Bロールに使用:
```
generate(app_id: "fal-ai/nano-banana-pro", input_data: {
  "prompt": "professional thumbnail for tech vlog, dark background, code on screen",
  "image_size": "landscape_16_9"
})
```

### VideoDBによる生成オーディオ

VideoDBが設定されている場合:
```python
voiceover = coll.generate_voice(text="Narration here", voice="alloy")
music = coll.generate_music(prompt="lo-fi background for coding vlog", duration=120)
sfx = coll.generate_sound_effect(prompt="subtle whoosh transition")
```

## レイヤー6: 最終仕上げ（Descript / CapCut）

最後のレイヤーは人間が行います。従来のエディタを使用:
- **ペーシング**: 速すぎる/遅すぎるカットの調整
- **キャプション**: 自動生成後、手動でクリーン
- **カラーグレーディング**: 基本的な補正とムード
- **最終オーディオミックス**: ボイス、音楽、SFXレベルのバランス
- **エクスポート**: プラットフォーム固有のフォーマットと品質設定

ここにセンスがあります。AIは反復的な作業を片付けます。最終的な判断はあなたが行います。

## ソーシャルメディアのリフレーミング

異なるプラットフォームには異なるアスペクト比が必要:

| プラットフォーム | アスペクト比 | 解像度 |
|----------|-------------|------------|
| YouTube | 16:9 | 1920x1080 |
| TikTok / Reels | 9:16 | 1080x1920 |
| Instagram Feed | 1:1 | 1080x1080 |
| X / Twitter | 16:9 or 1:1 | 1280x720 or 720x720 |

### FFmpegによるリフレーミング

```bash
# 16:9 to 9:16 (center crop)
ffmpeg -i input.mp4 -vf "crop=ih*9/16:ih,scale=1080:1920" vertical.mp4

# 16:9 to 1:1 (center crop)
ffmpeg -i input.mp4 -vf "crop=ih:ih,scale=1080:1080" square.mp4
```

### VideoDBによるリフレーミング

```python
from videodb import ReframeMode

# Smart reframe (AI-guided subject tracking)
reframed = video.reframe(start=0, end=60, target="vertical", mode=ReframeMode.smart)
```

## シーン検出とオートカット

### FFmpegによるシーン検出

```bash
# Detect scene changes (threshold 0.3 = moderate sensitivity)
ffmpeg -i input.mp4 -vf "select='gt(scene,0.3)',showinfo" -vsync vfr -f null - 2>&1 | grep showinfo
```

### オートカットのための無音検出

```bash
# Find silent segments (useful for cutting dead air)
ffmpeg -i input.mp4 -af silencedetect=noise=-30dB:d=2 -f null - 2>&1 | grep silence
```

### ハイライト抽出

Claudeを使用してトランスクリプト + シーンタイムスタンプを分析:
```
"Given this transcript with timestamps and these scene change points,
identify the 5 most engaging 30-second clips for social media."
```

## 各ツールの得意分野

| ツール | 強み | 弱み |
|------|----------|----------|
| Claude / Codex | 整理、計画、コード生成 | クリエイティブセンスのレイヤーではない |
| FFmpeg | 決定論的カット、バッチ処理、フォーマット変換 | ビジュアル編集UIなし |
| Remotion | プログラム可能なオーバーレイ、コンポーザブルシーン、再利用可能テンプレート | 非開発者には学習曲線 |
| Screen Studio | 洗練されたスクリーン録画を即座に | スクリーンキャプチャのみ |
| ElevenLabs | ボイス、ナレーション、音楽、SFX | ワークフローの中心ではない |
| Descript / CapCut | 最終ペーシング、キャプション、仕上げ | 手動、自動化不可 |

## 主要原則

1. **生成ではなく編集。** このワークフローは実際のフッテージのカット用で、プロンプトからの作成用ではない。
2. **スタイルの前に構造。** ビジュアルに触れる前にレイヤー2でストーリーを正しくする。
3. **FFmpegがバックボーン。** 退屈だが重要。長いフッテージを管理可能にする場所。
4. **再現性のためのRemotion。** 1回以上行うなら、Remotionコンポーネントにする。
5. **選択的に生成。** 存在しないアセットにのみAI生成を使用し、すべてには使わない。
6. **センスは最後のレイヤー。** AIは反復的な作業を片付ける。最終的なクリエイティブ判断はあなたが行う。

## 関連スキル

- `fal-ai-media` — AI画像、動画、音声生成
- `videodb` — サーバーサイド動画処理、インデックス、ストリーミング
- `content-engine` — プラットフォームネイティブのコンテンツ配信
