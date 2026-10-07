# Dış Servis Çağırma - Bölüm 2: Retry, Circuit Breaker ve Fallback

## 1. Her Hata Aynı Değildir

| Hata Türü | Örnek | Süre | Doğru Tepki |
|---|---|---|---|
| **Geçici** (transient) | Anlık ağ kopukluğu, tek timeout, yeniden başlarken 503 | ms / birkaç sn | **Tekrar dene** |
| **Süren** | Servis çöktü, bakımda, aşırı yük | Dakikalar | **Denemeyi bırak, bekle** |
| **Kalıcı** | Yanlış format, geçersiz anahtar (4xx) | Kod düzeltilene kadar | **Hiç tekrar deneme** |

> **Resilience (dayanıklılık):** Retry, circuit breaker, fallback gibi desenlerin genel adı.

## 2. Kurulum: Resilience4j

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

> ⚠️ **AOP bağımlılığı eksikse** anotasyonlar derlenir, uygulama çalışır, **hiçbir şey olmaz.** Hata alınmaz. (Anotasyonlar **proxy** ile çalışır.)

> Spring Framework 7 / Boot 4 ile çekirdeğe basit `@Retryable` geldi; Spring Retry projesi de var. Kavramlar her araçta aynı.

### Hata Sınıflarını Ayırmak

Retry, hangi hatanın geçici olduğunu bilmek zorundadır:

```java
.retrieve()
.onStatus(status -> status.value() == 404, (req, res) -> {
    throw new BusinessException("Desteklenmeyen döviz çifti");
})
.onStatus(HttpStatusCode::is4xxClientError, (req, res) -> {
    throw new ExternalClientException("Döviz servisine hatalı istek: " + res.getStatusCode());
})
.onStatus(HttpStatusCode::is5xxServerError, (req, res) -> {
    throw new ExternalServiceException("Döviz servisi hata döndü: " + res.getStatusCode());
})
```

| Exception | Anlamı | Retry? |
|---|---|---|
| `BusinessException` | **Kullanıcının** hatası | ❌ |
| `ExternalClientException` | **Bizim** hatamız (yanlış istek) | ❌ |
| `ExternalServiceException` | **Karşı tarafın** geçici sorunu | ✅ |
| `ResourceAccessException` | Timeout / bağlantı hatası | ✅ |

## 3. Retry

```yaml
resilience4j:
  retry:
    instances:
      exchangeRate:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - com.example.shop.exception.ExternalServiceException
          - org.springframework.web.client.ResourceAccessException
        ignore-exceptions:
          - com.example.shop.exception.BusinessException
          - com.example.shop.exception.ExternalClientException
```

```java
@Retry(name = "exchangeRate")
public ExchangeRateResponse getRate(String from, String to) { ... }
```

| Ayar | Anlamı |
|---|---|
| `name` | Yapılandırmaya bağlar. **Her dış servis için ayrı instance.** |
| `max-attempts: 3` | Toplam deneme: ilk + 2 tekrar |
| `retry-exceptions` | Tekrar denenecek hatalar |
| `ignore-exceptions` | **Asla** tekrar denenmeyecek hatalar |

### Backoff ve Jitter

| Kavram | Açıklama |
|---|---|
| **Wait duration** | Aynı milisaniyede tekrar denemek aynı sonucu verir ve zorlanan servise yük ekler |
| **Exponential backoff** | Bekleme katlanarak artar: 500 ms → 1 sn → 2 sn. Sorun uzadıkça servise daha çok nefes alma süresi. |
| **Jitter** | Bekleme sürelerine **rastgele fark.** 100 istek aynı anda hata alıp aynı anlarda tekrar denerse servis dalgalar halinde boğulur. |

> Jitter, caching'deki "TTL'lere rastgele fark ekle" fikrinin aynısı.

### ⚠️ Retry'ın Tehlikeleri

**Toplam süre:**
```
3 sn timeout × 3 deneme + beklemeler ≈ 10 sn
```
- Bu sürede Tomcat thread'i meşgul kalır, thread havuzu tükenmesi kötüleşir.
- **Sor:** "Kullanıcı bu cevap için toplamda en fazla ne kadar bekleyebilir?"

