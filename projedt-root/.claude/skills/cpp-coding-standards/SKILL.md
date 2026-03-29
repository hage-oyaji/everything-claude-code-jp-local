---
name: cpp-coding-standards
description: C++ Core Guidelines（isocpp.github.io）に基づくC++コーディング規約。モダンで安全かつイディオマティックなプラクティスを強制するために、C++コードの記述、レビュー、リファクタリング時に使用。
origin: ECC
---

# C++コーディング規約（C++ Core Guidelines）

[C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)に基づく、モダンC++（C++17/20/23）のための包括的コーディング規約。型安全性、リソース安全性、イミュータビリティ、明確さを強制する。

## 使用するとき

- 新しいC++コード（クラス、関数、テンプレート）を書くとき
- 既存のC++コードをレビューまたはリファクタリングするとき
- C++プロジェクトでアーキテクチャ上の判断を行うとき
- C++コードベース全体で一貫したスタイルを強制するとき
- 言語機能の選択（例: `enum` vs `enum class`、生ポインタ vs スマートポインタ）

### 使用しないとき

- C++以外のプロジェクト
- モダンC++機能を採用できないレガシーCコードベース
- 特定のガイドラインがハードウェア制約と矛盾する組込み/ベアメタルコンテキスト（選択的に適応する）

## 横断的原則

これらのテーマはガイドライン全体で繰り返し現れ、基盤を形成する:

1. **あらゆる場面でRAII** (P.8, R.1, E.6, CP.20): リソースのライフタイムをオブジェクトのライフタイムに結びつける
2. **デフォルトでイミュータブル** (P.10, Con.1-5, ES.25): `const`/`constexpr`から始め、ミュータビリティは例外
3. **型安全性** (P.4, I.4, ES.46-49, Enum.3): 型システムを使用してコンパイル時にエラーを防ぐ
4. **意図を表現する** (P.3, F.1, NL.1-2, T.10): 名前、型、コンセプトが目的を伝えるべき
5. **複雑さを最小化する** (F.2-3, ES.5, Per.4-5): シンプルなコードは正しいコード
6. **ポインタセマンティクスより値セマンティクス** (C.10, R.3-5, F.20, CP.31): 値返しとスコープ付きオブジェクトを優先

## 哲学とインターフェース (P.*, I.*)

### 主要ルール

| ルール | 要約 |
|------|---------|
| **P.1** | アイデアをコードで直接表現する |
| **P.3** | 意図を表現する |
| **P.4** | 理想的にはプログラムは静的に型安全であるべき |
| **P.5** | 実行時チェックよりコンパイル時チェックを優先 |
| **P.8** | リソースをリークさせない |
| **P.10** | ミュータブルデータよりイミュータブルデータを優先 |
| **I.1** | インターフェースを明示的にする |
| **I.2** | 非constグローバル変数を避ける |
| **I.4** | インターフェースを正確かつ強く型付けする |
| **I.11** | 生ポインタや参照で所有権を移転しない |
| **I.23** | 関数引数の数を少なく保つ |

### DO

```cpp
// P.10 + I.4: Immutable, strongly typed interface
struct Temperature {
    double kelvin;
};

Temperature boil(const Temperature& water);
```

### DON'T

```cpp
// Weak interface: unclear ownership, unclear units
double boil(double* temp);

// Non-const global variable
int g_counter = 0;  // I.2 violation
```

## 関数 (F.*)

### 主要ルール

| ルール | 要約 |
|------|---------|
| **F.1** | 意味のある操作を慎重に名付けた関数としてパッケージ化 |
| **F.2** | 関数は単一の論理操作を実行すべき |
| **F.3** | 関数を短くシンプルに保つ |
| **F.4** | コンパイル時に評価できる可能性がある場合は`constexpr`を宣言 |
| **F.6** | 関数がスローしない場合は`noexcept`を宣言 |
| **F.8** | 純粋関数を優先 |
| **F.16** | 「入力」パラメータは、安価にコピーできる型は値渡し、それ以外は`const&`で渡す |
| **F.20** | 「出力」値には出力パラメータより戻り値を優先 |
| **F.21** | 複数の「出力」値を返すには構造体を返すことを優先 |
| **F.43** | ローカルオブジェクトへのポインタや参照を返さない |

### パラメータ渡し

