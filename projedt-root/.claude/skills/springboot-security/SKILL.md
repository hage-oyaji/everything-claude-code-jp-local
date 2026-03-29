---
name: springboot-security
description: Java Spring Bootサービスにおける認証/認可、バリデーション、CSRF、シークレット、ヘッダー、レート制限、依存関係セキュリティのSpring Securityベストプラクティス。
origin: ECC
---

# Spring Bootセキュリティレビュー

認証の追加、入力の処理、エンドポイントの作成、シークレットの取り扱い時に使用します。

## 発動タイミング

- 認証の追加時（JWT、OAuth2、セッションベース）
- 認可の実装時（@PreAuthorize、ロールベースアクセス）
- ユーザー入力のバリデーション時（Bean Validation、カスタムバリデータ）
- CORS、CSRF、セキュリティヘッダーの設定時
- シークレットの管理時（Vault、環境変数）
- レート制限やブルートフォース保護の追加時
- 依存関係のCVEスキャン時

## 認証

- ステートレスJWTまたは失効リスト付き不透明トークンを優先
- セッションには`httpOnly`、`Secure`、`SameSite=Strict`Cookieを使用
- `OncePerRequestFilter`またはリソースサーバーでトークンを検証

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
  private final JwtService jwtService;

  public JwtAuthFilter(JwtService jwtService) {
    this.jwtService = jwtService;
  }

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain chain) throws ServletException, IOException {
    String header = request.getHeader(HttpHeaders.AUTHORIZATION);
    if (header != null && header.startsWith("Bearer ")) {
      String token = header.substring(7);
      Authentication auth = jwtService.authenticate(token);
      SecurityContextHolder.getContext().setAuthentication(auth);
    }
    chain.doFilter(request, response);
  }
}
```

## 認可

- メソッドセキュリティを有効化: `@EnableMethodSecurity`
- `@PreAuthorize("hasRole('ADMIN')")`または`@PreAuthorize("@authz.canEdit(#id)")`を使用
- デフォルトで拒否；必要なスコープのみ公開

```java
@RestController
@RequestMapping("/api/admin")
public class AdminController {

  @PreAuthorize("hasRole('ADMIN')")
  @GetMapping("/users")
  public List<UserDto> listUsers() {
    return userService.findAll();
  }

  @PreAuthorize("@authz.isOwner(#id, authentication)")
  @DeleteMapping("/users/{id}")
  public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.noContent().build();
  }
}
```

## 入力バリデーション

- コントローラーで`@Valid`を使用したBean Validation
- DTOに制約を適用: `@NotBlank`、`@Email`、`@Size`、カスタムバリデータ
- レンダリング前にHTMLをホワイトリストでサニタイズ

```java
// BAD: No validation
@PostMapping("/users")
public User createUser(@RequestBody UserDto dto) {
  return userService.create(dto);
}

// GOOD: Validated DTO
public record CreateUserDto(
    @NotBlank @Size(max = 100) String name,
    @NotBlank @Email String email,
    @NotNull @Min(0) @Max(150) Integer age
) {}

@PostMapping("/users")
public ResponseEntity<UserDto> createUser(@Valid @RequestBody CreateUserDto dto) {
  return ResponseEntity.status(HttpStatus.CREATED)
      .body(userService.create(dto));
}
```

## SQLインジェクション防止

- Spring Dataリポジトリまたはパラメータ化クエリを使用
- ネイティブクエリでは`:param`バインディングを使用；文字列連結は絶対にしない

```java
// BAD: String concatenation in native query
@Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)

// GOOD: Parameterized native query
@Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)
List<User> findByName(@Param("name") String name);

// GOOD: Spring Data derived query (auto-parameterized)
List<User> findByEmailAndActiveTrue(String email);
```

## パスワードエンコーディング

- パスワードは常にBCryptまたはArgon2でハッシュ — 平文で保存しない
- 手動ハッシュではなく`PasswordEncoder`Beanを使用

```java
@Bean
public PasswordEncoder passwordEncoder() {
  return new BCryptPasswordEncoder(12); // cost factor 12
}

