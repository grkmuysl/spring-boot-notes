# Spring Security - Bölüm 1: Temeller

## 1. Authentication ve Authorization

| Kavram | Soru | Benzetme |
|---|---|---|
| **Authentication** (kimlik doğrulama) | "Sen kimsin?" | Pasaport kontrolü |
| **Authorization** (yetkilendirme) | "Buna iznin var mı?" | Business class salonuna girebilir misin? |

Sıra her zaman aynıdır: **önce kimlik, sonra yetki.**

| Durum | Kod | Anlamı |
|---|---|---|
| Kimlik doğrulanamadı | **401 Unauthorized** | "Kim olduğunu bilmiyorum" |
| Kimlik var, yetki yok | **403 Forbidden** | "Kim olduğunu biliyorum ama iznin yok" |

> 401'in adı "Unauthorized" ama anlamı **unauthenticated**. Tarihsel bir isimlendirme hatası.

## 2. Bağımlılık Eklenince Ne Olur?

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

- **Tüm endpoint'ler kilitlenir** (401).
- Konsola rastgele bir şifre yazılır: `Using generated security password: ...`
- Kullanıcı adı `user` olan tek bir kullanıcı oluşturulur.

> **Felsefe:** Varsayılan olarak her şey kapalı. Neyin açık olacağına açıkça karar verilir. Korumayı unutmak güvenlik açığı değil, erişim hatası üretir.

## 3. Mimari: Filter Chain

```
HTTP İsteği
    │
    ▼
┌──────────────────── Security Filter Chain ────────────────────┐
│ [CORS] → [CSRF] → [Authentication] → ... → [Authorization]    │
└───────────────────────────────────────────────────────────────┘
    │
    ▼
DispatcherServlet → Controller → Service → Repository
```

- **Filter:** İstek controller'a ulaşmadan önce araya giren bileşen. Her filtrenin tek görevi vardır; işini yapıp isteği iletir veya durdurur.
- Proxy fikrinin HTTP seviyesindeki karşılığı.

> ⚠️ **Filtreler controller'dan önce çalışır.** Filtrede fırlatılan hata `@RestControllerAdvice`'a **ulaşmaz.**

### Kimlik Doğrulama Akışı

```
1. Authentication Filter    İstekten kimlik bilgisini çıkarır
         │
2. AuthenticationManager    İşi uygun provider'a devreder
         │
3. AuthenticationProvider   Asıl doğrulamayı yapar (DaoAuthenticationProvider)
         ├──► UserDetailsService   Kullanıcıyı getirir
         └──► PasswordEncoder      Şifreyi karşılaştırır
         │
4. SecurityContextHolder    Doğrulanan kullanıcı saklanır
         │
5. Authorization Filter     Yetki kontrolü yapılır
         │
      Controller
```

**Genelde yazılması gerekenler:**
- `UserDetailsService`: Kullanıcılar nereden yüklenecek?
- `PasswordEncoder`: Şifreler nasıl karşılaştırılacak?
- `SecurityFilterChain`: Hangi endpoint'e kim erişebilir?

### SecurityContextHolder ve ThreadLocal

- Giriş yapan kullanıcı `SecurityContextHolder`'da tutulur.
- Varsayılan olarak **`ThreadLocal`** kullanır: Her thread kendi kopyasını görür.
- Her istek ayrı thread'de çalıştığı için eşzamanlı isteklerde kullanıcılar karışmaz.
- İstek bitince temizlenir.

## 4. SecurityFilterChain

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

> **Eski yöntem:** `WebSecurityConfigurerAdapter` Spring Security 5.7'de kullanımdan kaldırıldı, 6'da silindi. Zincirleme yazım (`.csrf().disable().and()...`) yerine lambda kullanılır.

| Kural | Anlamı |
|---|---|
| `permitAll()` | Herkes erişebilir |
| `authenticated()` | Giriş yapmış herkes |
| `hasRole("ADMIN")` | Sadece ADMIN rolü |
| `hasAnyRole("ADMIN", "MANAGER")` | Bu rollerden biri |
| `denyAll()` | Kimse erişemez |

- `requestMatchers(HttpMethod.GET, ...)` sadece belirtilen HTTP metoduna uygulanır.
- `/**` yolun altındaki her şey demektir.