**Retry fırtınası:**
```
A → B → C (çökmüş), her katman 3 deneme
1 istek → C'ye 3 × 3 = 9 istek (üç katmanda 27)
```
- Toparlanmaya çalışan servis katlanarak artan yükle karşılaşır.
- **Kural:** Yeniden denemeyi zincirin **tek bir katmanında** yap (genelde dış servise en yakın olanında).

## 4. Idempotency

### ⚠️ Senaryo

```
1. "Karttan 1.500 TL çek" → 3 sn içinde cevap yok → TIMEOUT
2. Retry → tekrar gönder → başarılı
```

**Timeout ne söyler?** "Cevabı zamanında alamadım."
**Ne söylemez?** "İstek karşıya ulaşmadı."

İlk istek işlenmiş, cevap dönerken ağ yavaşlamış olabilir → **Ödeme iki kez alındı, 3.000 TL çekildi.**

### Idempotent İşlem

Bir kez yapmakla birden fazla kez yapmak **aynı sonucu** veren işlem.

| Metot | Idempotent? | Neden? |
|---|---|---|
| `GET` | ✅ | Sadece okur |
| `PUT` | ✅ | "Fiyatı 100 yap": iki kez yapınca da 100 |
| `DELETE` | ✅ | İkinci silme bir şey değiştirmez |
| `POST` | ❌ | "Sipariş oluştur": iki kez = iki sipariş |

> ⚠️ **Idempotent olmayan işlemi körü körüne tekrar denemek para kaybettiren bir hatadır.**

### Idempotency Key

```java
paymentClient.post()
        .uri("/v1/charges")
        .header("Idempotency-Key", order.getPaymentReference())
        .body(chargeRequest)
        .retrieve()
        .body(ChargeResponse.class);
```

1. İstemci benzersiz bir anahtar ekler.
2. Sunucu anahtarı kaydeder.
3. Aynı anahtarla gelen sonraki istekte işlemi **tekrarlamaz, ilk sonucu döner.**

> ⚠️ Anahtar **işlemin kendisine bağlı** olmalı (örn. sipariş kimliğinden türetilmiş). Her denemede yeni anahtar üretilirse mekanizma işe yaramaz.

**Kendi API'n için de geçerli:** Kullanıcı "Sipariş Ver"e iki kez basarsa veya mobil uygulama isteği otomatik tekrarlarsa `POST /api/orders` iki sipariş oluşturur.

> **Kural:** Tekrar denemeden önce sor: **"Bu işlem iki kez gerçekleşirse ne olur?"** Cevap "hiçbir şey" değilse, idempotency key kullan ya da hiç tekrar deneme.

## 5. Circuit Breaker

**Sorun:** Servis 10 dakika çökük. Her istek: gönder → 3 sn bekle → tekrar → 3 sn bekle... ≈ 10 sn sonra yine hata.
- Thread havuzu dolar, cevap süreleri her yerde uzar.
- Ayağa kalkmaya çalışan servis bekleyen isteklerle boğulur.

**Çözüm: Fail fast.** "Bu servis şu an çalışmıyor, bir süre istek göndermeyeceğim, hemen hata döneceğim."

> **Benzetme:** Elektrik sigortası. Kısa devrede sigorta atar, arıza tüm sisteme yayılmaz.

### Üç Durum

```
              başarısızlık oranı eşiği aştı
    ┌────────┐ ──────────────────────────► ┌────────┐
    │ CLOSED │                             │  OPEN  │
    │(normal)│ ◄──┐                        │(kesik) │
    └────────┘    │                        └────────┘
                  │ deneme çağrıları           │
                  │ başarılı                   │ bekleme süresi doldu
                  │                            ▼
                  │                       ┌───────────┐
                  └────────────────────── │ HALF_OPEN │
                                          │ (deneme)  │
                  deneme çağrıları ──────►└───────────┘
                  başarısız: tekrar OPEN
```

