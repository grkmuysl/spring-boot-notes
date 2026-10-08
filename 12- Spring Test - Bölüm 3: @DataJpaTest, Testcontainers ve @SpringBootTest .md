# Test Yazmak - Bölüm 3: @DataJpaTest, Testcontainers ve @SpringBootTest

## 1. Repository Testi Neyi Test Eder?

> ❌ Spring Data'nın hazır metotları (`save`, `findById`) test edilmez. Kütüphanenin kendi kodudur.

| ✅ Test Edilir | Örnek |
|---|---|
| Türetilmiş sorgular | `findByNameStartingWithIgnoreCase` gerçekten harf duyarsız mı? |
| `@Query` (JPQL / native) | Doğru sonucu dönüyor mu? |
| Entity eşlemeleri ve kısıtlar | `unique` kısıtı veritabanında oluşuyor mu? |
| Performans davranışları | `JOIN FETCH` ilişkiyi gerçekten yüklüyor mu? |

## 2. @DataJpaTest

```java
@DataJpaTest
class ProductRepositoryTest {

    @Autowired private ProductRepository productRepository;
    @Autowired private TestEntityManager entityManager;
}
```

Sadece JPA katmanını yükleyen **dilim testi**.

| Yüklenenler | Yüklenmeyenler |
|---|---|
| `@Entity` sınıfları | Controller'lar |
| Repository arayüzleri | Service'ler |
| `DataSource`, `EntityManager`, Hibernate | Security |
| `TestEntityManager` | Diğer `@Component`'ler |

### Varsayılan Davranışlar

- **Gömülü veritabanı:** Classpath'te H2 varsa, `application.properties`'teki veritabanı yerine bellek içi H2 kullanılır.
- **Her test transaction içinde çalışır ve sonunda geri alınır (rollback).** Testler birbirinden bağımsızdır.

### TestEntityManager

Test verisini **test edilen repository ile değil**, `TestEntityManager` ile hazırla. Test edilen şeyle veri hazırlanırsa, oradaki bir hata hem hazırlığı hem doğrulamayı bozup birbirini gizleyebilir.

### Örnek

```java
@Test
void findByNameStartingWithIgnoreCase_ignoresCase() {
    Category electronics = entityManager.persist(new Category("Elektronik"));
    entityManager.persist(new Product("Laptop", new BigDecimal("25000"), electronics));
    entityManager.persist(new Product("LAMBA", new BigDecimal("500"), electronics));
    entityManager.persist(new Product("Telefon", new BigDecimal("15000"), electronics)); // dışarıda kalmalı

    List<Product> result = productRepository.findByNameStartingWithIgnoreCase("la");

    assertThat(result).extracting(Product::getName)
            .containsExactlyInAnyOrder("Laptop", "LAMBA");
}
```

> **Dışarıda kalması gereken veriyi de ekle.** Sadece eşleşen verilerle test edilirse `findAll()` dönen bozuk bir sorgu da geçer.

## 3. ⚠️ Flush ve Persistence Context Tuzağı

```java
// YANLIŞ GÜVEN VEREN TEST
@Test
void updateStock_persistsNewValue() {
    Product laptop = entityManager.persist(new Product(...));
    laptop.setStock(42);
    Product found = productRepository.findById(laptop.getId()).get();
    assertThat(found.getStock()).isEqualTo(42); // GEÇER ama veritabanına hiç UPDATE gitmedi!
}
```

**İki sorun:**
1. `findById` persistence context'teki **aynı Java nesnesini** döner, veritabanına gitmez.
2. Dirty checking değişiklikleri **commit**'te yazar. Test sonunda **rollback** olduğu için `UPDATE` hiç çalışmaz. Eşleme hatası varsa yakalanmaz.

```java
// DOĞRU
@Test
void updateStock_persistsNewValue() {
    Product laptop = entityManager.persist(new Product(...));
    laptop.setStock(42);
    entityManager.flush();   // bekleyen değişiklikleri SQL olarak gönder
    entityManager.clear();   // persistence context'i boşalt

    Product found = productRepository.findById(laptop.getId()).get(); // gerçek SELECT
    assertThat(found.getStock()).isEqualTo(42);
}
```

| Metot | Ne Yapar? |
|---|---|
| `flush()` | Bekleyen değişiklikleri **hemen** SQL olarak veritabanına gönderir. Transaction açık kalır, hâlâ geri alınabilir. |
| `clear()` | Persistence context'i boşaltır. Sonraki okuma **gerçekten veritabanından** yapılır. |
| `saveAndFlush()` | Kaydeder ve hemen yazar. |

> **Kural:** Veriyi hazırladıktan sonra, doğrulamadan önce **`flush()` + `clear()`**. Aksi halde Hibernate'in önbelleği test edilir, veritabanı değil.

