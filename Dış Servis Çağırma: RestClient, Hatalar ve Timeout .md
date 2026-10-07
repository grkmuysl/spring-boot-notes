# Dış Servis Çağırma - Bölüm 1: RestClient, Hatalar ve Timeout

## 1. Sorun: Kontrolünde Olmayan Bağımlılık

Gerçek uygulamalar başka servislere bağımlıdır: ödeme, kargo, e-posta, döviz kuru. Bu çağrılarda uygulama bir **istemcidir.**

Dış servis:
- Yavaşlayabilir, çökebilir
- Haber vermeden bir alanın adını değiştirebilir
- İstek sayısını sınırlayabilir
- Aradaki **ağ** güvenilir değildir: paketler kaybolur, bağlantılar kopar

> Redis'teki ilkeler burada en çok sınanır: **Yavaş bir bağımlılık tüm uygulamayı yavaşlatmamalı, bir bileşenin bozulması sistemi durdurmamalı.**

## 2. Spring'de HTTP İstemcileri

| İstemci | Durum |
|---|---|
| `RestTemplate` | En eski. Bakım modunda, yeni özellik eklenmiyor. |
| `WebClient` | Reaktif (WebFlux) için. Klasik MVC'de gereksiz karmaşıklık. |
| **`RestClient`** | Spring 6.1 / Boot 3.2+. Akıcı yazım + senkron model. **Yeni projeler için önerilen.** |
| HTTP Interface (`@HttpExchange`) | Bildirimsel: Arayüz yazılır, implementasyonu Spring üretir. |

> Spring Cloud'daki **OpenFeign**, HTTP Interface'in öncülüdür.

## 3. RestClient ile İlk Çağrı

### Ayarlar

```java
@ConfigurationProperties(prefix = "clients.exchange-rate")
@Validated
public record ExchangeRateProperties(
        @NotBlank String baseUrl,
        @NotBlank String apiKey
) {}
```

```yaml
clients:
  exchange-rate:
    base-url: https://api.example-rates.com
    api-key: ${EXCHANGE_RATE_API_KEY}   # gizli bilgi, varsayılan yok
```

### İstemci

```java
@Component
public class ExchangeRateClient {

    private final RestClient restClient;

    public ExchangeRateClient(RestClient.Builder builder, ExchangeRateProperties properties) {
        this.restClient = builder
                .baseUrl(properties.baseUrl())
                .defaultHeader("X-Api-Key", properties.apiKey())
                .build();
    }

    public ExchangeRateResponse getRate(String from, String to) {
        return restClient.get()
                .uri("/v1/rates?base={from}&target={to}", from, to)
                .retrieve()
                .body(ExchangeRateResponse.class);
    }
}
```

- `retrieve()`: İsteği gönderir.
- `body(...)`: JSON cevabı Jackson ile nesneye çevirir.

### URI Şablonları

> ⚠️ String birleştirme (`"/rates?base=" + from`) yapılmaz. Özel karakterler URL'yi bozar, kullanıcı değeri URL'ye parametre **enjekte edebilir.**
> Yer tutucular (`{from}`) değerleri doğru kodlar (URL encoding). JPA'daki "sorgu parametresi kullan" kuralının HTTP versiyonu.

### Neden RestClient.Builder Enjekte Edilir?

`RestClient.create()` yerine Spring Boot'un builder'ı:
- Uygulamanın **`ObjectMapper`** ayarları kullanılır.
- **`http.client.requests`** metrikleri otomatik toplanır (Actuator).
- Tracing varsa **`traceId`** istek başlıklarına otomatik eklenir.

> **Builder prototype scope'tadır:** Her enjeksiyonda yeni builder gelir.
> **Neden?** Builder değiştirilebilir. Singleton olsaydı `ExchangeRateClient`'ın eklediği `baseUrl` ve API anahtarı, aynı builder'ı alan `CargoClient`'a sızardı. Singleton'da değiştirilebilir state paylaşma tehlikesi.

### Dış Servisin DTO'ları