| Durum | Davranış |
|---|---|
| **CLOSED** | Normal. İstekler gider, sonuçlar sayılır. |
| **OPEN** | İstekler **hiç gönderilmez**, anında `CallNotPermittedException`. Thread'ler serbest, servis rahat bırakılır. |
| **HALF_OPEN** | Bekleme süresi doldu, birkaç **deneme** isteği geçer. Başarılı → CLOSED, başarısız → OPEN. |

> ⚠️ Elektrikte kapalı devre = akım geçer. **CLOSED iyidir, OPEN sorun var demektir.**

### Yapılandırma

```yaml
resilience4j:
  circuitbreaker:
    instances:
      exchangeRate:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 20
        minimum-number-of-calls: 10
        failure-rate-threshold: 50
        slow-call-duration-threshold: 2s
        slow-call-rate-threshold: 80
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        ignore-exceptions:
          - com.example.shop.exception.BusinessException
```

| Ayar | Anlamı |
|---|---|
| `sliding-window-size: 20` | Son 20 çağrıya bakılır, eskiler hesaba katılmaz |
| `failure-rate-threshold: 50` | %50 veya fazlası başarısızsa devre açılır |
| `minimum-number-of-calls: 10` | En az 10 çağrı olmadan oran hesaplanmaz |
| `slow-call-*` | 2 sn'den uzun çağrılar "yavaş"; %80'i yavaşsa **hata olmasa bile** devre açılır |
| `wait-duration-in-open-state: 30s` | OPEN'da HALF_OPEN'a geçmeden önce bekleme |
| `ignore-exceptions` | Başarısızlık sayılmayacak hatalar |

> **`minimum-number-of-calls` neden?** İlk 2 çağrıdan biri başarısız = %50. Bu ayar olmasa devre gereksiz yere açılırdı.

> **Yavaş çağrılar neden?** Her çağrıya 2,9 sn'de cevap veren servis 3 sn timeout'a takılmaz ama uygulamayı yavaşlatır. **Thread havuzunu tüketen sadece hata değil, yavaşlıktır.**

> ⚠️ **`BusinessException` neden yok sayılır?** Kullanıcıların hatalı istekleri servisin çöktüğü anlamına gelmez. Sayılırsa **sağlıklı servisin devresi açılır**, herkes için özellik durur.

### Her Bağımlılık İçin Ayrı Devre

Döviz ve ödeme servisi için **ayrı** circuit breaker'lar. Döviz çökünce ödeme almaya devam edilebilmeli. (Evdeki her oda için ayrı sigorta.)

### Bulkhead (Bölme)

> **Benzetme:** Gemideki su geçirmez bölmeler. Bir bölmeye su dolsa diğerleri kuru kalır, gemi batmaz.

```yaml
resilience4j:
  bulkhead:
    instances:
      exchangeRate:
        max-concurrent-calls: 20
```

- Dış servise **aynı anda** en fazla 20 çağrı. 21. beklemeden reddedilir.
- Yavaşlayan döviz servisi 200 thread'in hepsini değil, en fazla 20'sini meşgul eder; **180 thread** diğer endpoint'lere kalır.

## 6. Fallback

```java
@CircuitBreaker(name = "exchangeRate", fallbackMethod = "getRateFallback")
@Retry(name = "exchangeRate")
public ExchangeRateResponse getRate(String from, String to) {
    ExchangeRateResponse response = // ... RestClient çağrısı
    lastKnownRates.save(from, to, response);   // başarılı sonucu sakla
    return response;
}

private ExchangeRateResponse getRateFallback(String from, String to, Throwable ex) {
    log.warn("Döviz servisi kullanılamıyor, son bilinen kur kullanılıyor: {}/{}", from, to, ex);
    return lastKnownRates.find(from, to)
            .orElseThrow(() -> new ExternalServiceException("Döviz kuru şu an alınamıyor"));
}
```

- Fallback metodu: **Aynı parametreler + sonda `Throwable`**, **aynı dönüş tipi.**
- Hata ne olursa olsun (timeout, 503, devre açık) çağrılır.
- **Son bilinen değer** Redis'te saklanabilir; TTL normal önbellekten **uzun** (örn. 24 saat), çünkü bu bir hızlandırıcı değil, **yedek.**

