# Spring Boot: Test

## 0. Neden ve Nasıl Test?

### Test Piramidi

```
          /\         E2E            → Az, yavaş, pahalı
         /  \
        /----\       Entegrasyon    → Kritik akışlar, gerçek DB / önbellek
       /      \
      /--------\     Slice          → Tek katman (web, veri)
     /          \
    /------------\   Birim          → Çok, hızlı, ucuz
```

| Seviye | Ne Test Edilir? | Spring Context | Hız |
|---|---|---|---|
| **Birim** | Tek sınıf, bağımlılıklar mock'lu | Yok | Milisaniye |
| **Slice** | Tek katman (`@WebMvcTest`, `@DataJpaTest`...) | Kısmi | Saniye |
| **Entegrasyon** | Tüm katmanlar + gerçek altyapı | Tam (`@SpringBootTest`) | Saniyeler |
| **E2E** | Çalışan uygulama, gerçek istemci | Ayrı süreç | Dakika |

> **Kural:** Bir davranışı **en alttaki** (en ucuz) seviyede test et. Üst seviyeye ancak alt seviye o davranışı göremiyorsa çık.

### Sektörde Yaygın Araç Seti

| Amaç | Araç | Not |
|---|---|---|
| Test çatısı | **JUnit 5 (Jupiter)** | Fiili standart |
| Doğrulama (assertion) | **AssertJ** | `assertThat(...)` zinciri |
| Mock | **Mockito** | Spring'e entegre: `@MockitoBean` |
| HTTP katmanı | **MockMvc**, **WebTestClient**, **RestAssured** | |
| Gerçek veritabanı / broker | **Testcontainers** | H2'nin yerini aldı |
| Dış HTTP servislerini taklit | **WireMock** | |
| Asenkron bekleme | **Awaitility** | `Thread.sleep()` yerine |
| Sözleşme testi | **Spring Cloud Contract**, **Pact** | Mikroservisler arası |
| Mimari kurallar | **ArchUnit** | Katman / paket bağımlılıkları |
| Kapsam (coverage) | **JaCoCo** | CI'da raporlama |
| Mutasyon testi | **PIT (Pitest)** | Testlerin kalitesini ölçer |

### Bağımlılık

```xml
<!-- JUnit 5, AssertJ, Mockito, MockMvc, Hamcrest, JsonPath, Awaitility... hepsi içinde -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

> Sürüm yazılmaz, Spring Boot BOM'u yönetir.

---

## 1. Birim Testi: JUnit 5 + Mockito

### Spring'siz Test

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderRepository orders;
    @Mock PaymentClient payment;
    @InjectMocks OrderService service;

    @Test
    void odemeBasariliysaSiparisOnaylanir() {
        // Given
        given(orders.findById(1L)).willReturn(Optional.of(new Order(1L)));
        given(payment.charge(any())).willReturn(PaymentResult.OK);

        // When
        service.complete(1L);

        // Then
        then(orders).should().save(argThat(o -> o.status() == CONFIRMED));
    }
}
```

| Parça | Anlamı |
|---|---|
| `@Mock` | Sahte nesne oluşturur |
| `@InjectMocks` | Mock'ları test edilen sınıfa constructor ile verir |
| `given(...).willReturn(...)` | Mock'un davranışını tanımlar (BDD stili `when/thenReturn`) |
| `then(mock).should()` | Metodun çağrıldığını doğrular (`verify`) |
| `any()`, `argThat(...)` | Argüman eşleyicileri |