```java
public record ExchangeRateResponse(String base, String target, BigDecimal rate, Instant timestamp) {}
```

```java
// Service katmanı kendi kavramlarıyla çalışır
public BigDecimal convert(BigDecimal amount, String from, String to) {
    ExchangeRateResponse response = exchangeRateClient.getRate(from, to);
    return amount.multiply(response.rate());
}
```

- Dış DTO **istemci sınıfının dışına sızmamalı.**
- Sağlayıcı alan adını değiştirirse veya sağlayıcı değişirse, değişiklik sadece istemcide kalır.
- **Anti-corruption layer:** Dış sistemin modeli senin modelini "bozmasın."

| DTO Konusu | Bu Konu |
|---|---|
| **Bizim** sunduğumuz sözleşmeyi iç modelden koruyorduk | **Dış dünyanın** sözleşmesinden kendimizi koruyoruz |

> Spring Boot'un `ObjectMapper`'ı tanımadığı alanları **yok sayar.** Sağlayıcı yeni alan eklerse uygulama bozulmaz.

## 4. Hataları Ele Almak

| Durum | Exception |
|---|---|
| 4xx cevap | `HttpClientErrorException` (örn. `.NotFound`) |
| 5xx cevap | `HttpServerErrorException` |
| Bağlantı kurulamadı, timeout | `ResourceAccessException` |

### ⚠️ Dış Servisin Hata Kodunu Kullanıcıya Taşıma

**Senaryo:** Ödeme sağlayıcısının API anahtarı yanlış, sağlayıcı **401** dönüyor.
- 401 olduğu gibi iletilirse ön yüz "kullanıcı kimliği doğrulanmadı" sanır ve **kullanıcıyı çıkış yaptırır.**
- Kullanıcı tekrar girer, tekrar dener, tekrar atılır. Oysa sorun **bizim** API anahtarımızda.

> **Kural:** Dış servisin hata kodları, senin API'nin hata kodları değildir. Kendi kavramlarına çevir.

```java
public ExchangeRateResponse getRate(String from, String to) {
    return restClient.get()
            .uri("/v1/rates?base={from}&target={to}", from, to)
            .retrieve()
            .onStatus(status -> status.value() == 404, (request, response) -> {
                throw new BusinessException("Desteklenmeyen döviz çifti: " + from + "/" + to);
            })
            .onStatus(HttpStatusCode::isError, (request, response) -> {
                throw new ExternalServiceException(
                        "Döviz kuru servisi hata döndü: " + response.getStatusCode());
            })
            .body(ExchangeRateResponse.class);
}
```

- **İlk eşleşen `onStatus` çalışır:** Spesifik (404) genelden (tüm hatalar) önce.
- 404 = Kullanıcı desteklenmeyen döviz istedi → İş kuralı hatası → 400.

```java
@ExceptionHandler(ExternalServiceException.class)
public ResponseEntity<ErrorResponse> handleExternal(ExternalServiceException ex) {
    log.error("Dış servis hatası", ex);
    return build(HttpStatus.BAD_GATEWAY,
            "Bağlı bir servise şu anda ulaşılamıyor, lütfen daha sonra tekrar deneyin");
}
```

- İç detaylar (sağlayıcı adı, cevap kodu) **loglanır**, istemciye gitmez.

### Ağ Geçidi Hata Kodları

| Kod | Anlamı |
|---|---|
| **502 Bad Gateway** | Bağımlı servis geçersiz / hatalı cevap verdi |
| **503 Service Unavailable** | Şu an hizmet veremiyorum (geçici) |
| **504 Gateway Timeout** | Bağımlı servis zamanında cevap vermedi |

Hepsi 5xx (hata istemcide değil) ve "daha sonra tekrar denenebilir" anlamı taşır.

### Hata Kimin?

| Dış Servisin Cevabı | Sorun Kimde? | Tekrar Denemek? | Log Seviyesi |
|---|---|---|---|
| **5xx** | Onlarda, genelde geçici | İşe yarayabilir | WARN / ERROR |
| **4xx** (meşru iş durumları hariç) | Genelde **bizde**: yanlış format, eksik alan, yanlış anahtar | Hiçbir şey değiştirmez | **ERROR** (kodumuzda hata var) |