```cpp
// F.16: Cheap types by value, others by const&
void print(int x);                           // cheap: by value
void analyze(const std::string& data);       // expensive: by const&
void transform(std::string s);               // sink: by value (will move)

// F.20 + F.21: Return values, not output parameters
struct ParseResult {
    std::string token;
    int position;
};

ParseResult parse(std::string_view input);   // GOOD: return struct

// BAD: output parameters
void parse(std::string_view input,
           std::string& token, int& pos);    // avoid this
```

### 純粋関数とconstexpr

```cpp
// F.4 + F.8: Pure, constexpr where possible
constexpr int factorial(int n) noexcept {
    return (n <= 1) ? 1 : n * factorial(n - 1);
}

static_assert(factorial(5) == 120);
```

### アンチパターン

- 関数から`T&&`を返す (F.45)
- `va_arg` / Cスタイル可変引数を使用する (F.55)
- 他のスレッドに渡すラムダで参照キャプチャする (F.53)
- `const T`を返す（ムーブセマンティクスを阻害する） (F.49)

## クラスとクラス階層 (C.*)

### 主要ルール

| ルール | 要約 |
|------|---------|
| **C.2** | 不変条件がある場合は`class`を使用; データメンバーが独立に変化する場合は`struct` |
| **C.9** | メンバーの公開を最小化する |
| **C.20** | デフォルト操作の定義を避けられるなら避ける（Rule of Zero） |
| **C.21** | コピー/ムーブ/デストラクタのいずれかを定義または`=delete`する場合、すべてを処理する（Rule of Five） |
| **C.35** | 基底クラスのデストラクタ: publicでvirtual、またはprotectedで非virtual |
| **C.41** | コンストラクタは完全に初期化されたオブジェクトを作成すべき |
| **C.46** | 単一引数のコンストラクタは`explicit`を宣言する |
| **C.67** | ポリモーフィッククラスはpublicなコピー/ムーブを抑制すべき |
| **C.128** | 仮想関数: `virtual`、`override`、`final`のいずれか1つだけを指定 |

### Rule of Zero

```cpp
// C.20: Let the compiler generate special members
struct Employee {
    std::string name;
    std::string department;
    int id;
    // No destructor, copy/move constructors, or assignment operators needed
};
```

### Rule of Five

```cpp
// C.21: If you must manage a resource, define all five
class Buffer {
public:
    explicit Buffer(std::size_t size)
        : data_(std::make_unique<char[]>(size)), size_(size) {}

    ~Buffer() = default;

    Buffer(const Buffer& other)
        : data_(std::make_unique<char[]>(other.size_)), size_(other.size_) {
        std::copy_n(other.data_.get(), size_, data_.get());
    }

    Buffer& operator=(const Buffer& other) {
        if (this != &other) {
            auto new_data = std::make_unique<char[]>(other.size_);
            std::copy_n(other.data_.get(), other.size_, new_data.get());
            data_ = std::move(new_data);
            size_ = other.size_;
        }
        return *this;
    }

    Buffer(Buffer&&) noexcept = default;
    Buffer& operator=(Buffer&&) noexcept = default;

private:
    std::unique_ptr<char[]> data_;
    std::size_t size_;
};
```

### クラス階層

```cpp
// C.35 + C.128: Virtual destructor, use override
class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;  // C.121: pure interface
};

class Circle : public Shape {
public:
    explicit Circle(double r) : radius_(r) {}
    double area() const override { return 3.14159 * radius_ * radius_; }

private:
    double radius_;
};
```

### アンチパターン

- コンストラクタ/デストラクタで仮想関数を呼ぶ (C.82)
- 非トリビアル型に`memset`/`memcpy`を使用する (C.90)
- 仮想関数とオーバーライダーで異なるデフォルト引数を提供する (C.140)
- データメンバーを`const`や参照にする（ムーブ/コピーを抑制する） (C.12)

## リソース管理 (R.*)

### 主要ルール

| ルール | 要約 |
|------|---------|
| **R.1** | RAIIを使用してリソースを自動管理する |
| **R.3** | 生ポインタ (`T*`) は非所有 |
| **R.5** | スコープ付きオブジェクトを優先; 不必要にヒープ確保しない |
| **R.10** | `malloc()`/`free()`を避ける |
| **R.11** | `new`と`delete`を明示的に呼ばない |
| **R.20** | `unique_ptr`または`shared_ptr`で所有権を表現する |
| **R.21** | 所有権を共有しない限り`shared_ptr`より`unique_ptr`を優先 |
| **R.22** | `shared_ptr`の作成には`make_shared()`を使用 |