// In service
public User register(CreateUserDto dto) {
  String hashedPassword = passwordEncoder.encode(dto.password());
  return userRepository.save(new User(dto.email(), hashedPassword));
}
```

## CSRF保護

- ブラウザセッションアプリではCSRFを有効に保つ；フォーム/ヘッダーにトークンを含める
- Bearerトークンを使用する純粋なAPIでは、CSRFを無効にしてステートレス認証に依存

```java
http
  .csrf(csrf -> csrf.disable())
  .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

## シークレット管理

- ソースコードにシークレットを入れない；環境変数またはVaultから読み込む
- `application.yml`を認証情報フリーに保つ；プレースホルダーを使用
- トークンとDB認証情報を定期的にローテーション

```yaml
# BAD: Hardcoded in application.yml
spring:
  datasource:
    password: mySecretPassword123

# GOOD: Environment variable placeholder
spring:
  datasource:
    password: ${DB_PASSWORD}

# GOOD: Spring Cloud Vault integration
spring:
  cloud:
    vault:
      uri: https://vault.example.com
      token: ${VAULT_TOKEN}
```

## セキュリティヘッダー

```java
http
  .headers(headers -> headers
    .contentSecurityPolicy(csp -> csp
      .policyDirectives("default-src 'self'"))
    .frameOptions(HeadersConfigurer.FrameOptionsConfig::sameOrigin)
    .xssProtection(Customizer.withDefaults())
    .referrerPolicy(rp -> rp.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.NO_REFERRER)));
```

## CORS設定

- CORSはコントローラーごとではなく、セキュリティフィルターレベルで設定
- 許可オリジンを制限 — プロダクションでは`*`を使用しない

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
  CorsConfiguration config = new CorsConfiguration();
  config.setAllowedOrigins(List.of("https://app.example.com"));
  config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
  config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
  config.setAllowCredentials(true);
  config.setMaxAge(3600L);

  UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
  source.registerCorsConfiguration("/api/**", config);
  return source;
}

// In SecurityFilterChain:
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

## レート制限

- Bucket4jまたはゲートウェイレベルの制限をコストの高いエンドポイントに適用
- バースト時にログとアラート；リトライヒント付きで429を返す

```java
// Using Bucket4j for per-endpoint rate limiting
@Component
public class RateLimitFilter extends OncePerRequestFilter {
  private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

  private Bucket createBucket() {
    return Bucket.builder()
        .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1))))
        .build();
  }

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain chain) throws ServletException, IOException {
    String clientIp = request.getRemoteAddr();
    Bucket bucket = buckets.computeIfAbsent(clientIp, k -> createBucket());

    if (bucket.tryConsume(1)) {
      chain.doFilter(request, response);
    } else {
      response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
      response.getWriter().write("{\"error\": \"Rate limit exceeded\"}");
    }
  }
}
```

## 依存関係のセキュリティ

- CIでOWASP Dependency Check / Snykを実行
- Spring BootとSpring Securityをサポート対象バージョンに保つ
- 既知のCVEでビルドを失敗させる

## ログとPII

- シークレット、トークン、パスワード、完全なPANデータをログに記録しない
- センシティブフィールドをリダクト；構造化JSONログを使用

## ファイルアップロード

- サイズ、コンテンツタイプ、拡張子をバリデーション
- Webルート外に保存；必要に応じてスキャン

## リリース前チェックリスト

- [ ] 認証トークンが正しくバリデーションされ期限切れになる
- [ ] すべてのセンシティブパスに認可ガードがある
- [ ] すべての入力がバリデーションされサニタイズされている
- [ ] 文字列連結のSQLがない
- [ ] CSRFの態勢がアプリタイプに適切
- [ ] シークレットが外部化されている；コミットされたものがない
- [ ] セキュリティヘッダーが設定されている
- [ ] APIにレート制限がある
- [ ] 依存関係がスキャンされ最新
- [ ] ログにセンシティブデータがない

**覚えておくべきこと**: デフォルトで拒否、入力をバリデーション、最小権限、そして設定によるセキュアファーストを優先してください。