> Eski veri dönmek burada **bilinçli bir karar.** Birkaç saat eski kur, hiç kur olmamasından iyidir.

### Fallback Stratejileri

| Strateji | Ne Zaman? | Örnek |
|---|---|---|
| **Son bilinen değer** | Biraz eski veri kabul edilebilirse | Döviz kuru, hava durumu |
| **Varsayılan değer** | Makul varsayılan varsa | Öneri servisi çöktü → en çok satanlar |
| **Özelliği kapatmak** | Özellik isteğe bağlıysa | Döviz çevirme yok → sadece TL |
| **Sonra yapmak** | Beklemeye alınabilirse | E-posta → kuyruğa koy |
| **Açıkça hata dönmek** | Hiçbiri uygun değilse | Ödeme → "şu an alınamıyor" |

> ⚠️ **Fallback asla yalan söylememeli.** Ödeme servisi çökünce "ödeme başarılı" dönmek = ürünleri bedavaya göndermek.

> **Fallback seçimi teknik değil, iş kararıdır.** Ürün sahibiyle birlikte karar verilir.

### Anotasyon Sırası

```
Retry ( CircuitBreaker ( Bulkhead ( asıl metot ) ) )
```

- Her yeniden deneme circuit breaker'dan geçer ve sayaca yansır.
- Devre açıksa retry denemeleri anında `CallNotPermittedException` alır, boşuna beklenmez.

> ⚠️ **Proxy ile çalışır:** Aynı sınıftan çağrılırsa retry, circuit breaker ve fallback'in **hiçbiri** çalışmaz.
> Dış çağrıları **kendi istemci sınıflarına** koymak iyi alışkanlık; diğer sınıflar onları her zaman proxy üzerinden çağırır.

## 7. Gözlemlemek

```
GET /actuator/metrics/resilience4j.circuitbreaker.state?tag=name:exchangeRate
GET /actuator/metrics/resilience4j.retry.calls?tag=name:exchangeRate
```

- Circuit breaker'ın **açılması** = bir bağımlılık çöktü. Grafana'da panel ve **"devre açıldığında bildir"** uyarısı.
- Retry metrikleri: "İlk denemede başarılı" ile "tekrar deneyerek başarılı" oranı, bir servisin **bozulmaya başladığının ilk işaretidir.**

> ⚠️ Circuit breaker durumu health'e eklenebilir (`register-health-indicator: true`), ama **readiness grubuna girmemeli.** Fallback ile çalışabilen uygulama, bir bağımlılık çöktü diye trafikten çıkarılmamalı. (Redis'teki dersin aynısı.)

## 8. Test Etmek

- **WireMock:** "İlk iki isteğe 503, üçüncüsüne 200" → retry'ı test et. "5 sn gecikme" → timeout ve circuit breaker'ı test et.

```java
@SpringBootTest
class ExchangeRateClientResilienceTest {

    @Autowired private ExchangeRateClient client;              // Spring'den: proxy devrede
    @Autowired private CircuitBreakerRegistry circuitBreakerRegistry;
    @MockitoBean private LastKnownRateStore lastKnownRates;

    @Test
    void getRate_whenCircuitOpen_returnsLastKnownRate() {
        circuitBreakerRegistry.circuitBreaker("exchangeRate").transitionToOpenState();
        when(lastKnownRates.find("USD", "TRY"))
                .thenReturn(Optional.of(new ExchangeRateResponse("USD", "TRY",
                        new BigDecimal("34.10"), Instant.now())));

        ExchangeRateResponse response = client.getRate("USD", "TRY");

        assertThat(response.rate()).isEqualByComparingTo("34.10");
    }
}
```

- `CircuitBreakerRegistry`: Tüm circuit breaker'ları tutan bean. Devre elle açılır, dış servise istek atılmaz, fallback doğrudan çalışır.

## 9. Büyük Resim: Katmanlı Savunma