### スマートポインタの使用

```cpp
// R.11 + R.20 + R.21: RAII with smart pointers
auto widget = std::make_unique<Widget>("config");  // unique ownership
auto cache  = std::make_shared<Cache>(1024);        // shared ownership

// R.3: Raw pointer = non-owning observer
void render(const Widget* w) {  // does NOT own w
    if (w) w->draw();
}

render(widget.get());
```

### RAIIパターン

```cpp
// R.1: Resource acquisition is initialization
class FileHandle {
public:
    explicit FileHandle(const std::string& path)
        : handle_(std::fopen(path.c_str(), "r")) {
        if (!handle_) throw std::runtime_error("Failed to open: " + path);
    }

    ~FileHandle() {
        if (handle_) std::fclose(handle_);
    }

    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    FileHandle(FileHandle&& other) noexcept
        : handle_(std::exchange(other.handle_, nullptr)) {}
    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) {
            if (handle_) std::fclose(handle_);
            handle_ = std::exchange(other.handle_, nullptr);
        }
        return *this;
    }

private:
    std::FILE* handle_;
};
```

### アンチパターン

- 裸の`new`/`delete` (R.11)
- C++コードでの`malloc()`/`free()` (R.10)
- 単一の式内での複数リソース確保 (R.13 -- 例外安全性の危険)
- `unique_ptr`で十分な場合の`shared_ptr` (R.21)

## 式と文 (ES.*)

### 主要ルール

| ルール | 要約 |
|------|---------|
| **ES.5** | スコープを小さく保つ |
| **ES.20** | オブジェクトを常に初期化する |
| **ES.23** | `{}`初期化構文を優先 |
| **ES.25** | 変更する意図がない限りオブジェクトを`const`または`constexpr`で宣言 |
| **ES.28** | `const`変数の複雑な初期化にはラムダを使用 |
| **ES.45** | マジック定数を避け、シンボリック定数を使用 |
| **ES.46** | ナローイング/ロッシー算術変換を避ける |
| **ES.47** | `0`や`NULL`ではなく`nullptr`を使用 |
| **ES.48** | キャストを避ける |
| **ES.50** | `const`をキャストで除去しない |

### 初期化

```cpp
// ES.20 + ES.23 + ES.25: Always initialize, prefer {}, default to const
const int max_retries{3};
const std::string name{"widget"};
const std::vector<int> primes{2, 3, 5, 7, 11};

// ES.28: Lambda for complex const initialization
const auto config = [&] {
    Config c;
    c.timeout = std::chrono::seconds{30};
    c.retries = max_retries;
    c.verbose = debug_mode;
    return c;
}();
```

### アンチパターン

- 未初期化変数 (ES.20)
- ポインタとして`0`や`NULL`を使用 (ES.47 -- `nullptr`を使用)
- Cスタイルキャスト (ES.48 -- `static_cast`、`const_cast`等を使用)
- `const`のキャスト除去 (ES.50)
- 名前付き定数なしのマジックナンバー (ES.45)
- 符号付きと符号なし算術の混在 (ES.100)
- ネストしたスコープでの名前の再利用 (ES.12)

## エラーハンドリング (E.*)

### 主要ルール

| ルール | 要約 |
|------|---------|
| **E.1** | 設計の早い段階でエラーハンドリング戦略を策定する |
| **E.2** | 関数が割り当てられたタスクを実行できないことを通知するために例外をスローする |
| **E.6** | リークを防ぐためにRAIIを使用する |
| **E.12** | スローが不可能または許容できない場合は`noexcept`を使用 |
| **E.14** | 目的に合わせて設計されたユーザー定義型を例外として使用 |
| **E.15** | 値でスロー、参照でキャッチ |
| **E.16** | デストラクタ、デアロケーション、swapは失敗してはならない |
| **E.17** | すべての関数ですべての例外をキャッチしようとしない |

### 例外階層

