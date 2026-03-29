---
paths:
  - "**/*.py"
  - "**/*.pyi"
---
# Python コーディングスタイル

> このファイルは [common/coding-style.md](../common/coding-style.md) を Python 固有の内容で拡張します。

## 標準

- **PEP 8** 規約に従う
- すべての関数シグネチャに**型アノテーション**を使用

## イミュータビリティ

イミュータブルなデータ構造を優先：

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class User:
    name: str
    email: str

from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float
```

## フォーマット

- コードフォーマットには **black**
- インポートソートには **isort**
- リンティングには **ruff**

## 参考

スキル `python-patterns` に Python のイディオムとパターンの詳細があります。
