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
# C++ テスト

> このファイルは [common/testing.md](../common/testing.md) をC++固有の内容で拡張します。

## フレームワーク

**GoogleTest**（gtest/gmock）と**CMake/CTest**を使用する。

## テストの実行

```bash
cmake --build build && ctest --test-dir build --output-on-failure
```

## カバレッジ

```bash
cmake -DCMAKE_CXX_FLAGS="--coverage" -DCMAKE_EXE_LINKER_FLAGS="--coverage" ..
cmake --build .
ctest --output-on-failure
lcov --capture --directory . --output-file coverage.info
```

## サニタイザ

CIでは必ずサニタイザ付きでテストを実行する:

```bash
cmake -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined" ..
```

## 参考

スキル: `cpp-testing` で詳細なC++テストパターン、TDDワークフロー、GoogleTest/GMockの使い方を参照してください。