```cpp
// E.14 + E.15: Custom exception types, throw by value, catch by reference
class AppError : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};

class NetworkError : public AppError {
public:
    NetworkError(const std::string& msg, int code)
        : AppError(msg), status_code(code) {}
    int status_code;
};

void fetch_data(const std::string& url) {
    // E.2: Throw to signal failure
    throw NetworkError("connection refused", 503);
}

void run() {
    try {
        fetch_data("https://api.example.com");
    } catch (const NetworkError& e) {
        log_error(e.what(), e.status_code);
    } catch (const AppError& e) {
        log_error(e.what());
    }
    // E.17: Don't catch everything here -- let unexpected errors propagate
}
```

### アンチパターン

- `int`や文字列リテラルなどの組み込み型をスローする (E.14)
- 値でキャッチする（スライシングのリスク） (E.15)
- エラーを黙って飲み込む空のキャッチブロック
- フロー制御に例外を使用する (E.3)
- `errno`のようなグローバルステートに基づくエラーハンドリング (E.28)

## 定数とイミュータビリティ (Con.*)

### すべてのルール

| ルール | 要約 |
|------|---------|
| **Con.1** | デフォルトでオブジェクトをイミュータブルにする |
| **Con.2** | デフォルトでメンバー関数を`const`にする |
| **Con.3** | デフォルトで`const`へのポインタと参照を渡す |
| **Con.4** | 構築後に変更されない値には`const`を使用 |
| **Con.5** | コンパイル時に計算可能な値には`constexpr`を使用 |

```cpp
// Con.1 through Con.5: Immutability by default
class Sensor {
public:
    explicit Sensor(std::string id) : id_(std::move(id)) {}

    // Con.2: const member functions by default
    const std::string& id() const { return id_; }
    double last_reading() const { return reading_; }

    // Only non-const when mutation is required
    void record(double value) { reading_ = value; }

private:
    const std::string id_;  // Con.4: never changes after construction
    double reading_{0.0};
};

// Con.3: Pass by const reference
void display(const Sensor& s) {
    std::cout << s.id() << ": " << s.last_reading() << '\n';
}

// Con.5: Compile-time constants
constexpr double PI = 3.14159265358979;
constexpr int MAX_SENSORS = 256;
```

## 並行処理と並列処理 (CP.*)

### 主要ルール

| ルール | 要約 |
|------|---------|
| **CP.2** | データ競合を避ける |
| **CP.3** | 書き込み可能データの明示的な共有を最小化する |
| **CP.4** | スレッドではなくタスクの観点で考える |
| **CP.8** | 同期に`volatile`を使用しない |
| **CP.20** | RAIIを使用し、生の`lock()`/`unlock()`を使用しない |
| **CP.21** | 複数のミューテックスの取得には`std::scoped_lock`を使用 |
| **CP.22** | ロック保持中に未知のコードを呼ばない |
| **CP.42** | 条件なしで待機しない |
| **CP.44** | `lock_guard`と`unique_lock`に名前を付けることを忘れない |
| **CP.100** | 絶対に必要でない限りロックフリープログラミングを使用しない |

### 安全なロック

```cpp
// CP.20 + CP.44: RAII locks, always named
class ThreadSafeQueue {
public:
    void push(int value) {
        std::lock_guard<std::mutex> lock(mutex_);  // CP.44: named!
        queue_.push(value);
        cv_.notify_one();
    }

    int pop() {
        std::unique_lock<std::mutex> lock(mutex_);
        // CP.42: Always wait with a condition
        cv_.wait(lock, [this] { return !queue_.empty(); });
        const int value = queue_.front();
        queue_.pop();
        return value;
    }

private:
    std::mutex mutex_;             // CP.50: mutex with its data
    std::condition_variable cv_;
    std::queue<int> queue_;
};
```

### 複数ミューテックス

```cpp
// CP.21: std::scoped_lock for multiple mutexes (deadlock-free)
void transfer(Account& from, Account& to, double amount) {
    std::scoped_lock lock(from.mutex_, to.mutex_);
    from.balance_ -= amount;
    to.balance_ += amount;
}
```

### アンチパターン

- 同期に`volatile`を使用する (CP.8 -- ハードウェアI/O専用)
- スレッドをデタッチする (CP.26 -- ライフタイム管理がほぼ不可能になる)
- 名前なしロックガード: `std::lock_guard<std::mutex>(m);`は即座に破棄される (CP.44)
- コールバック呼び出し中にロックを保持する (CP.22 -- デッドロックのリスク)
- 深い専門知識なしのロックフリープログラミング (CP.100)