> Bu ayrım, Bölüm 2'deki "hangi hatalarda retry yapılır?" sorusunun cevabıdır.

## 5. Timeout'lar: En Önemli Ayar

| Timeout | Ne Zaman Dolar? | Tipik Değer |
|---|---|---|
| **Connect timeout** | Sunucu erişilemiyorsa (kapalı, ağ yolu yok) | 1-3 sn |
| **Read timeout** | Bağlantı kuruldu ama cevap gelmiyorsa (sunucu yavaş) | Ölçüme göre |

> ⚠️ HTTP kütüphanesine bağlı olarak varsayılan **sonsuz** olabilir. Dış servis cevap vermezse thread **sonsuza kadar** bekler.

### Zincirleme Çöküş (Cascading Failure)

```
Tomcat thread havuzu: 200 thread
Döviz servisi yavaşladı → her istek 60 sn takılıyor
Bir dakikada 200 thread'in HEPSİ döviz servisini bekliyor
→ Giriş, sepet, sipariş... HİÇBİR endpoint cevap veremiyor
```

> Bir yan servisin yavaşlaması **tüm uygulamayı** çökertti. HikariCP havuzu dolması senaryosunun thread havuzu versiyonu.

### Timeout Ayarlamak

```java
public ExchangeRateClient(RestClient.Builder builder, ExchangeRateProperties properties) {
    HttpClient httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(2))
            .build();
    JdkClientHttpRequestFactory requestFactory = new JdkClientHttpRequestFactory(httpClient);
    requestFactory.setReadTimeout(Duration.ofSeconds(3));

    this.restClient = builder
            .baseUrl(properties.baseUrl())
            .defaultHeader("X-Api-Key", properties.apiKey())
            .requestFactory(requestFactory)
            .build();
}
```

- `RestClient` HTTP işini bir **request factory**'ye devreder.
- Timeout aşılınca `ResourceAccessException` → `ExternalServiceException` → **504**
- Değerleri `@ConfigurationProperties`'e taşımak her ortamda ayrı ayarlamayı sağlar.
- Yeni Boot sürümlerinde `application.yml` özellikleri de var; sürüm belgelerine bakılmalı.

> **Timeout'u açıkça ayarlamadan production'a dış servis çağrısı gönderme.**

### Hangi Değer?

- `http.client.requests` metriği gerçek cevap sürelerini gösterir.
- Normal 200 ms, en yavaşların çoğu < 800 ms ise → 3 sn read timeout makul.
- Çok kısa → Normal yavaşlıklarda gereksiz hata.
- Çok uzun → Thread'ler gereksiz bağlanır.

> **Önce ölç, sonra ayarla.**

## 6. ⚠️ Dış Çağrıyı Transaction İçinde Yapma

```java
@Transactional
public OrderResponse placeOrder(OrderRequest request) {
    Order order = orderRepository.save(new Order(...));      // 1. veritabanı
    paymentClient.charge(order.getTotal(), request.card());  // 2. dış servis (3 sn)
    order.markAsPaid();                                      // 3. veritabanı
    return toResponse(order);
}
```

| Sorun | Açıklama |
|---|---|
| **Bağlantı havuzu** | Dış çağrı süresince veritabanı bağlantısı **boşta meşgul.** HikariCP varsayılan 10 bağlantı; 10 eşzamanlı siparişte havuz dolar, tüm veritabanı erişimi bekler. |
| **Geri alınamayan yan etki** | Ödeme alındı, 3. adımda hata → **rollback.** Veritabanında sipariş yok, ama **kartından para çekildi.** |

> Transaction'ın "ya hepsi ya hiçbiri" garantisi **sadece veritabanı** için geçerli. Dış servis çağrısını geri alamaz.

### Doğru Yaklaşım

