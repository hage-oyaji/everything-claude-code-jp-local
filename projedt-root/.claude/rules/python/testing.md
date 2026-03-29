---
paths:
  - "**/*.py"
  - "**/*.pyi"
---
# Python テスト

> このファイルは [common/testing.md](../common/testing.md) を Python 固有の内容で拡張します。

## フレームワーク

テストフレームワークとして **pytest** を使用。

## カバレッジ

```bash
pytest --cov=src --cov-report=term-missing
```

## テスト構成

テスト分類に `pytest.mark` を使用：

```python
import pytest

@pytest.mark.unit
def test_calculate_total():
    ...

@pytest.mark.integration
def test_database_connection():
    ...
```

## 参考

スキル `python-testing` に pytest パターンとフィクスチャの詳細があります。