## テンプレートとジェネリックプログラミング (T.*)

### 主要ルール

| ルール | 要約 |
|------|---------|
| **T.1** | テンプレートを使用して抽象化のレベルを上げる |
| **T.2** | テンプレートを使用して多くの引数型に対するアルゴリズムを表現する |
| **T.10** | すべてのテンプレート引数にコンセプトを指定する |
| **T.11** | 可能な限り標準コンセプトを使用する |
| **T.13** | シンプルなコンセプトには省略記法を優先 |
| **T.43** | `typedef`より`using`を優先 |
| **T.120** | 本当に必要な場合にのみテンプレートメタプログラミングを使用する |
| **T.144** | 関数テンプレートを特殊化しない（代わりにオーバーロードする） |

### コンセプト (C++20)

```cpp
#include <concepts>

// T.10 + T.11: Constrain templates with standard concepts
template<std::integral T>
T gcd(T a, T b) {
    while (b != 0) {
        a = std::exchange(b, a % b);
    }
    return a;
}

// T.13: Shorthand concept syntax
void sort(std::ranges::random_access_range auto& range) {
    std::ranges::sort(range);
}

// Custom concept for domain-specific constraints
template<typename T>
concept Serializable = requires(const T& t) {
    { t.serialize() } -> std::convertible_to<std::string>;
};

template<Serializable T>
void save(const T& obj, const std::string& path);
```

### アンチパターン

- 可視名前空間内の制約なしテンプレート (T.47)
- オーバーロードの代わりに関数テンプレートを特殊化する (T.144)
- `constexpr`で十分な場合のテンプレートメタプログラミング (T.120)
- `using`の代わりに`typedef` (T.43)

## 標準ライブラリ (SL.*)

### 主要ルール

| ルール | 要約 |
|------|---------|
| **SL.1** | 可能な限りライブラリを使用する |
| **SL.2** | 他のライブラリより標準ライブラリを優先 |
| **SL.con.1** | C配列より`std::array`または`std::vector`を優先 |
| **SL.con.2** | デフォルトで`std::vector`を優先 |
| **SL.str.1** | 文字シーケンスを所有するには`std::string`を使用 |
| **SL.str.2** | 文字シーケンスを参照するには`std::string_view`を使用 |
| **SL.io.50** | `endl`を避ける（`'\n'`を使用 -- `endl`はフラッシュを強制する） |

```cpp
// SL.con.1 + SL.con.2: Prefer vector/array over C arrays
const std::array<int, 4> fixed_data{1, 2, 3, 4};
std::vector<std::string> dynamic_data;

// SL.str.1 + SL.str.2: string owns, string_view observes
std::string build_greeting(std::string_view name) {
    return "Hello, " + std::string(name) + "!";
}

// SL.io.50: Use '\n' not endl
std::cout << "result: " << value << '\n';
```

## 列挙型 (Enum.*)

### 主要ルール

| ルール | 要約 |
|------|---------|
| **Enum.1** | マクロより列挙型を優先 |
| **Enum.3** | 素の`enum`より`enum class`を優先 |
| **Enum.5** | 列挙子にALL_CAPSを使用しない |
| **Enum.6** | 無名列挙を避ける |

```cpp
// Enum.3 + Enum.5: Scoped enum, no ALL_CAPS
enum class Color { red, green, blue };
enum class LogLevel { debug, info, warning, error };

// BAD: plain enum leaks names, ALL_CAPS clashes with macros
enum { RED, GREEN, BLUE };           // Enum.3 + Enum.5 + Enum.6 violation
#define MAX_SIZE 100                  // Enum.1 violation -- use constexpr
```

## ソースファイルと命名 (SF.*, NL.*)

### 主要ルール

| ルール | 要約 |
|------|---------|
| **SF.1** | コードファイルには`.cpp`、インターフェースファイルには`.h`を使用 |
| **SF.7** | ヘッダーのグローバルスコープで`using namespace`を書かない |
| **SF.8** | すべての`.h`ファイルに`#include`ガードを使用 |
| **SF.11** | ヘッダーファイルは自己完結型であるべき |
| **NL.5** | 名前に型情報をエンコードしない（ハンガリアン記法なし） |
| **NL.8** | 一貫した命名スタイルを使用する |
| **NL.9** | ALL_CAPSはマクロ名にのみ使用 |
| **NL.10** | `underscore_style`の名前を優先 |

