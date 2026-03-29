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
# C++ パターン

> このファイルは [common/patterns.md](../common/patterns.md) をC++固有の内容で拡張します。

## RAII (Resource Acquisition Is Initialization)

リソースのライフタイムをオブジェクトのライフタイムに紐付ける:

```cpp
class FileHandle {
public:
    explicit FileHandle(const std::string& path) : file_(std::fopen(path.c_str(), "r")) {}
    ~FileHandle() { if (file_) std::fclose(file_); }
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
private:
    std::FILE* file_;
};
```

## 5の規則/0の規則

- **0の規則**: カスタムデストラクタ、コピー/ムーブコンストラクタ、代入演算子が不要なクラスを優先する
- **5の規則**: デストラクタ/コピーコンストラクタ/コピー代入/ムーブコンストラクタ/ムーブ代入のいずれかを定義する場合、5つすべてを定義する

## 値セマンティクス

- 小さい/トリビアルな型は値渡しする
- 大きい型は `const&` で渡す
- 値で返す（RVO/NRVOに依存する）
- シンクパラメータにはムーブセマンティクスを使用する

## エラーハンドリング

- 例外的な状況には例外を使用する
- 存在しない可能性がある値には `std::optional` を使用する
- 想定されるエラーには `std::expected`（C++23）またはresult型を使用する

## 参考

スキル: `cpp-coding-standards` で包括的なC++パターンとアンチパターンを参照してください。
