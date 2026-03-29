---
paths:
  - "**/*.py"
  - "**/*.pyi"
---
# Python パターン

> このファイルは [common/patterns.md](../common/patterns.md) を Python 固有の内容で拡張します。

## Protocol（ダックタイピング）

```python
from typing import Protocol

class Repository(Protocol):
    def find_by_id(self, id: str) -> dict | None: ...
    def save(self, entity: dict) -> dict: ...
```

## DTO としての Dataclass

```python
from dataclasses import dataclass

@dataclass
class CreateUserRequest:
    name: str
    email: str
    age: int | None = None
```

## コンテキストマネージャとジェネレータ

- リソース管理にはコンテキストマネージャ（`with` 文）を使用
- 遅延評価とメモリ効率の良いイテレーションにはジェネレータを使用

## 参考

スキル `python-patterns` にデコレータ、並行処理、パッケージ構成を含む包括的なパターンがあります。