```
1. Kısa transaction: Siparişi "ÖDEME_BEKLİYOR" durumunda kaydet
2. Transaction DIŞINDA: Ödeme servisini çağır
3. Kısa transaction: Sonuca göre "ÖDENDİ" veya "ÖDEME_BAŞARISIZ" yap
```

- Sistem her an **tutarlı ve açıklanabilir** bir durumda (state machine mantığı).
- Asenkron işler konusunda Spring Events ile daha zarif hali görülecek.

## 7. HTTP Interface: Bildirimsel İstemciler

```java
public interface ExchangeRateApi {

    @GetExchange("/v1/rates")
    ExchangeRateResponse getRate(@RequestParam("base") String from,
                                 @RequestParam("target") String to);
}
```

```java
@Configuration
public class HttpClientConfig {

    @Bean
    public ExchangeRateApi exchangeRateApi(RestClient.Builder builder, ExchangeRateProperties properties) {
        RestClient restClient = builder
                .baseUrl(properties.baseUrl())
                .defaultHeader("X-Api-Key", properties.apiKey())
                // ... timeout ayarları
                .build();

        HttpServiceProxyFactory factory = HttpServiceProxyFactory
                .builderFor(RestClientAdapter.create(restClient))
                .build();
        return factory.createClient(ExchangeRateApi.class);
    }
}
```

| Controller (sunucu) | HTTP Interface (istemci) |
|---|---|
| `@GetMapping` | `@GetExchange` |
| "Bu istek gelirse şu metot çalışsın" | "Bu metot çağrılırsa şu isteği at" |

- `createClient` bir **proxy** döner: Metot çağrılınca anotasyonlara bakıp HTTP isteğini oluşturur, `RestClient` ile gönderir, cevabı çevirir.
- Spring Data repository'leriyle **aynı mekanizma.**

> **Proxy listesi:** `@Transactional`, `@Cacheable`, `@PreAuthorize`, Spring Data repository'leri, Hibernate lazy loading, **HTTP Interface istemcileri.**

| Ne Zaman? | Seçim |
|---|---|
| Basit, standart çağrılar | **HTTP Interface** (az kod, okunaklı) |
| `onStatus` gibi ayrıntılı özelleştirme | **RestClient** doğrudan |

İkisi uyumlu: HTTP Interface arkada zaten `RestClient` kullanır.

## 8. Test Etmek

Gerçek servise istek atılmaz: Yavaş, ücretli olabilir, test verisi kirletir, **hata durumları (500, timeout) kurulamaz.**

> Bir sınıfı değil, **bir HTTP sunucusunu** taklit etmek gerekir. Test edilen şey HTTP katmanının kendisi: URL, başlıklar, JSON, hata dönüşümü.

```java
@RestClientTest(ExchangeRateClient.class)
@EnableConfigurationProperties(ExchangeRateProperties.class)
@TestPropertySource(properties = {
        "clients.exchange-rate.base-url=https://api.test",
        "clients.exchange-rate.api-key=test-key"
})
class ExchangeRateClientTest {

    @Autowired private ExchangeRateClient client;
    @Autowired private MockRestServiceServer server;

    @Test
    void getRate_parsesResponse() {
        server.expect(requestTo("https://api.test/v1/rates?base=USD&target=TRY"))
                .andExpect(header("X-Api-Key", "test-key"))
                .andRespond(withSuccess("""
                        {"base":"USD","target":"TRY","rate":34.25,"timestamp":"2026-10-07T07:00:00Z"}
                        """, MediaType.APPLICATION_JSON));

        ExchangeRateResponse response = client.getRate("USD", "TRY");

        assertThat(response.rate()).isEqualByComparingTo("34.25");
    }

    @Test
    void getRate_whenServerError_throwsExternalServiceException() {
        server.expect(requestTo(startsWith("https://api.test/v1/rates")))
                .andRespond(withServerError());

        assertThatThrownBy(() -> client.getRate("USD", "TRY"))
                .isInstanceOf(ExternalServiceException.class);
    }
}
```

