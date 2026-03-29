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
# C++ コーディングスタイル

> このファイルは [common/coding-style.md](../common/coding-style.md) をC++固有の内容で拡張します。

## モダンC++ (C++17/20/23)

- C言語スタイルの構文よりも**モダンC++の機能**を優先する
- 型がコンテキストから明らかな場合は `auto` を使用する
- コンパイル時定数には `constexpr` を使用する
- 構造化束縛を使用する: `auto [key, value] = map_entry;`

## リソース管理

- **あらゆる場所でRAII** — 手動の `new`/`delete` は使用しない
- 排他的所有権には `std::unique_ptr` を使用する
- 共有所有権が本当に必要な場合のみ `std::shared_ptr` を使用する
- 生の `new` よりも `std::make_unique` / `std::make_shared` を使用する

## 命名規則

- 型/クラス: `PascalCase`
- 関数/メソッド: `snake_case` または `camelCase`（プロジェクトの規約に従う）
- 定数: `kPascalCase` または `UPPER_SNAKE_CASE`
- 名前空間: `lowercase`
- メンバ変数: `snake_case_`（末尾アンダースコア）または `m_` プレフィックス

## フォーマット

- **clang-format** を使用する — スタイルの議論は不要
- コミット前に `clang-format -i <file>` を実行する

## 参考

スキル: `cpp-coding-standards` で包括的なC++コーディング標準とガイドラインを参照してください。
