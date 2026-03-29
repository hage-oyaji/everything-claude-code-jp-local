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
# C++ セキュリティ

> このファイルは [common/security.md](../common/security.md) をC++固有の内容で拡張します。

## メモリ安全性

- 生の `new`/`delete` は使用しない — スマートポインタを使用する
- C言語スタイルの配列は使用しない — `std::array` または `std::vector` を使用する
- `malloc`/`free` は使用しない — C++のアロケーションを使用する
- 絶対に必要な場合を除き `reinterpret_cast` は避ける

## バッファオーバーフロー

- `char*` よりも `std::string` を使用する
- 安全性が重要な場合は境界チェック付きの `.at()` を使用する
- `strcpy`、`strcat`、`sprintf` は使用しない — `std::string` または `fmt::format` を使用する

## 未定義動作

- 変数は必ず初期化する
- 符号付き整数のオーバーフローを避ける
- nullポインタやダングリングポインタのデリファレンスは行わない
- CIでサニタイザを使用する:
  ```bash
  cmake -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined" ..
  ```

## 静的解析

- 自動チェックに**clang-tidy**を使用する:
  ```bash
  clang-tidy --checks='*' src/*.cpp
  ```
- 追加の解析に**cppcheck**を使用する:
  ```bash
  cppcheck --enable=all src/
  ```

## 参考

スキル: `cpp-coding-standards` で詳細なセキュリティガイドラインを参照してください。