### Sıra Önemlidir

Kurallar **yukarıdan aşağıya** kontrol edilir, **ilk eşleşen** uygulanır.

```java
.requestMatchers("/api/**").authenticated()
.requestMatchers("/api/admin/**").hasRole("ADMIN")   // ASLA ULAŞILMAZ
```

- Spesifik kurallar üstte, genel kurallar altta.
- `anyRequest()` her zaman en sonda.

### HTTP Basic

- İstemci her istekte `Authorization: Basic <base64(kullanıcı:şifre)>` gönderir.
- **Base64 şifreleme değildir**, herkes çözebilir. Sadece HTTPS ile kullanılır.
- Her istekte şifre gönderildiği için ideal değildir. JWT'ye geçilmesinin sebebi bu.

### CSRF

**Saldırı:** Kullanıcı bankaya giriş yapmış (cookie var). Kötü niyetli site arka planda bankaya istek atar, tarayıcı cookie'yi **otomatik** ekler, banka isteği meşru sanır.

| Durum | CSRF |
|---|---|
| Stateless REST API, token elle ekleniyor (JWT) | Kapatılabilir |
| Kimlik doğrulama cookie ile (oturum, cookie'de token) | **Kapatılmamalı** |

> "Herkes kapatıyor" diye körü körüne kapatılmaz. Neden kapatıldığı bilinmeli.

## 5. PasswordEncoder

| | Yön | Şifre için |
|---|---|---|
| **Şifreleme** (encryption) | İki yönlü, anahtarla geri çözülür | ❌ Anahtar ele geçerse hepsi açığa çıkar |
| **Hashleme** (hashing) | Tek yönlü, geri çözülemez | ✅ |

Doğrulama: Girilen şifre de hashlenir ve kayıtlı hash ile karşılaştırılır.

```java
String hash = passwordEncoder.encode("gizli123");
passwordEncoder.matches("gizli123", hash);   // true
```

### BCrypt'in Özellikleri

- **Salt:** Her hash'e rastgele değer eklenir ve hash'in içinde saklanır. Aynı şifre her seferinde **farklı hash** üretir. Rainbow table saldırılarını önler.
- Bu yüzden karşılaştırma `equals` ile değil **`matches`** ile yapılır.
- **Kasıtlı olarak yavaştır:** `$2a$10$...` içindeki `10` maliyet faktörü (2¹⁰ tur). Kaba kuvvet saldırılarını pratik olarak imkansız kılar.

> MD5, SHA-256 gibi hızlı hash'ler şifre için uygun değildir, çünkü hızlıdırlar.

## 6. UserDetailsService

Spring Security kendi `User` entity'ni tanımaz, `UserDetails` arayüzünü anlar.

```java
@Entity
@Table(name = "users") // "user" SQL'de ayrılmış kelime
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @Column(nullable = false)
    private String password; // BCrypt hash'i

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Role role;
}

public enum Role { USER, ADMIN }
```

> **`@Enumerated(EnumType.STRING)`:** Yazılmazsa varsayılan `ORDINAL` sırayı (0, 1) kaydeder. Enum'a yeni değer eklenince eski kayıtların anlamı kayar. `STRING` her zaman daha güvenli.

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("Kullanıcı bulunamadı"));

        return org.springframework.security.core.userdetails.User
                .withUsername(user.getUsername())
                .password(user.getPassword())
                .roles(user.getRole().name())
                .build();
    }
}
```

- Context'te tek bir `UserDetailsService` ve bir `PasswordEncoder` bean'i varsa Spring Security bunları **otomatik bağlar.** Başka kod gerekmez (DI).
- **User enumeration:** Kullanıcı bulunamadığında da şifre yanlış olduğunda da aynı genel hata dönülmeli. Spring Security `UsernameNotFoundException`'ı `BadCredentialsException`'a çevirir.

### Kullanıcı Kaydı

```java
@Transactional
public void register(RegisterRequest request) {
    if (userRepository.existsByUsername(request.username())) {
        throw new BusinessException("Bu kullanıcı adı zaten alınmış");
    }
    User user = new User(request.username(),
                         passwordEncoder.encode(request.password()), // kaydetmeden ÖNCE hashle
                         Role.USER);                                  // rol SUNUCUDA belirlenir
    userRepository.save(user);
}
```

> ⚠️ `RegisterRequest`'te **`role` alanı olmamalı.** Aksi halde herkes kendini ADMIN olarak kaydedebilir.

## 7. Roller ve Yetkiler

Yetkiler (`GrantedAuthority`) basit string'lerdir. **Rol = `ROLE_` önekiyle başlayan yetki.**

| Kod | Gerçekte |
|---|---|
| `.roles("ADMIN")` | `ROLE_ADMIN` ekler (öneki kendisi koyar) |
| `.authorities("ROLE_ADMIN")` | `ROLE_ADMIN` ekler (öneki sen yazarsın) |
| `hasRole("ADMIN")` | `ROLE_ADMIN` arar (öneki kendisi ekler) |
| `hasAuthority("ROLE_ADMIN")` | Birebir `ROLE_ADMIN` arar |

```java
.authorities("ADMIN")                    // yetki: "ADMIN"
.requestMatchers(...).hasRole("ADMIN")   // aranan: "ROLE_ADMIN" → 403!
```

> **Kural:** `roles()` ile verdiysen `hasRole()`, `authorities()` ile verdiysen `hasAuthority()` ile kontrol et.

- **Rol:** Kaba taneli grup (USER, ADMIN)
- **Yetki:** İnce taneli izin (`product:create`, `order:delete`)

## 8. Method Security

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig { ... }
```

