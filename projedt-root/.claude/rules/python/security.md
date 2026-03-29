---
paths:
  - "**/*.py"
  - "**/*.pyi"
---
# Python セキュリティ

> このファイルは [common/security.md](../common/security.md) を Python 固有の内容で拡張します。

## シークレット管理

```python
import os
from dotenv import load_dotenv

load_dotenv()

api_key = os.environ["OPENAI_API_KEY"]  # Raises KeyError if missing
```

## セキュリティスキャン

- 静的セキュリティ解析には **bandit** を使用：
  ```bash
  bandit -r src/
  ```

## 参考

スキル `django-security` に Django 固有のセキュリティガイドライン（該当する場合）があります。