- Spring context **açılmaz** → En hızlı test türü.
- Constructor injection kullanan sınıflar Spring olmadan kolayca test edilir. (Field injection'dan kaçınmanın bir sebebi de bu.)

### Parametreli Test

```java
@ParameterizedTest
@CsvSource({ "100, 0", "1000, 50", "5000, 500" })
void indirimHesaplanir(int tutar, int beklenen) {
    assertThat(calculator.discount(tutar)).isEqualTo(beklenen);
}
```

- Aynı testi farklı girdilerle çalıştırır. `@ValueSource`, `@EnumSource`, `@MethodSource` da kullanılabilir.

## 2. AssertJ: Doğrulamalar

| Kullanım | Örnek |
|---|---|
| Eşitlik | `assertThat(count).isEqualTo(1)` |
| Boolean / null | `isTrue()`, `isFalse()`, `isNull()`, `isNotNull()` |
| Koleksiyon | `hasSize(3)`, `contains(...)`, `containsExactly(...)`, `extracting(User::name)` |
| String | `startsWith(...)`, `contains(...)`, `matches(regex)` |
| Hata bekleniyor | `assertThatThrownBy(() -> ...).isInstanceOf(X.class).hasMessageContaining("...")` |
| Hata beklenmiyor | `assertThatCode(() -> ...).doesNotThrowAnyException()` |
| Tüm hataları topla | `SoftAssertions.assertSoftly(soft -> { ... })` |

> JUnit'in `assertEquals(expected, actual)` sırası sık karıştırılır. AssertJ cümle gibi okunur ve IDE uygun metotları önerir; sektörde bu yüzden tercih edilir.

## 3. Slice Testler: Sadece Bir Katman

`@SpringBootTest` tüm uygulamayı yükler. Slice testler yalnızca ilgili katmanın bean'lerini yükler.

| Anotasyon | Yüklenen | Tipik Kullanım |
|---|---|---|
| `@WebMvcTest(XController.class)` | Controller, filtreler, `@ControllerAdvice`, Jackson | Request/response, validation, hata kodları |
| `@DataJpaTest` | JPA, repository'ler, `DataSource` | Özel sorgular, mapping |
| `@JdbcTest` | `JdbcTemplate`, `DataSource` | |
| `@JsonTest` | Jackson | Serileştirme formatı |
| `@RestClientTest` | `RestClient` / `RestTemplate` + sahte sunucu | Dış API istemcisi |
| `@WebFluxTest` | WebFlux controller'ları | Reaktif uygulamalar |

### `@WebMvcTest` Örneği

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired MockMvc mockMvc;
    @MockitoBean UserService userService;   // Servis katmanı yüklenmez, mock verilir

    @Test
    void gecersizIstek400Doner() throws Exception {
        mockMvc.perform(post("/api/users")
                        .contentType(APPLICATION_JSON)
                        .content("""
                                {"email":"gecersiz"}
                                """))
                .andExpect(status().isBadRequest());
    }
}
```

- `@MockitoBean`, Spring context'indeki bean'i mock ile **değiştirir.**

> ⚠️ `@MockBean` Spring Boot 3.4'te deprecated oldu, 4.0'da kaldırıldı. Yerine `@MockitoBean` / `@MockitoSpyBean` kullanılır.

> ⚠️ `@DataJpaTest` varsayılan olarak gömülü veritabanı (H2) kullanmaya çalışır. Gerçek veritabanıyla çalışmak için `@AutoConfigureTestDatabase(replace = NONE)` + Testcontainers.

## 4. Entegrasyon Testi: `@SpringBootTest`

```java
@SpringBootTest
@AutoConfigureMockMvc
class OrderIntegrationTest {

    @Autowired MockMvc mockMvc;

