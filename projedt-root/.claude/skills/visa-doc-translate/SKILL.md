---
name: visa-doc-translate
description: ビザ申請書類（画像）を英語に翻訳し、原本と翻訳を含むバイリンガルPDFを作成
---

ビザ申請のための書類翻訳をサポートします。

## 手順

ユーザーが画像ファイルパスを提供した場合、確認を求めずに以下のステップを自動的に実行してください:

1. **画像変換**: ファイルがHEICの場合、`sips -s format png <input> --out <output>`を使用してPNGに変換

2. **画像回転**:
   - EXIFオリエンテーションデータを確認
   - EXIFデータに基づいて画像を自動回転
   - EXIFオリエンテーションが6の場合、反時計回りに90度回転
   - 必要に応じて追加の回転を適用（書類が逆さまの場合は180度をテスト）

3. **OCRテキスト抽出**:
   - 複数のOCR方法を自動的に試行:
     - macOS Visionフレームワーク（macOSで推奨）
     - EasyOCR（クロスプラットフォーム、tesseract不要）
     - Tesseract OCR（利用可能な場合）
   - 書類からすべてのテキスト情報を抽出
   - 書類の種類を特定（預金証明書、在職証明書、退職証明書など）

4. **翻訳**:
   - すべてのテキスト内容をプロフェッショナルに英語に翻訳
   - 元の書類の構造とフォーマットを維持
   - ビザ申請に適切なプロフェッショナルな用語を使用
   - 固有名詞は原語で保持し、英語を括弧内に記載
   - 中国語の名前はピンイン形式を使用（例: WU Zhengye）
   - すべての数字、日付、金額を正確に保持

5. **PDF生成**:
   - PILとreportlabライブラリを使用してPythonスクリプトを作成
   - ページ1: 回転した原本画像をA4ページに合わせて中央配置・スケーリングして表示
   - ページ2: 適切なフォーマットで英語翻訳を表示:
     - タイトルは中央揃えで太字
     - コンテンツは左揃えで適切な間隔
     - 公式文書に適したプロフェッショナルなレイアウト
   - 下部にノートを追加: 「This is a certified English translation of the original document」
   - スクリプトを実行してPDFを生成

6. **出力**: 同じディレクトリに`<original_filename>_Translated.pdf`という名前のPDFファイルを作成

## 対応書類

- 銀行預金証明書（存款证明）
- 収入証明書（收入证明）
- 在職証明書（在职证明）
- 退職証明書（退休证明）
- 不動産証明書（房产证明）
- 営業許可証（营业执照）
- 身分証明書とパスポート
- その他の公的書類

## 技術実装

### OCR方法（試行順）

1. **macOS Visionフレームワーク**（macOSのみ）:
   ```python
   import Vision
   from Foundation import NSURL
   ```

2. **EasyOCR**（クロスプラットフォーム）:
   ```bash
   pip install easyocr
   ```

3. **Tesseract OCR**（利用可能な場合）:
   ```bash
   brew install tesseract tesseract-lang
   pip install pytesseract
   ```

### 必要なPythonライブラリ

```bash
pip install pillow reportlab
```

macOS Visionフレームワーク用:
```bash
pip install pyobjc-framework-Vision pyobjc-framework-Quartz
```

## 重要なガイドライン

- 各ステップでユーザーの確認を求めないこと
- 最適な回転角度を自動的に判断する
- 1つのOCR方法が失敗した場合、複数の方法を試行する
- すべての数字、日付、金額が正確に翻訳されていることを確認する
- クリーンでプロフェッショナルなフォーマットを使用する
- プロセス全体を完了し、最終的なPDFの場所を報告する

## 使用例

```bash
/visa-doc-translate RetirementCertificate.PNG
/visa-doc-translate BankStatement.HEIC
/visa-doc-translate EmploymentLetter.jpg
```

## 出力例

このスキルは以下を行います:
1. 利用可能なOCR方法を使用してテキストを抽出
2. プロフェッショナルな英語に翻訳
3. 以下を含む`<filename>_Translated.pdf`を生成:
   - ページ1: 元の書類画像
   - ページ2: プロフェッショナルな英語翻訳

オーストラリア、アメリカ、カナダ、イギリスなど、翻訳書類が必要な国へのビザ申請に最適です。
