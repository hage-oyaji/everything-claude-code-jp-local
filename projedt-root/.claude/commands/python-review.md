---
description: PEP 8準拠、型ヒント、セキュリティ、Pythonicなイディオムに関する包括的なPythonコードレビュー。python-reviewerエージェントを呼び出します。
---

# Pythonコードレビュー

このコマンドは、Python固有の包括的なコードレビューのために**python-reviewer**エージェントを呼び出します。

## このコマンドの動作

1. **Python変更の特定**: `git diff`で変更された`.py`ファイルを検出
2. **静的解析の実行**: `ruff`、`mypy`、`pylint`、`black --check`を実行
3. **セキュリティスキャン**: SQLインジェクション、コマンドインジェクション、安全でないデシリアライゼーションをチェック
4. **型安全性レビュー**: 型ヒントとmypyエラーを分析
5. **Pythonicコードチェック**: PEP 8とPythonのベストプラクティスへの準拠を検証
6. **レポート生成**: 重要度別に問題を分類

## 使用するタイミング

`/python-review`は以下の場合に使用してください：
- Pythonコードを作成または変更した後
- Pythonの変更をコミットする前
- Pythonコードを含むプルリクエストのレビュー時
- 新しいPythonコードベースへのオンボーディング時
- Pythonicなパターンとイディオムの学習時

## レビューカテゴリ

### CRITICAL（修正必須）
- SQL/コマンドインジェクションの脆弱性
- 安全でないeval/execの使用
- Pickleの安全でないデシリアライゼーション
- ハードコードされた認証情報
- YAMLの安全でないロード
- エラーを隠すbare except句

### HIGH（修正推奨）
- パブリック関数の型ヒント欠落
- ミュータブルなデフォルト引数
- 例外の黙殺
- リソースに対するコンテキストマネージャーの未使用
- リスト内包表記の代わりにCスタイルのループ
- isinstance()の代わりにtype()を使用
- ロックなしの競合状態

### MEDIUM（検討事項）
- PEP 8フォーマット違反
- パブリック関数のdocstring欠落
- ロギングの代わりにprint文を使用
- 非効率な文字列操作
- 名前付き定数なしのマジックナンバー
- フォーマットにf-stringを使用していない
- 不要なリスト生成

## 自動チェックの実行

```bash
# Type checking
mypy .

# Linting and formatting
ruff check .
black --check .
isort --check-only .

# Security scanning
bandit -r .

# Dependency audit
pip-audit
safety check

# Testing
pytest --cov=app --cov-report=term-missing
```

## 使用例

```text
User: /python-review

Agent:
# Python Code Review Report

## Files Reviewed
- app/routes/user.py (modified)
- app/services/auth.py (modified)

## Static Analysis Results
✓ ruff: No issues
✓ mypy: No errors
⚠️ black: 2 files need reformatting
✓ bandit: No security issues

## Issues Found

[CRITICAL] SQL Injection vulnerability
File: app/routes/user.py:42
Issue: User input directly interpolated into SQL query
```python
query = f"SELECT * FROM users WHERE id = {user_id}"  # Bad
```
Fix: Use parameterized query
```python
query = "SELECT * FROM users WHERE id = %s"  # Good
cursor.execute(query, (user_id,))
```

[HIGH] Mutable default argument
File: app/services/auth.py:18
Issue: Mutable default argument causes shared state
```python
def process_items(items=[]):  # Bad
    items.append("new")
    return items
```
Fix: Use None as default
```python
def process_items(items=None):  # Good
    if items is None:
        items = []
    items.append("new")
    return items
```

[MEDIUM] Missing type hints
File: app/services/auth.py:25
Issue: Public function without type annotations
```python
def get_user(user_id):  # Bad
    return db.find(user_id)
```
Fix: Add type hints
```python
def get_user(user_id: str) -> Optional[User]:  # Good
    return db.find(user_id)
```

[MEDIUM] Not using context manager
File: app/routes/user.py:55
Issue: File not closed on exception
```python
f = open("config.json")  # Bad
data = f.read()
f.close()
```
Fix: Use context manager
```python
with open("config.json") as f:  # Good
    data = f.read()
```

## Summary
- CRITICAL: 1
- HIGH: 1
- MEDIUM: 2

Recommendation: ❌ Block merge until CRITICAL issue is fixed

## Formatting Required
Run: `black app/routes/user.py app/services/auth.py`
```

## 承認基準

| ステータス | 条件 |
|--------|-----------|
| ✅ 承認 | CRITICALまたはHIGHの問題がない |
| ⚠️ 警告 | MEDIUMの問題のみ（注意してマージ） |
| ❌ ブロック | CRITICALまたはHIGHの問題が検出された |

## 他のコマンドとの連携

- まず`/tdd`を使用してテストが通ることを確認
- Python固有でない問題には`/code-review`を使用
- コミット前に`/python-review`を使用
- 静的解析ツールが失敗した場合は`/build-fix`を使用

## フレームワーク固有のレビュー

### Djangoプロジェクト
レビューアーがチェックする項目：
- N+1クエリの問題（`select_related`と`prefetch_related`を使用）
- モデル変更に対するマイグレーションの欠落
- ORMで対応可能な場合の生SQLの使用
- 複数ステップ操作での`transaction.atomic()`の欠落

### FastAPIプロジェクト
レビューアーがチェックする項目：
- CORSの設定ミス
- リクエストバリデーション用のPydanticモデル
- レスポンスモデルの正確性
- 適切なasync/awaitの使用
- 依存性注入パターン

### Flaskプロジェクト
レビューアーがチェックする項目：
- コンテキスト管理（アプリコンテキスト、リクエストコンテキスト）
- 適切なエラーハンドリング
- Blueprintの構成
- 設定管理

## 関連

- エージェント：`agents/python-reviewer.md`
- スキル：`skills/python-patterns/`、`skills/python-testing/`

## よくある修正

### 型ヒントの追加
```python
# Before
def calculate(x, y):
    return x + y

# After
from typing import Union

def calculate(x: Union[int, float], y: Union[int, float]) -> Union[int, float]:
    return x + y
```

### コンテキストマネージャーの使用
```python
# Before
f = open("file.txt")
data = f.read()
f.close()

# After
with open("file.txt") as f:
    data = f.read()
```

### リスト内包表記の使用
```python
# Before
result = []
for item in items:
    if item.active:
        result.append(item.name)

# After
result = [item.name for item in items if item.active]
```

### ミュータブルなデフォルトの修正
```python
# Before
def append(value, items=[]):
    items.append(value)
    return items

# After
def append(value, items=None):
    if items is None:
        items = []
    items.append(value)
    return items
```

### f-stringの使用（Python 3.6+）
```python
# Before
name = "Alice"
greeting = "Hello, " + name + "!"
greeting2 = "Hello, {}".format(name)

# After
greeting = f"Hello, {name}!"
```

### ループ内での文字列連結の修正
```python
# Before
result = ""
for item in items:
    result += str(item)

# After
result = "".join(str(item) for item in items)
```

## Pythonバージョン互換性

レビューアーは新しいPythonバージョンの機能を使用しているコードを記録します：

| 機能 | 最小Python |
|---------|----------------|
| 型ヒント | 3.5+ |
| f-string | 3.6+ |
| セイウチ演算子（`:=`） | 3.8+ |
| 位置専用パラメータ | 3.8+ |
| match文 | 3.10+ |
| 型ユニオン（`x | None`） | 3.10+ |

プロジェクトの`pyproject.toml`または`setup.py`で正しい最小Pythonバージョンを指定してください。