    @Test
    void siparisOlusturulur() throws Exception {
        mockMvc.perform(post("/api/orders").contentType(APPLICATION_JSON).content("..."))
               .andExpect(status().isCreated())
               .andExpect(jsonPath("$.status").value("CREATED"));
    }
}
```

### `webEnvironment` Seçenekleri

| Değer | Sunucu | HTTP İstemcisi |
|---|---|---|
| `MOCK` (varsayılan) | Yok, sahte servlet ortamı | `MockMvc` |
| `RANDOM_PORT` | Gerçek sunucu, rastgele port | `TestRestTemplate`, `WebTestClient`, `RestAssured` |
| `DEFINED_PORT` | Gerçek sunucu, tanımlı port | Aynı |
| `NONE` | Web yok | Sadece servis / bean testi |

### HTTP Test İstemcileri

| İstemci | Nerede? | Özellik |
|---|---|---|
| **MockMvc** | `MOCK` ortamı | Sunucu açmaz, en yaygın |
| **MockMvcTester** | `MOCK` ortamı | MockMvc'nin AssertJ tarzı hali (Spring 6.2+) |
| **WebTestClient** | WebFlux veya `RANDOM_PORT` | Akıcı API, reaktif uygulamalarda standart |
| **TestRestTemplate** | `RANDOM_PORT` | Gerçek HTTP; eski ama yaygın |
| **RestAssured** | `RANDOM_PORT` | `given().when().then()` söz dizimi; QA ekiplerinde popüler |

### MockMvc Kısa Başvuru

| Metot | Anlamı |
|---|---|
| `perform(get/post/put/patch/delete(...))` | İsteği çalıştırır |
| `.header(...)`, `.param(...)`, `.contentType(...)`, `.content(...)` | İsteği tanımlar |
| `.andExpect(status().isOk())` | Durum kodu |
| `.andExpect(jsonPath("$.alan").value(x))` | JSON alanı (`$` = kök) |
| `.andExpect(jsonPath("$.alan").doesNotExist())` | Alan olmamalı |
| `.andReturn()` | Ham sonuç |
| `.with(jwt())`, `.with(user("ali").roles("ADMIN"))` | Güvenlik (spring-security-test) |

### ⚠️ `@Transactional` Test Tuzağı

- Test metodu `@Transactional` ise test sonunda her şey **rollback** edilir → Temiz ama **commit hiç olmaz.**
- Bu durumda `@TransactionalEventListener(AFTER_COMMIT)`, commit sonrası önbellek silme, `@ApplicationModuleListener` gibi davranışlar **çalışmaz.**
- `RANDOM_PORT` ile atılan HTTP isteği **başka thread'de** çalışır → Test transaction'ına katılmaz, geri **alınmaz.**

| Yalıtım Yöntemi | Gerçek Commit | Hız |
|---|---|---|
| `@Transactional` rollback | ❌ | Hızlı |
| Her test benzersiz veri (`UUID`) kullanır | ✅ | Hızlı |
| `@Sql` / `TRUNCATE` ile temizlik | ✅ | Yavaş |

## 5. Testcontainers: Gerçek Altyapı

### Neden H2 Değil?

- H2, PostgreSQL/MySQL'in söz dizimini ve davranışını **birebir** taklit etmez (`ON CONFLICT`, JSONB, kilitler...).
- "H2'de geçti, production'da patladı" → Test güven vermez.
- Testcontainers, testte **gerçek** veritabanını Docker'da açar. Sektörde bugün standart yaklaşım.

### `@ServiceConnection` ile Kurulum (Spring Boot 3.1+)

```java
@TestConfiguration(proxyBeanMethods = false)
class TestcontainersConfig {

    @Bean
    @ServiceConnection   // URL, kullanıcı, şifre otomatik bağlanır
    PostgreSQLContainer<?> postgres() {
        return new PostgreSQLContainer<>("postgres:17");
    }
}
```

```java
@SpringBootTest
@Import(TestcontainersConfig.class)
class OrderIntegrationTest { ... }
```

| Desteklenen Örnekler |
|---|
| PostgreSQL, MySQL, MongoDB, Redis, Kafka, RabbitMQ, Elasticsearch, LocalStack (AWS) |

- `@DynamicPropertySource` eski yöntemdir; `@ServiceConnection` ile gerek kalmaz.
- Container bean olarak tanımlanınca Spring context'iyle **birlikte** yaşar → Context önbelleğiyle uyumlu.

> **Geliştirmede de kullanılabilir:** `SpringApplication.from(App::main).with(TestcontainersConfig.class).run(args)` ile uygulama yerelde container'larla ayağa kalkar. Ekipte "bende çalışıyor" sorunu azalır.

> ⚠️ Testcontainers 2.x'te modül adları (`testcontainers-postgresql`) ve bazı paketler değişti. Sürüme göre kontrol et.

## 6. Dış Servisler ve Asenkron Davranış

### WireMock: Dış HTTP API'yi Taklit Etmek

```java
stubFor(get(urlEqualTo("/rates/USD"))
        .willReturn(okJson("""
                {"rate": 34.5}
                """)));
```

- Gerçek ödeme / kur / SMS servisine gitmeden istemci kodu test edilir.
- Hata senaryoları kolayca kurulur: `serverError()`, `withFixedDelay(5000)` (timeout testi).
- Mockito ile istemci sınıfını mock'lamaktan farkı: **HTTP katmanı** da (serileştirme, header, timeout) test edilir.

### Awaitility: Asenkron Sonucu Beklemek

```java
await().atMost(Duration.ofSeconds(5))
       .untilAsserted(() -> assertThat(stockOf(100L)).isEqualTo(9));