### Kısıt Testi

```java
@Test
void save_withDuplicateName_violatesUniqueConstraint() {
    productRepository.saveAndFlush(new Product("Laptop", ...));

    assertThatThrownBy(() -> productRepository.saveAndFlush(new Product("Laptop", ...)))
            .isInstanceOf(DataIntegrityViolationException.class);
}
```

Kısıt ihlali ancak SQL veritabanına gittiğinde ortaya çıkar, bu yüzden `saveAndFlush` kullanılır. Bu test **"son savunma hattının"** gerçekten var olduğunu kanıtlar.

## 4. N+1 Çözümünü Test Etmek

```java
@Test
void findAllWithCategory_loadsCategoryEagerly() {
    Category electronics = entityManager.persist(new Category("Elektronik"));
    entityManager.persist(new Product("Laptop", new BigDecimal("25000"), electronics));
    entityManager.flush();
    entityManager.clear(); // ZORUNLU

    List<Product> products = productRepository.findAllWithCategory();

    assertThat(Hibernate.isInitialized(products.get(0).getCategory())).isTrue();
}
```

- `Hibernate.isInitialized(...)`: Lazy ilişki **proxy** mi, yoksa yüklü gerçek nesne mi?
- `JOIN FETCH` çalışıyorsa `true`, normal `findAll()` ile `false`.
- `clear()` yapılmazsa kategori persistence context'te gerçek nesne olarak kalır ve **her zaman `true`** döner.

> `JOIN FETCH` kaldırılırsa uygulama hata vermez, sadece yavaşlar. Bu test o sessiz regression'ı yakalar.

## 5. H2'nin Sınırı ve Testcontainers

**H2 başka bir veritabanıdır.** Production veritabanından (PostgreSQL, MySQL) farkları:
- SQL lehçesi (native query'ler, `ILIKE`, JSON operatörleri)
- Veri tipleri (`jsonb`, diziler)
- Kısıt ve hata davranışları
- Flyway migration'ları gerçek veritabanının lehçesiyle yazılır

> **H2'de geçen testler production'da patlayabilir.** Testin değeri, production'a ne kadar benzediğiyle orantılıdır.

### Testcontainers

Test başlarken Docker'da **gerçek veritabanı** başlatır, test bitince siler. **Docker kurulu ve çalışıyor olmalı.**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class ProductRepositoryTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    // testler aynı kalır
}
```

| Anotasyon | Görevi |
|---|---|
| `@Testcontainers` + `@Container` | Konteynerin yaşam döngüsünü JUnit yönetir |
| `@ServiceConnection` | Bağlantı bilgilerini (rastgele port, kullanıcı, şifre) Spring Boot **otomatik** yapılandırır (3.1+) |
| `@AutoConfigureTestDatabase(replace = NONE)` | "H2 ile değiştirme, verdiğim veritabanını kullan" |
| **`static`** | Konteyner sınıftaki tüm testler için **bir kez** başlatılır |

- Eski yöntem: `@DynamicPropertySource` ile bağlantı bilgileri elle aktarılırdı.
- **Bedeli:** İlk başlatma birkaç saniye sürer ve Docker gerektirir.
- Testlerin kendisi **değişmez**, sadece altındaki veritabanı değişir.

## 6. @SpringBootTest

**Uygulamanın tamamını** yükler: Gerçek service'ler, repository'ler, security zinciri.

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
@TestPropertySource(properties = {
        "jwt.secret=dGVzdC1zZWNyZXQta2V5LWZvci1qd3QtdGVzdHMtMzItYnl0ZXM=",
        "jwt.expiration-ms=900000"
})
class AuthFlowIntegrationTest {

    @Autowired private MockMvc mockMvc;
    @Autowired private ObjectMapper objectMapper;
}
```

| Anotasyon | Görevi |
|---|---|
| `@AutoConfigureMockMvc` | Tam context'te `MockMvc` kullanılır. Sunucu başlamaz, ama istek gerçek service ve veritabanına iner. |
| `@Transactional` | Her test sonunda veritabanı değişiklikleri geri alınır |
| `@TestPropertySource` | Yapılandırma değerlerini test için ezer |

> ⚠️ `jwt.secret=${JWT_SECRET}` ortam değişkeni test ortamında yoksa `JwtService` oluşturulamaz ve **context ayağa kalkmaz.** Testler için sabit bir test anahtarı verilir (gizli değildir, production anahtarıyla ilgisi yoktur).
> Alternatif: `src/test/resources/application.properties`

### JWT Akışını Uçtan Uca Test Etmek

