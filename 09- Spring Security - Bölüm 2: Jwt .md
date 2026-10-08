# Spring Security - Bölüm 2: JWT ile Stateless Kimlik Doğrulama

## 1. Sorun: Kullanıcıyı İstekler Arasında Hatırlamak

HTTP **durumsuzdur** (stateless): Her istek bağımsızdır, sunucu bir öncekini hatırlamaz.

| Yaklaşım | Nasıl Çalışır | Sorunu |
|---|---|---|
| **HTTP Basic** | Her istekte kullanıcı adı + şifre | Şifre her istekte ağda dolaşır, her istekte BCrypt çalışır |
| **Session** | Sunucu bellekte oturum tutar, istemciye `JSESSIONID` cookie'si verir | Birden fazla sunucuda oturum paylaşılmalı (Redis), cookie mobil için doğal değil |
| **Token** | Sunucu imzalı token verir, **hiçbir şey saklamaz** | İptal edilemez (bkz. bölüm 8) |

> **Benzetme:**
> - **Session = Vestiyer.** Numara verirler, paltonu bulmak için kendi kayıtlarına bakmaları gerekir.
> - **Token = Konser bilekliği.** Görevli listeye bakmaz, sahtesi yapılamayan bilekliğin kendisi kanıttır.

**JWT** (JSON Web Token) bu bilekliğin en yaygın standardıdır.

## 2. JWT'nin Yapısı

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbGkiLCJpYXQiOjE3Mjgw....Xk3v...
└────── Header ─────┘ └───────────── Payload ─────────────┘ └ Signature ┘
```

```json
// Header: imza algoritması
{ "alg": "HS256" }

// Payload: claim'ler
{
  "sub": "ali",          // subject: token kime ait
  "iat": 1728000000,     // issued at: üretilme zamanı
  "exp": 1728003600      // expiration: geçerlilik sonu
}
```

- **Claim:** Payload'daki her alan. `sub`, `iat`, `exp` standarttır; kendi claim'lerin de eklenebilir (`"role": "ADMIN"`).
- **Signature:** Header + payload, sunucunun **gizli anahtarıyla** HMAC-SHA256'dan geçirilir. Token geri geldiğinde aynı işlem tekrarlanıp karşılaştırılır. Payload değiştirilirse imza uyuşmaz; değiştiren kişi anahtarı bilmediği için doğru imzayı üretemez.

> ⚠️ **JWT şifrelenmiş değildir.** Base64URL sadece kodlamadır, içerik herkes tarafından okunabilir (jwt.io).
> İmza token'ın **değiştirilmediğini** garanti eder, **gizli olduğunu** değil.
> Payload'a **asla** şifre, kimlik no, kart bilgisi gibi hassas veri konmaz.

## 3. Kurulum

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

- API ve implementasyon ayrı (JPA / Hibernate ilişkisi gibi).
- Eski eğitimlerdeki `Jwts.parser().setSigningKey(...)` 0.12'de kaldırıldı.

```properties
jwt.secret=${JWT_SECRET}
jwt.expiration-ms=900000
```

> ⚠️ **Gizli anahtar asla koda veya Git'e girmez.** Anahtarı bilen herkes istediği kullanıcı adına (admin dahil) token üretebilir.
> - Ortam değişkeninden okunur: `${JWT_SECRET}`
> - HS256 için en az **256 bit** (32 bayt), Base64 kodlu rastgele değer.
> - Üretmek için: `openssl rand -base64 32`

## 4. JwtService

```java
@Service
public class JwtService {
    private final SecretKey key;
    private final long expirationMs;

