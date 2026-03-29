---
paths:
  - "**/*.cpp"
  - "**/*.hpp"
  - "**/*.cc"
  - "**/*.hh"
  - "**/*.cxx"
  - "**/*.h"
  - "**/CMakeLists.txt"
---
# C++ フック

> このファイルは [common/hooks.md](../common/hooks.md) をC++固有の内容で拡張します。

## ビルドフック

C++の変更をコミットする前に、以下のチェックを実行してください:

```bash
# フォーマットチェック
clang-format --dry-run --Werror src/*.cpp src/*.hpp

# 静的解析
clang-tidy src/*.cpp -- -std=c++17

# ビルド
cmake --build build

# テスト
ctest --test-dir build --output-on-failure
```

## 推奨CIパイプライン

1. **clang-format** — フォーマットチェック
2. **clang-tidy** — 静的解析
3. **cppcheck** — 追加の解析
4. **cmake build** — コンパイル
5. **ctest** — サニタイザ付きテスト実行