```java
@Test
void registerLoginAndAccessProtectedEndpoint() throws Exception {
    // 1. Kayıt
    mockMvc.perform(post("/api/auth/register")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            {"username": "ali", "password": "gizli123"}
                            """))
            .andExpect(status().isCreated());

    // 2. Giriş, token al
    MvcResult loginResult = mockMvc.perform(post("/api/auth/login")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            {"username": "ali", "password": "gizli123"}
                            """))
            .andExpect(status().isOk())
            .andReturn();

    String token = objectMapper.readValue(
            loginResult.getResponse().getContentAsString(), LoginResponse.class).token();

    // 3. Token ile korumalı endpoint
    mockMvc.perform(get("/api/orders").header("Authorization", "Bearer " + token))
            .andExpect(status().isOk());
}

@Test
void accessWithInvalidToken_returns401() throws Exception {
    mockMvc.perform(get("/api/orders").header("Authorization", "Bearer bozuk.bir.token"))
            .andExpect(status().isUnauthorized());
}
```

- `"""`: Metin blokları (Java 15+), JSON'u kaçış karakteri olmadan yazmayı sağlar.
- `andReturn()`: Cevabın kendisini döner, token çıkarılıp sonraki istekte kullanılır.

**İlk testin kanıtladıkları:**
- Şifre hashlenip kaydediliyor (hashlenmeseydi `matches` başarısız olur, giriş yapılamazdı)
- `AuthenticationManager` + `UserDetailsService` + `PasswordEncoder` birlikte çalışıyor
- Token doğru üretiliyor
- JWT filtresi token'ı doğrulayıp `SecurityContext`'e koyuyor
- Authorization filtresi kullanıcıyı tanıyor

> Birim testleri parçaları **ayrı ayrı** test eder. Bu test, parçaların **birbirine doğru bağlandığını** test eder.

**Bozuk token testi:** JWT filtresindeki `catch (JwtException)` bloğunun 500 yerine 401 ürettiğini doğrular.

### @PreAuthorize'ı Test Etmek

```java
@SpringBootTest
class ProductServiceSecurityTest {

    @Autowired
    private ProductService productService; // gerçek nesne değil, PROXY

    @Test
    @WithMockUser(roles = "USER")
    void delete_asUser_throwsAccessDenied() {
        assertThatThrownBy(() -> productService.delete(1L))
                .isInstanceOf(AccessDeniedException.class);
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void delete_asAdmin_isAllowed() {
        // önce ürün kaydedilmeli
        assertThatCode(() -> productService.delete(existingId))
                .doesNotThrowAnyException();
    }
}
```

- Service gerçek bean olduğu için `@PreAuthorize`'ı işleyen **proxy devrede.**
- `@WithMockUser` HTTP'ye bağlı değildir, burada da çalışır.

> **Proxy'nin araya girmesini test etmek için nesne Spring'den alınmalı.** `new ProductService(...)` ile oluşturulan nesnede proxy yoktur.

### MockMvc mi, Gerçek HTTP mi?

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
```

| | `@AutoConfigureMockMvc` | `RANDOM_PORT` + `TestRestTemplate` |
|---|---|---|
| Sunucu | Yok, simülasyon | Gerçek sunucu, rastgele port |
| Production'a yakınlık | Yüksek | En yüksek |
| `@Transactional` rollback | ✅ Çalışır | ❌ **Çalışmaz** |
| Temizlik | Otomatik | Elle (`@AfterEach` → `deleteAll()`) |

> **Rollback neden çalışmaz?** İstek sunucunun thread'inde işlenir, testin thread'inde değil. Test transaction'ı oraya ulaşamaz.

## 7. Context Önbelleği ve Test Hızı

- `@SpringBootTest` yavaştır (tüm uygulama ayağa kalkar).
- Spring, **aynı yapılandırmaya** sahip test sınıflarında context'i **yeniden kullanır.**
- Farklı `@MockitoBean` veya `@TestPropertySource` → **ayrı context** → her sınıfta uygulama yeniden başlar.

**Çözüm:** Ortak ayarlar için temel sınıf.

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
@Testcontainers
public abstract class BaseIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
}

class AuthFlowIntegrationTest extends BaseIntegrationTest { ... }
class OrderFlowIntegrationTest extends BaseIntegrationTest { ... }
```

Tüm entegrasyon testleri aynı context'i ve aynı konteyneri paylaşır.

## 8. Büyük Resim

| Test | Araç | Test Eder | Test Edemez (Kör Nokta) |
|---|---|---|---|
| **Birim** | JUnit + Mockito | İş kuralları, sınır durumları, yan etkiler | Sorgular, HTTP, security, Spring davranışları |
| **Web dilimi** | `@WebMvcTest` | URL eşleme, JSON, validation, exception handler, URL security | İş mantığı (mock), `@PreAuthorize`, JWT filtresi |
| **JPA dilimi** | `@DataJpaTest` | Sorgular, eşlemeler, kısıtlar, `JOIN FETCH` | İş mantığı, HTTP, security |
| **Entegrasyon** | `@SpringBootTest` | Parçaların birlikte çalışması, JWT akışı, `@PreAuthorize` | (Her şeyi test eder ama yavaştır) |