| Mekanizma | Çözdüğü Sorun | Tek Cümleyle |
|---|---|---|
| **Timeout** | Sonsuz bekleme | "En fazla bu kadar beklerim." |
| **Retry** | Geçici hatalar | "Bir kez daha deneyeyim." |
| **Circuit breaker** | Süren hatalar | "Bu servis çökmüş, bir süre rahat bırakayım." |
| **Bulkhead** | Bir bağımlılığın her şeyi tüketmesi | "Bu servise en fazla şu kadar kaynak ayırırım." |
| **Fallback** | Servis yokken ne yapılacağı | "Olmadığında bunu yaparım." |

> Hepsinin altındaki fikir: **Graceful degradation.**
> Her seviyede sorulacak soru: **"Bu bağımlılık çökerse, kullanıcı ne görmeli?"**

---

## Sorular

**1. Hangi hatalarda retry yapılır, hangilerinde yapılmaz?**
Geçici hatalarda (5xx, timeout, bağlantı hatası) yapılır. 4xx hatalarında yapılmaz; istek değişmedikçe aynı hata alınır.

**2. Exponential backoff ve jitter nedir?**
Exponential backoff denemeler arası beklemeyi katlayarak artırır. Jitter beklemelere rastgelelik ekler, çok sayıda isteğin aynı anlarda tekrar deneyip servisi dalgalar halinde boğmasını önler.

**3. Retry fırtınası nedir, nasıl önlenir?**
Zincirdeki her katmanın tekrar denemesiyle tek isteğin katlanarak çoğalmasıdır (3 katman × 3 deneme = 27). Retry zincirin tek bir katmanında yapılmalıdır.

**4. Idempotency nedir? Hangi HTTP metotları idempotent'tir?**
Bir işlemi bir veya birden fazla kez yapmanın aynı sonucu vermesidir. GET, PUT, DELETE idempotent; POST değildir.

**5. Timeout'a düşen ödeme isteğini tekrar denemenin riski nedir?**
Timeout isteğin karşıya ulaşmadığını garanti etmez; ilk istek işlenmiş olabilir. Retry ödemeyi ikinci kez alabilir.

**6. Idempotency key nasıl çalışır?**
İstemci işleme bağlı benzersiz bir anahtar ekler; sunucu aynı anahtarla gelen tekrar isteklerinde işlemi yapmadan ilk sonucu döner.

**7. Circuit breaker'ın üç durumu nedir?**
CLOSED (normal, sonuçlar sayılır), OPEN (istekler gönderilmez, anında hata), HALF_OPEN (birkaç deneme isteği; başarılıysa CLOSED, değilse OPEN).

**8. `minimum-number-of-calls` neden gereklidir?**
Az sayıda çağrıyla hesaplanan oran yanıltıcıdır; devrenin gereksiz yere açılmasını önler.

**9. Circuit breaker neden yavaş çağrıları da sayar?**
Hata vermeyen ama yavaş bir servis de thread havuzunu tüketir.

**10. Kullanıcı kaynaklı `BusinessException` circuit breaker'da neden yok sayılır?**
Servisin çöktüğünü göstermez; sayılırsa sağlıklı servisin devresi açılır ve herkes etkilenir.

**11. Bulkhead hangi sorunu çözer?**
Bir dış servise eşzamanlı çağrı sayısını sınırlar; yavaş bir servis tüm thread havuzunu ele geçiremez.

**12. Fallback stratejileri nelerdir?**
Son bilinen değer, varsayılan değer, özelliği kapatmak, işlemi sonraya bırakmak veya açıkça hata dönmek.

**13. Ödeme için "başarılı" dönen fallback neden yanlıştır?**
Fallback yalan söylememelidir; alınmamış ödemeyi başarılı göstermek ürünlerin bedavaya gönderilmesine yol açar.

**14. Resilience4j anotasyonları hangi durumlarda sessizce çalışmaz?**
AOP bağımlılığı eksikse ve metot aynı sınıfın içinden çağrılıyorsa (self-invocation).

**15. Retry ve circuit breaker birlikte kullanıldığında hangisi önce çalışır?**
Varsayılan sırada retry en dışta, circuit breaker içtedir. Her deneme devre sayacına yansır; devre açıksa denemeler anında reddedilir.

**16. Açık circuit breaker readiness'ı etkilemeli mi?**
Hayır. Fallback ile çalışabilen uygulama trafikten çıkarılmamalıdır.