```java
@PreAuthorize("hasRole('ADMIN')")
public void delete(Long id) { ... }

@PreAuthorize("hasRole('ADMIN') or #username == authentication.name")
public UserResponse getProfile(String username) { ... }
```

- `#username`: Metot parametresi, `authentication.name`: Giriş yapan kullanıcı.
- "Kendi verisine erişebilir" gibi URL kurallarıyla yapılamayan kontroller.
- URL kuralları ve method security alternatif değil, **katmandır.** Birlikte kullanılır.

> ⚠️ **`@PreAuthorize` proxy ile çalışır.** Sınıf içinden çağrılan veya `private` metotlarda kontrol **yapılmaz**. Hata alınmadığı için sessiz bir güvenlik açığıdır.

### Giriş Yapan Kullanıcıya Erişmek

```java
@GetMapping("/me")
public UserResponse me(@AuthenticationPrincipal UserDetails currentUser) {
    return userService.findByUsername(currentUser.getUsername());
}
```

- `SecurityContextHolder.getContext().getAuthentication()` her yerden çalışır ama gizli bağımlılık yaratır ve testi zorlaştırır.
- Tercih edilen: Kullanıcıyı controller'da alıp service'e parametre olarak geçmek.

## 9. Security Hatalarını Yönetmek

`ExceptionTranslationFilter` security hatalarını yakalar ve yönlendirir:

| Hata | Yönlendirildiği Yer | Cevap |
|---|---|---|
| `AuthenticationException` | `AuthenticationEntryPoint` | 401 |
| `AccessDeniedException` | `AccessDeniedHandler` | 403 |

Tutarlı JSON formatı için:

```java
@Component
public class JsonAuthenticationEntryPoint implements AuthenticationEntryPoint {
    private final ObjectMapper objectMapper;

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException ex) throws IOException {
        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        objectMapper.writeValue(response.getOutputStream(),
                new ErrorResponse(401, "Unauthorized", "Kimlik doğrulaması gerekli", LocalDateTime.now()));
    }
}
```

```java
http.exceptionHandling(ex -> ex
        .authenticationEntryPoint(jsonAuthenticationEntryPoint)
        .accessDeniedHandler(jsonAccessDeniedHandler));
```

- Controller dünyasının dışında olunduğu için `ResponseEntity` dönülemez, cevap doğrudan `HttpServletResponse`'a yazılır.

> ⚠️ **İstisna:** `@PreAuthorize`'ın fırlattığı `AccessDeniedException` service'ten gelir, `@RestControllerAdvice`'a ulaşır. `Exception.class` handler'ı yakalarsa **403 yerine 500** döner. `AccessDeniedException` için ayrı handler yazılmalı.

---