> Her test türünün **kör noktası** var, bir sonraki tür onu kapatır. Hiçbiri tek başına yeterli değil.

### Pratik Dağılım

- **İş kurallarının tamamı** → Birim testi (çok sayıda, hızlı)
- **Her controller** → Birkaç web dilimi testi
- **Özel yazılan her sorgu** → JPA dilimi testi
- **Kritik akışlar** (kayıt-giriş, sipariş) → Birkaç entegrasyon testi

---

## Sorular

**1. Repository testlerinde neden `save` ve `findById` test edilmez?**
Spring Data'nın kendi kodudur ve kütüphane tarafından test edilir. Geliştiricinin yazdığı sorgular, eşlemeler ve kısıtlar test edilmelidir.

**2. `@DataJpaTest` varsayılan olarak hangi veritabanını kullanır ve testler neden birbirini etkilemez?**
Classpath'te H2 varsa bellek içi H2 kullanır. Her test bir transaction içinde çalışır ve sonunda geri alınır.

**3. Test verisi neden test edilen repository yerine `TestEntityManager` ile hazırlanır?**
Test edilen şeyle veri hazırlanırsa, oradaki hata hem hazırlığı hem doğrulamayı bozup birbirini gizleyebilir.

**4. `@DataJpaTest`'te entity değiştirilip `findById` ile okunduğunda yeni değer geliyor. `UPDATE`'in çalıştığı kanıtlanmış olur mu?**
Hayır. `findById` persistence context'teki aynı nesneyi döner ve rollback yüzünden `UPDATE` hiç çalışmaz. `flush()` + `clear()` gerekir.

**5. `flush()` ile commit farkı nedir?**
`flush()` değişiklikleri SQL olarak gönderir ama transaction açık kalır ve geri alınabilir. Commit transaction'ı kalıcı olarak sonlandırır.

**6. `unique` kısıtı testinde neden `saveAndFlush` kullanılır?**
Kısıt ihlali ancak SQL veritabanına gittiğinde ortaya çıkar.

**7. `JOIN FETCH`'in çalıştığı nasıl test edilir? `clear()` neden gerekir?**
`Hibernate.isInitialized(...)` ile ilişkinin yüklü olduğu doğrulanır. `clear()` yapılmazsa ilişkili entity persistence context'te kalır ve sonuç her zaman `true` olur.

**8. Testlerde H2 kullanmanın riski nedir?**
H2 production veritabanından farklıdır (lehçe, tipler, davranışlar). H2'de geçen testler production'da patlayabilir.

**9. Testcontainers nedir? `@ServiceConnection` ne işe yarar?**
Testler için Docker'da gerçek veritabanı başlatan kütüphanedir. `@ServiceConnection` konteynerin bağlantı bilgilerini Spring Boot'a otomatik aktarır.

**10. Testcontainers konteyneri neden `static` tanımlanır?**
Sınıftaki tüm testler için bir kez başlatılsın diye. Aksi halde her testte yeni veritabanı başlar.

**11. `@SpringBootTest` context'i `JwtService` yüzünden ayağa kalkmıyor. Sebep ne olabilir?**
`jwt.secret` ortam değişkeninden okunuyor ve testte yok. `@TestPropertySource` veya test `application.properties` ile değer verilmelidir.

**12. `@PreAuthorize` neden `@WebMvcTest`'te değil `@SpringBootTest`'te test edilir?**
`@WebMvcTest`'te service mock'lanır, proxy yoktur. `@SpringBootTest`'te service gerçek bean'dir ve proxy ile sarılıdır.

**13. `RANDOM_PORT` testlerinde `@Transactional` neden rollback yapmaz?**
İstek sunucunun thread'inde işlenir; test transaction'ı o thread'e ulaşamaz. Temizlik elle yapılmalıdır.

**14. Entegrasyon testleri neden yavaşlar? Nasıl önlenir?**
Farklı yapılandırmalar (farklı `@MockitoBean`, özellikler) her sınıf için ayrı context oluşturur. Ortak bir temel sınıftan türetmek context'in paylaşılmasını sağlar.

**15. Uçtan uca JWT testi, şifrenin hashlendiğini nasıl dolaylı olarak doğrular?**
Giriş sırasında `matches` gönderilen şifreyi kayıtlı hash ile karşılaştırır. Şifre düz metin kaydedilseydi eşleşme başarısız olur, giriş yapılamazdı.