| Araç | Görevi |
|---|---|
| `@RestClientTest` | Dilim testi: Sadece istemci ve HTTP altyapısı (`@WebMvcTest`, `@DataJpaTest`'in kardeşi) |
| `MockRestServiceServer` | İstekleri yakalayan sahte sunucu |
| `expect(...)` | Gelmesi gereken isteği doğrular (URL, başlık) → `verify` gibi |
| `andRespond(...)` | Verilecek cevabı belirler → `when().thenReturn()` gibi |

> ⚠️ **Sınırı:** İstekler gerçek ağa gitmediği için **timeout test edilemez.**
> **WireMock:** Testlerde gerçek HTTP sunucusu başlatır, "5 sn gecikmeyle cevap ver" denebilir. Testcontainers'ın veritabanı için yaptığını HTTP servisleri için yapar.

---

## Sorular

**1. Yeni bir Spring MVC projesinde hangi HTTP istemcisi tercih edilir?**
`RestClient`. `RestTemplate` bakım modunda, `WebClient` reaktif dünya için tasarlanmıştır.

**2. `RestClient.Builder` enjekte etmenin faydaları nelerdir?**
Uygulamanın `ObjectMapper`'ı kullanılır, `http.client.requests` metrikleri ve tracing başlıkları otomatik eklenir.

**3. Spring Boot `RestClient.Builder`'ı neden prototype scope ile tanımlar?**
Builder değiştirilebilir; singleton olsaydı bir istemcinin `baseUrl` ve başlıkları diğer istemcilere sızardı.

**4. URI'de neden string birleştirme yerine yer tutucu kullanılır?**
Yer tutucular değerleri doğru kodlar; string birleştirme özel karakterlerde URL'yi bozar ve parametre enjeksiyonuna açıktır.

**5. Anti-corruption layer nedir?**
Dış sistemin modelinin uygulamanın kendi modeline sızmasını önleyen katman. Dış DTO istemci sınıfında kalır, kendi kavramlarımıza çevrilir.

**6. Dış servisin 401'ini kullanıcıya iletmek neden tehlikelidir?**
Ön yüz bunu kullanıcının kimlik sorunu sanıp çıkış yaptırır; oysa sorun uygulamanın API anahtarındadır. Dış hatalar 502/503/504'e çevrilir.

**7. 502, 503 ve 504 farkı nedir?**
502 bağımlı servisin hatalı cevabı, 503 geçici olarak hizmet verilememesi, 504 bağımlı servisin zamanında cevap vermemesidir.

**8. Dış servisin 4xx ve 5xx cevapları neden farklı ele alınır?**
5xx genelde karşı tarafın geçici sorunudur, tekrar deneme işe yarayabilir. 4xx genelde bizim hatamızdır (yanlış istek), tekrar denemek bir şey değiştirmez.

**9. Connect timeout ve read timeout farkı nedir?**
Connect timeout bağlantı kurulamadığında, read timeout bağlantı kurulup cevap gelmediğinde dolar.

**10. Timeout'suz yavaş bir dış servis ilgisiz endpoint'leri nasıl etkiler?**
Bekleyen istekler Tomcat'in sınırlı thread havuzunu doldurur, uygulama yeni istek kabul edemez (cascading failure).

**11. Dış servis çağrısı neden transaction içinde yapılmaz?**
Bağlantı havuzunu gereksiz meşgul eder ve geri alınamaz: Rollback veritabanını geri alır ama ödeme gibi dış yan etkileri geri alamaz.

**12. `@HttpExchange` arayüzünün implementasyonunu kim yazar?**
Spring çalışma zamanında bir proxy üretir; Spring Data repository'leriyle aynı mekanizmadır.

**13. `@RestClientTest` ve `MockRestServiceServer` ne işe yarar?**
Sadece istemciyi yükleyen dilim testi ve istekleri yakalayıp sahte cevap veren sunucu. URL, başlık, JSON ve hata dönüşümleri test edilir.

**14. `MockRestServiceServer` ile timeout neden test edilemez? Alternatif nedir?**
İstekler gerçek ağa gitmez. Gecikmeli cevap verebilen gerçek bir HTTP sunucusu başlatan WireMock kullanılır.
