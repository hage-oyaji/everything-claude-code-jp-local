---
name: springboot-verification
description: "Spring Bootプロジェクトの検証ループ：ビルド、静的解析、カバレッジ付きテスト、セキュリティスキャン、リリースまたはPR前の差分レビュー。"
origin: ECC
---

# Spring Boot検証ループ

PR前、大きな変更後、デプロイ前に実行します。

## 発動タイミング

- Spring BootサービスのPRを作成する前
- 大規模なリファクタリングや依存関係アップグレード後
- ステージングまたはプロダクションのデプロイ前検証
- ビルド→lint→テスト→セキュリティスキャンのフルパイプライン実行
- テストカバレッジが閾値を満たしているか検証

## フェーズ 1: ビルド

```bash
mvn -T 4 clean verify -DskipTests
# or
./gradlew clean assemble -x test
```

ビルドが失敗したら、停止して修正します。

## フェーズ 2: 静的解析

Maven（一般的なプラグイン）：
```bash
mvn -T 4 spotbugs:check pmd:check checkstyle:check
```

Gradle（設定されている場合）：
```bash
./gradlew checkstyleMain pmdMain spotbugsMain
```

## フェーズ 3: テスト + カバレッジ

```bash
mvn -T 4 test
mvn jacoco:report   # verify 80%+ coverage
# or
./gradlew test jacocoTestReport
```

レポート：
- 総テスト数、成功/失敗
- カバレッジ %（行/ブランチ）

### ユニットテスト

モックされた依存関係でサービスロジックを分離してテスト：

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

  @Mock private UserRepository userRepository;
  @InjectMocks private UserService userService;

  @Test
  void createUser_validInput_returnsUser() {
    var dto = new CreateUserDto("Alice", "alice@example.com");
    var expected = new User(1L, "Alice", "alice@example.com");
    when(userRepository.save(any(User.class))).thenReturn(expected);

    var result = userService.create(dto);

    assertThat(result.name()).isEqualTo("Alice");
    verify(userRepository).save(any(User.class));
  }

  @Test
  void createUser_duplicateEmail_throwsException() {
    var dto = new CreateUserDto("Alice", "existing@example.com");
    when(userRepository.existsByEmail(dto.email())).thenReturn(true);

    assertThatThrownBy(() -> userService.create(dto))
        .isInstanceOf(DuplicateEmailException.class);
  }
}
```

### Testcontainersによる統合テスト

H2ではなく実際のデータベースに対してテスト：

```java
@SpringBootTest
@Testcontainers
class UserRepositoryIntegrationTest {

  @Container
  static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
      .withDatabaseName("testdb");

  @DynamicPropertySource
  static void configureProperties(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", postgres::getJdbcUrl);
    registry.add("spring.datasource.username", postgres::getUsername);
    registry.add("spring.datasource.password", postgres::getPassword);
  }

  @Autowired private UserRepository userRepository;

  @Test
  void findByEmail_existingUser_returnsUser() {
    userRepository.save(new User("Alice", "alice@example.com"));

    var found = userRepository.findByEmail("alice@example.com");

    assertThat(found).isPresent();
    assertThat(found.get().getName()).isEqualTo("Alice");
  }
}
```

### MockMvcによるAPIテスト

フルSpringコンテキストでコントローラーレイヤーをテスト：

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

  @Autowired private MockMvc mockMvc;
  @MockBean private UserService userService;

  @Test
  void createUser_validInput_returns201() throws Exception {
    var user = new UserDto(1L, "Alice", "alice@example.com");
    when(userService.create(any())).thenReturn(user);

    mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"name": "Alice", "email": "alice@example.com"}
                """))
        .andExpect(status().isCreated())
        .andExpect(jsonPath("$.name").value("Alice"));
  }

  @Test
  void createUser_invalidEmail_returns400() throws Exception {
    mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"name": "Alice", "email": "not-an-email"}
                """))
        .andExpect(status().isBadRequest());
  }
}
```

## フェーズ 4: セキュリティスキャン

```bash
# Dependency CVEs
mvn org.owasp:dependency-check-maven:check
# or
./gradlew dependencyCheckAnalyze

# Secrets in source
grep -rn "password\s*=\s*\"" src/ --include="*.java" --include="*.yml" --include="*.properties"
grep -rn "sk-\|api_key\|secret" src/ --include="*.java" --include="*.yml"

# Secrets (git history)
git secrets --scan  # if configured
```

### 一般的なセキュリティ発見事項

```
# Check for System.out.println (use logger instead)
grep -rn "System\.out\.print" src/main/ --include="*.java"

# Check for raw exception messages in responses
grep -rn "e\.getMessage()" src/main/ --include="*.java"

# Check for wildcard CORS
grep -rn "allowedOrigins.*\*" src/main/ --include="*.java"
```

## フェーズ 5: Lint/フォーマット（オプションゲート）

```bash
mvn spotless:apply   # if using Spotless plugin
./gradlew spotlessApply
```

## フェーズ 6: 差分レビュー

```bash
git diff --stat
git diff
```

チェックリスト：
- デバッグログが残っていない（`System.out`、ガードなしの`log.debug`）
- 意味のあるエラーとHTTPステータス
- 必要な箇所にトランザクションとバリデーションがある
- 設定変更が文書化されている

## 出力テンプレート

```
VERIFICATION REPORT
===================
Build:     [PASS/FAIL]
Static:    [PASS/FAIL] (spotbugs/pmd/checkstyle)
Tests:     [PASS/FAIL] (X/Y passed, Z% coverage)
Security:  [PASS/FAIL] (CVE findings: N)
Diff:      [X files changed]

Overall:   [READY / NOT READY]

Issues to Fix:
1. ...
2. ...
```

## 継続モード

- 重要な変更時または長いセッションで30〜60分ごとにフェーズを再実行
- 短いループを維持: `mvn -T 4 test` + spotbugsでクイックフィードバック

**覚えておくべきこと**: 早いフィードバックは遅い驚きに勝ります。ゲートを厳しく保ちましょう — プロダクションシステムでは警告を欠陥として扱います。