### ヘッダーガード

```cpp
// SF.8: Include guard (or #pragma once)
#ifndef PROJECT_MODULE_WIDGET_H
#define PROJECT_MODULE_WIDGET_H

// SF.11: Self-contained -- include everything this header needs
#include <string>
#include <vector>

namespace project::module {

class Widget {
public:
    explicit Widget(std::string name);
    const std::string& name() const;

private:
    std::string name_;
};

}  // namespace project::module

#endif  // PROJECT_MODULE_WIDGET_H
```

### 命名規約

```cpp
// NL.8 + NL.10: Consistent underscore_style
namespace my_project {

constexpr int max_buffer_size = 4096;  // NL.9: not ALL_CAPS (it's not a macro)

class tcp_connection {                 // underscore_style class
public:
    void send_message(std::string_view msg);
    bool is_connected() const;

private:
    std::string host_;                 // trailing underscore for members
    int port_;
};

}  // namespace my_project
```

### アンチパターン

- ヘッダーのグローバルスコープでの`using namespace std;` (SF.7)
- インクルード順序に依存するヘッダー (SF.10, SF.11)
- `strName`、`iCount`のようなハンガリアン記法 (NL.5)
- マクロ以外でのALL_CAPS (NL.9)

## パフォーマンス (Per.*)

### 主要ルール

| ルール | 要約 |
|------|---------|
| **Per.1** | 理由なく最適化しない |
| **Per.2** | 早すぎる最適化をしない |
| **Per.6** | 測定なしにパフォーマンスを主張しない |
| **Per.7** | 最適化を可能にする設計をする |
| **Per.10** | 静的型システムに頼る |
| **Per.11** | 計算を実行時からコンパイル時に移動する |
| **Per.19** | メモリに予測可能にアクセスする |

### ガイドライン

```cpp
// Per.11: Compile-time computation where possible
constexpr auto lookup_table = [] {
    std::array<int, 256> table{};
    for (int i = 0; i < 256; ++i) {
        table[i] = i * i;
    }
    return table;
}();

// Per.19: Prefer contiguous data for cache-friendliness
std::vector<Point> points;           // GOOD: contiguous
std::vector<std::unique_ptr<Point>> indirect_points; // BAD: pointer chasing
```

### アンチパターン

- プロファイリングデータなしの最適化 (Per.1, Per.6)
- 明確な抽象化よりも「賢い」低レベルコードを選ぶ (Per.4, Per.5)
- データレイアウトとキャッシュ動作を無視する (Per.19)

## クイックリファレンスチェックリスト

C++の作業を完了とマークする前に:

- [ ] 生の`new`/`delete`なし -- スマートポインタまたはRAIIを使用 (R.11)
- [ ] 宣言時にオブジェクトを初期化 (ES.20)
- [ ] 変数はデフォルトで`const`/`constexpr` (Con.1, ES.25)
- [ ] メンバー関数は可能な限り`const` (Con.2)
- [ ] 素の`enum`ではなく`enum class` (Enum.3)
- [ ] `0`/`NULL`ではなく`nullptr` (ES.47)
- [ ] ナローイング変換なし (ES.46)
- [ ] Cスタイルキャストなし (ES.48)
- [ ] 単一引数コンストラクタは`explicit` (C.46)
- [ ] Rule of ZeroまたはRule of Fiveを適用 (C.20, C.21)
- [ ] 基底クラスのデストラクタはpublic virtualまたはprotected非virtual (C.35)
- [ ] テンプレートはコンセプトで制約 (T.10)
- [ ] ヘッダーのグローバルスコープに`using namespace`なし (SF.7)
- [ ] ヘッダーにインクルードガードがあり自己完結型 (SF.8, SF.11)
- [ ] ロックはRAII（`scoped_lock`/`lock_guard`）を使用 (CP.20)
- [ ] 例外はカスタム型、値でスロー、参照でキャッチ (E.14, E.15)
- [ ] `std::endl`ではなく`'\n'` (SL.io.50)
- [ ] マジックナンバーなし (ES.45)