    public JwtService(@Value("${jwt.secret}") String secret,
                      @Value("${jwt.expiration-ms}") long expirationMs) {
        this.key = Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret));
        this.expirationMs = expirationMs;
    }

    public String generateToken(UserDetails user) {
        Date now = new Date();
        return Jwts.builder()
                .subject(user.getUsername())
                .issuedAt(now)
                .expiration(new Date(now.getTime() + expirationMs))
                .signWith(key)
                .compact();
    }

    public String extractUsername(String token) {
        return parseClaims(token).getSubject();
    }

    private Claims parseClaims(String token) {
        return Jwts.parser()
                .verifyWith(key)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

- `@Value` ile yapılandırma değerleri constructor'a enjekte edilir.
- Anahtar `final` field'da tutulur: Değişmeyen ayar olduğu için singleton'da güvenlidir.
- `parseSignedClaims` tek seferde **imzayı doğrular, süreyi kontrol eder, payload'ı döner.**

| Durum | Exception |
|---|---|
| İmza uyuşmuyor | `SignatureException` |
| Süre dolmuş | `ExpiredJwtException` |
| Token bozuk | `MalformedJwtException` |

Hepsi **`JwtException`**'dan türer, tek tipte yakalanabilir.

## 5. Giriş Endpoint'i

```java
public record LoginRequest(@NotBlank String username, @NotBlank String password) {}
public record LoginResponse(String token, String tokenType, long expiresIn) {}
```

Şifre doğrulaması elle yazılmaz. Hazır akış (`AuthenticationManager` → `DaoAuthenticationProvider` → `UserDetailsService` + `PasswordEncoder`) tetiklenir:

```java
// SecurityConfig
@Bean
public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
    return config.getAuthenticationManager();
}
```

```java
@Service
public class AuthService {
    private final AuthenticationManager authenticationManager;
    private final JwtService jwtService;

    public LoginResponse login(LoginRequest request) {
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(request.username(), request.password()));

        UserDetails user = (UserDetails) authentication.getPrincipal();
        return new LoginResponse(jwtService.generateToken(user), "Bearer", 900);
    }
}
```

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @PostMapping("/register")
    public ResponseEntity<Void> register(@Valid @RequestBody RegisterRequest request) {
        authService.register(request);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }

    @PostMapping("/login")
    public LoginResponse login(@Valid @RequestBody LoginRequest request) {
        return authService.login(request);
    }
}
```

### Hatalı Giriş

Yanlış şifrede `BadCredentialsException` fırlatılır. Bu hata **controller'ın içinden** geldiği için `@RestControllerAdvice`'a **ulaşır**:

```java
@ExceptionHandler(BadCredentialsException.class)
public ResponseEntity<ErrorResponse> handleBadCredentials(BadCredentialsException ex) {
    return build(HttpStatus.UNAUTHORIZED, "Kullanıcı adı veya şifre hatalı");
}
```

> Yazılmazsa `Exception.class` handler'ı yakalar ve **500** döner.
> **Kural:** Hatanın **nereden** fırlatıldığı, **nerede** yakalanacağını belirler.
> - Filtrede → `AuthenticationEntryPoint` / `AccessDeniedHandler`
> - Controller / service'te → `@RestControllerAdvice`

## 6. JWT Filtresi

İstemci sonraki isteklerde token'ı gönderir:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIi...
```