```

- `@Async`, event dinleyicileri, Kafka tüketicileri için.
- ❌ `Thread.sleep(2000)`: Ya gereksiz bekletir ya da yetmez → Flaky test.
- ✅ Awaitility koşul sağlanır sağlanmaz devam eder.

## 7. Mimari ve Sözleşme Testleri

### ArchUnit: Mimari Kurallar Testte

```java
@ArchTest
static final ArchRule controllerRepositoryKullanamaz =
        noClasses().that().resideInAPackage("..controller..")
                   .should().dependOnClassesThat().resideInAPackage("..repository..");
```

- Katman ihlalleri, döngüsel bağımlılıklar, isimlendirme kuralları CI'da yakalanır.
- Spring Modulith'in `verify()`'ı arkada ArchUnit kullanır.

### Sözleşme Testi (Consumer-Driven Contracts)

| Araç | Yaklaşım |
|---|---|
| **Spring Cloud Contract** | Sözleşme üretici tarafında yazılır; testler ve tüketici için stub otomatik üretilir |
| **Pact** | Sözleşmeyi tüketici yazar; dil bağımsız, Pact Broker ile paylaşılır |

- Mikroservisler arasında "API değişti, karşı taraf kırıldı" sorununu **deploy'dan önce** yakalar.

## 8. İyi Pratikler

### Context Önbelleğini Koru

- Spring, **aynı konfigürasyondaki** test sınıfları için context'i bir kez açar ve paylaşır.
- Farklı `@MockitoBean`, `properties`, `@ActiveProfiles` veya `@DirtiesContext` → **Yeni context** → Yavaş test.
- **Çözüm:** Entegrasyon testleri ortak bir taban sınıftan türesin; sınıfa özel mock'lardan kaçın.

### Testi Okunabilir Yaz

| Pratik | Açıklama |
|---|---|
| **Given – When – Then** (Arrange – Act – Assert) | Hazırla, çalıştır, doğrula |
| **Davranışı anlatan isim** | `testOrder2()` ❌ → `stokYetersizseSiparisReddedilir()` ✅ |
| **`@DisplayName`** | Raporda okunaklı açıklama |
| **`@Nested`** | Testleri konuya göre grupla |
| **Test Data Builder / Object Mother** | `anOrder().withStatus(PAID).build()` ile test verisi |
| **Bir test, bir davranış** | Kırıldığında sebebi belli olsun |

### Neyi Test Etme

- Framework'ün kendisini (Spring'in `@Autowired`'ı çalışıyor mu?).
- Getter/setter, basit DTO'lar.
- Sadece mock'ların birbirini çağırdığını doğrulayan, davranış içermeyen testler.

### Kapsam ve Kalite

| Araç | Ölçtüğü |
|---|---|
| **JaCoCo** | Kodun yüzde kaçı testlerde **çalıştı** |
| **PIT** | Kod değiştirilince (mutasyon) testler bunu **yakaladı mı** |

> ⚠️ %100 coverage, kodun **doğru** olduğunu göstermez; sadece çalıştırıldığını gösterir. Doğrulama yapmayan test de coverage'ı artırır. Mutasyon testi bu açığı kapatır.

## 9. Ne Zaman Hangi Test?

| Test Edilecek Şey | Önerilen Seviye |
|---|---|
| İş kuralı, hesaplama | Birim testi (+ `@ParameterizedTest`) |
| Validation, HTTP durum kodları, JSON formatı | `@WebMvcTest` |
| Özel JPA / SQL sorgusu | `@DataJpaTest` + Testcontainers |
| Dış API istemcisi | `@RestClientTest` veya WireMock |
| Transaction, önbellek, event'ler, güvenlik zinciri | `@SpringBootTest` + Testcontainers |
| Asenkron işlemler | `@SpringBootTest` + Awaitility |
| Katman / modül kuralları | ArchUnit / Spring Modulith |
| Servisler arası API uyumu | Spring Cloud Contract / Pact |

> **Pratik bakış:**
> - Birim testleri "kod **yazdığın gibi** çalışıyor mu?", entegrasyon testleri "sistem **beklendiği gibi** çalışıyor mu?" sorusunu cevaplar.
> - Modern Spring projelerinde yaygın denge: Bol birim testi + slice testler + Testcontainers ile az sayıda ama kritik entegrasyon testi.
> - Yavaş test suite'i çalıştırılmaz. Context önbelleği ve container'ları yeniden kullanmak, hızlı testin anahtarıdır.

---

## Sorular

**1. Test piramidi neyi anlatır?**
Altta çok sayıda hızlı birim testi, üstlere doğru daha az sayıda ama daha kapsamlı ve yavaş test olmasını. Her davranış en ucuz seviyede test edilmelidir.

**2. `@Mock` ile `@MockitoBean` farkı nedir?**
`@Mock` Spring'siz, saf Mockito mock'udur. `@MockitoBean` Spring context'indeki bean'i mock ile değiştirir.

**3. `@MockBean` neden artık kullanılmamalı?**
Spring Boot 3.4'te deprecated oldu, 4.0'da kaldırıldı; yerine `@MockitoBean` geldi.

**4. Slice test nedir? Örnek verin.**
Sadece tek bir katmanın bean'lerini yükleyen testtir. `@WebMvcTest` web katmanını, `@DataJpaTest` JPA katmanını yükler.

**5. `@SpringBootTest`'in `MOCK` ve `RANDOM_PORT` modları arasındaki fark nedir?**
`MOCK` gerçek sunucu açmaz, MockMvc ile test edilir. `RANDOM_PORT` gerçek sunucuyu rastgele portta açar; TestRestTemplate, WebTestClient veya RestAssured ile istek atılır.

**6. `@Transactional` test hangi davranışları göremez?**
Commit olmadığı için `AFTER_COMMIT` dinleyicilerini, commit sonrası önbellek silmeyi ve event tabanlı asenkron işlemleri. `RANDOM_PORT` ile atılan istekler de rollback edilmez.

**7. Testlerde neden H2 yerine Testcontainers tercih edilir?**
H2 gerçek veritabanının söz dizimini ve davranışını birebir taklit etmez. Testcontainers gerçek veritabanını Docker'da açar; testler production'a daha çok güven verir.

**8. `@ServiceConnection` ne işe yarar?**
Container'ın bağlantı bilgilerini (URL, kullanıcı, şifre) Spring'e otomatik verir; `@DynamicPropertySource` yazmaya gerek bırakmaz.

**9. Spring test context önbelleği nedir?**
Aynı konfigürasyondaki test sınıfları için context'in bir kez oluşturulup paylaşılmasıdır. Farklı mock'lar veya property'ler yeni context açtırıp testleri yavaşlatır.

**10. WireMock ne zaman kullanılır?**
Dış HTTP servislerini taklit etmek için. Mockito'dan farkı, serileştirme, header ve timeout gibi HTTP katmanı davranışlarının da test edilmesidir.

**11. Asenkron kodu test ederken neden `Thread.sleep()` kullanılmamalı?**
Ya gereksiz bekletir ya da yetmez ve test rastgele kırılır. Awaitility koşul sağlanır sağlanmaz devam eder.

**12. ArchUnit ne işe yarar?**
Katman ve paket bağımlılığı gibi mimari kuralları test olarak yazıp CI'da doğrulamayı sağlar.

**13. Sözleşme testi hangi sorunu çözer?**
Mikroservisler arasında API değişikliğinin karşı tarafı kırmasını deploy'dan önce yakalar. Yaygın araçlar Spring Cloud Contract ve Pact'tir.

**14. %100 kod kapsamı neden yeterli değildir?**
Kapsam sadece kodun çalıştırıldığını gösterir, doğru sonucu ürettiğini değil. Mutasyon testi (PIT) testlerin gerçekten hata yakalayıp yakalamadığını ölçer.

**15. Hangi durumda birim testi, hangi durumda entegrasyon testi seçilir?**
Saf iş kuralları birim testiyle test edilir. Transaction, önbellek, event, güvenlik zinciri ve veritabanı kısıtları gibi bileşenlerin birlikte çalışmasına bağlı davranışlar entegrasyon testi gerektirir.