> **Bearer** = "Taşıyana erişim ver." Token'ı elinde tutan yetkilidir. Token çalınırsa çalan da yetkili olur.

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        String header = request.getHeader("Authorization");

        // 1. Token yoksa hiçbir şey yapma, devam et
        if (header == null || !header.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String token = header.substring(7);

        try {
            // 2. Token'ı doğrula, kullanıcı adını çıkar
            String username = jwtService.extractUsername(token);

            // 3. Kullanıcıyı yükle, SecurityContext'e koy
            if (SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails user = userDetailsService.loadUserByUsername(username);
                UsernamePasswordAuthenticationToken authentication =
                        new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (JwtException | UsernameNotFoundException ex) {
            // Geçersiz token: kimlik doğrulanmadan devam et
        }

        // 4. Her durumda zincire devam et
        filterChain.doFilter(request, response);
    }
}
```

| Adım | Açıklama |
|---|---|
| **`OncePerRequestFilter`** | Filtrenin bir istekte yalnızca bir kez çalışmasını garanti eder |
| **1. Token yoksa** | Hata verilmez. İstek herkese açık bir endpoint'e gidiyor olabilir. Korumalı endpoint'lerde ilerideki **authorization filtresi** boş context'i görür ve 401 döner. |
| **2-3. Token geçerliyse** | Kullanıcı yüklenir, `SecurityContextHolder`'a konur. Artık `@AuthenticationPrincipal` ile erişilebilir. |
| **Şifre `null`** | Kimlik token ile doğrulandı, şifreyi bellekte tutmaya gerek yok |
| **Geçersiz token** | Exception yakalanır, kimlik belirlenmeden devam edilir. Yakalanmazsa 500 döner (filtre hataları `@RestControllerAdvice`'a ulaşmaz). |
| **4. `doFilter`** | Unutulursa istek takılır, hiçbir cevap dönmez |

> **Görev ayrımı:** Filtre sadece **"kim?"** sorusunu cevaplar. **"Erişebilir mi?"** kararı authorization filtresinindir.

### Tuzak: Filtrenin İki Kez Kaydedilmesi

Spring Boot, `Filter` tipindeki her bean'i **genel servlet zincirine de** otomatik kaydeder. Filtre hem security zincirinde hem dışında yer alır.

```java
@Bean
public FilterRegistrationBean<JwtAuthenticationFilter> jwtFilterRegistration(JwtAuthenticationFilter filter) {
    FilterRegistrationBean<JwtAuthenticationFilter> registration = new FilterRegistrationBean<>(filter);
    registration.setEnabled(false); // servlet zincirine kaydetme
    return registration;
}
```

## 7. SecurityConfig

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http,
                                               JwtAuthenticationFilter jwtFilter) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
        .exceptionHandling(ex -> ex
                .authenticationEntryPoint(jsonAuthenticationEntryPoint)
                .accessDeniedHandler(jsonAccessDeniedHandler))
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);

    return http.build();
}
```

| Değişiklik | Sebep |
|---|---|
| `httpBasic` kaldırıldı | Kimlik artık token ile geliyor |
| `SessionCreationPolicy.STATELESS` | Oturum oluşturulmaz, `JSESSIONID` gönderilmez. Yazılmazsa kimlik cookie ile taşınabilir ve CSRF'in kapalı olması açığa dönüşür. **CSRF kapatma ile birlikte gider.** |
| `addFilterBefore(...)` | Filtre, yetki kontrolünden **önce** çalışmalı. Sonra çalışırsa kullanıcı tanınmadan yetki kontrol edilir, her istek 401 alır. |

### Uçtan Uca Akış

```
1. POST /api/auth/register   {username, password}         → 201 Created
2. POST /api/auth/login      {username, password}         → {token: "eyJ..."}
3. GET  /api/orders          Authorization: Bearer eyJ... → 200 OK
```

3. istekte: JWT filtresi token'ı doğrular, kullanıcıyı context'e koyar → Authorization filtresi `authenticated()` kuralını kontrol eder → Controller çalışır.

## 8. JWT'nin Bedeli: İptal Edilemeyen Token

Sunucu hiçbir şey saklamadığı için token'ı "iptal edildi" olarak işaretleyecek bir yer yoktur. **Token süresi dolana kadar geçerlidir.**

- Şifre değiştirildi → eski token'lar çalışmaya devam eder.
- Kullanıcı engellendi → token'ı çalışmaya devam eder.
- "Çıkış yap" → istemci token'ı siler, ama kopyası başka yerdeyse hâlâ geçerlidir.

> "Sunucu hiçbir şey hatırlamaz" demek, "sunucu hiçbir şeyi unutamaz" demektir.

### Çözüm: Access Token + Refresh Token

| Token | Ömür | Saklandığı Yer | Görevi |
|---|---|---|---|
| **Access token** (JWT) | Kısa (5-15 dk) | Sadece istemci | Her istekte gönderilir |
| **Refresh token** | Uzun (gün / hafta) | İstemci **ve veritabanı** | Sadece yeni access token almak için |

- Access token süresi dolunca istemci refresh token'ı `/api/auth/refresh`'e gönderip yenisini alır.
- Refresh token veritabanında olduğu için **iptal edilebilir** (çıkış, şifre değişikliği, engelleme).
- Saldırgan en fazla access token'ın kalan süresi kadar erişebilir.

### Roller: Token'dan mı, Veritabanından mı?

| | Rolleri token'dan okumak | Her istekte veritabanından yüklemek |
|---|---|---|
| **Performans** | Sorgu yok | Her istekte 1 sorgu |
| **Rol değişikliği** | Token süresi dolana kadar eski yetkiler geçerli | Anında etkili |
| **Silinen hesap** | Token süresi dolana kadar çalışır | Anında reddedilir |

Başlangıç için veritabanından yüklemek daha güvenli ve anlaşılırdır.

### İstemci Token'ı Nerede Saklamalı?

| Yer | Risk |
|---|---|
| `localStorage` | Sayfadaki her JavaScript erişebilir. **XSS** açığında token çalınır. |
| `httpOnly` cookie | JavaScript erişemez, XSS'e karşı korur. Ama tarayıcı otomatik gönderdiği için **CSRF** riski geri gelir, CSRF koruması açılmalı. |
| Mobil | iOS Keychain, Android Keystore |

Mükemmel seçenek yok, sadece farklı riskler var.

### Alternatif: Spring'in Kendi JWT Desteği

`spring-boot-starter-oauth2-resource-server` modülünde JWT doğrulama hazırdır:

```java
http.oauth2ResourceServer(o -> o.jwt(...));
```

Filtreyi, imza doğrulamayı ve hata yönetimini Spring yapar. Keycloak, Auth0 gibi ayrı bir kimlik sunucusu kullanan büyük sistemlerde standart yol budur. Elle yazılan filtre, bu modülün arka planda ne yaptığını anlamayı sağlar.

---
