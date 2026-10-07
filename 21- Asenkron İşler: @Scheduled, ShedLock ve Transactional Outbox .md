# Asenkron İşler - Bölüm 2: @Scheduled, ShedLock ve Transactional Outbox

## 1. Sorun: Hiçbir İsteğin Tetiklemediği İşler

Bazı işleri kullanıcı değil **zaman** tetikler:
- Ödemesi 30 dk'da tamamlanmayan siparişleri iptal et, stoğu geri ver
- Her gece 02:00'de günlük satış raporu gönder
- Süresi dolmuş refresh token'ları temizle
- Başarısız e-postaları belirli aralıklarla tekrar dene

## 2. @Scheduled

```java
@SpringBootApplication
@EnableScheduling
public class ShopApplication { ... }
```

> ⚠️ `@EnableScheduling` unutulursa görevler **hiç çalışmaz**, hata alınmaz.

```java
@Component
public class OrderExpirationJob {

    private final OrderExpirationService expirationService;

    @Scheduled(fixedDelay = 1, timeUnit = TimeUnit.MINUTES)
    public void cancelExpiredOrders() {
        expirationService.cancelExpiredOrders();
    }
}
```

- Metot **parametre almaz, `void` döner.** Spring belirtilen zamanlarda kendisi çağırır.

### Görev Sınıfı İnce Tutulur

Görev sınıfı iş yapmaz, **service'e devreder:**
- İş mantığı (`@Transactional` dahil) service'te kalır.
- Service testte doğrudan çağrılabilir.
- Aynı mantık bir admin endpoint'inden "şimdi çalıştır" ile tetiklenebilir.
- `@Transactional` başka bean'den çağrıldığı için proxy üzerinden **kesin** çalışır.

> Controller HTTP'yi service'e bağlıyordu; görev sınıfı **zamanı** service'e bağlar.

### Zamanlama Seçenekleri

| Seçenek | Anlamı |
|---|---|
| `fixedRate` | Her N sürede bir başlat (önceki çalışmanın **başlangıcından** itibaren) |
| `fixedDelay` | Önceki çalışma **bittikten** N süre sonra başlat |
| `initialDelay` | Uygulama başladıktan sonra ilk çalışma için bekle |
| `cron` | Takvim tabanlı ("her gün 02:00") |

> Görev periyodundan uzun sürerse: `fixedRate` arka arkaya **arasız** çalışır, `fixedDelay` her zaman **ara bırakır.** Veritabanı tarayan görevler için `fixedDelay` daha güvenli.

### Cron İfadeleri

```java
@Scheduled(cron = "0 0 2 * * *", zone = "Europe/Istanbul")
public void sendDailyReport() { ... }
```

```
┌───────────── saniye (0-59)
│ ┌─────────── dakika (0-59)
│ │ ┌───────── saat (0-23)
│ │ │ ┌─────── ayın günü (1-31)
│ │ │ │ ┌───── ay (1-12)
│ │ │ │ │ ┌─── haftanın günü (0-7 veya MON-SUN)
0 0 2 * * *
```

> ⚠️ **Tuzak 1:** Linux cron **5 alanlı** (saniye yok), Spring cron **6 alanlı.** İnternetten bulunan `0 2 * * *` Spring'de yanlış yorumlanır.

> ⚠️ **Tuzak 2: Saat dilimi.** Docker konteynerleri genelde **UTC.** `zone` yazılmazsa 02:00 görevi İstanbul saatiyle **05:00'te** çalışır. Takvime bağlı her görevde `zone` açıkça yazılır.

```java
@Scheduled(cron = "${jobs.daily-report.cron}", zone = "Europe/Istanbul")   // yapılandırmadan
```

### Zamanlayıcının Thread Havuzu

| Havuz | Görevi | Varsayılan |
|---|---|---|
| Tomcat | HTTP istekleri | 200 thread |
| `@Async` | Arka plan işleri | Sınırsız kuyruk |
| **Zamanlayıcı** | `@Scheduled` görevler | ⚠️ **Tek thread** |

> Gece raporu 10 dk sürerse, o sürede "her dakika siparişleri iptal et" görevi **çalışamaz**, sırasını bekler.

```yaml
spring:
  task:
    scheduling:
      pool:
        size: 4
      thread-name-prefix: scheduling-
```

### Hatalar

- Periyodik görevde exception → Spring loglar, görev **bir sonraki zamanında yine çalışır.**
- `GlobalExceptionHandler` yok. Görev kendi hatalarını yakalayıp loglamalı, **metrikle izlenmeli.**
- "Gece raporu üç gündür hata veriyor ama kimse fark etmedi" klasik bir hikâyedir.
- `requestId` yok; her çalışmaya kendi kimliği (`jobRunId`) MDC'ye konabilir.

## 3. Toplu İşleri Doğru Yazmak

```java
@Service
public class OrderExpirationService {

    private final OrderRepository orderRepository;
    private final Clock clock;

    public void cancelExpiredOrders() {
        Instant cutoff = clock.instant().minus(Duration.ofMinutes(30));
        Page<Order> page;
        do {
            page = orderRepository.findByStatusAndCreatedAtBefore(
                    OrderStatus.PENDING_PAYMENT, cutoff, PageRequest.of(0, 100));
            page.forEach(order -> cancelSingle(order.getId()));
        } while (page.hasNext());
    }
}
```

| Karar | Sebep |
|---|---|
| **Sayfa sayfa** (100'erlik) | 50.000 kaydı tek seferde belleğe çekmek uygulamayı çökertir |
| **Hep sıfırıncı sayfa** | İşlenenler durum değiştirip sorgudan çıkar; `page++` kayıt atlatır |
| **Her kayıt ayrı transaction** | Tek transaction'da bir hata **hepsini** geri alır ve bağlantı dakikalarca meşgul kalır |
| **`Clock` enjeksiyonu** | Zaman test edilebilir olur (bölüm 7) |

> ⚠️ Başarısız kayıtlar hep ilk sayfada kalırsa döngü sonsuza dönebilir; gerçek kodda güvenlik önlemi gerekir.

### Yarıda Kalırsa?

Görev 50.000'in 20.000'ini işledi, deploy oldu. **Sorun yok**, çünkü görev **idempotent:**
- Sorgu "durumu `PENDING_PAYMENT` olan" kayıtları seçer.
- İptal edilenler artık bu durumda değil.
- Sonraki çalışmada **kendiliğinden** kaldığı yerden devam eder.

> **Sor:** "Bu görev yarıda kesilip yeniden başlatılırsa ne olur?"
> **Cevap "kaldığı yerden devam eder" olmalı.** Yol: Kayıtları **durum alanıyla** seç, işledikçe durumu değiştir.

## 4. Birden Fazla Sunucu: Aynı Görev Üç Kez

```
Sunucu A: 02:00 → raporu gönder
Sunucu B: 02:00 → raporu gönder
Sunucu C: 02:00 → raporu gönder
```

- Yöneticiler **üç** rapor alır (can sıkıcı).
- "Başarısız ödemeleri tekrar dene" görevi üç kez çalışırsa, idempotency key yoksa **müşteriden üç kez para çekilir.**

> `@Scheduled` her sunucuda bağımsızdır. Caffeine'in diğer önbelleklerden habersiz olmasıyla aynı sorun → **Ortak bir şey üzerinden koordinasyon.**

| Çözüm | Not |
|---|---|
| Görevleri tek sunucuda açmak | O sunucu çökerse görevler durur |
| Platform zamanlayıcısı | Kubernetes CronJob |
| Quartz (küme desteği) | Daha ağır |
| **ShedLock** | En basit ve yaygın |

## 5. ShedLock

**Fikir:** Tüm sunucuların paylaştığı **veritabanında** kilit al. Alan çalıştırır, alamayan o turu **atlar.**

```
02:00  Sunucu A: kilidi aldı → çalışıyor
02:00  Sunucu B: kilit dolu → atladı
02:00  Sunucu C: kilit dolu → atladı
```

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>6.0.2</version>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
    <version>6.0.2</version>
</dependency>
```

```sql
-- V9__create_shedlock_table.sql
CREATE TABLE shedlock (
    name       VARCHAR(64)  NOT NULL PRIMARY KEY,
    lock_until TIMESTAMP    NOT NULL,
    locked_at  TIMESTAMP    NOT NULL,
    locked_by  VARCHAR(255) NOT NULL
);
```

```java
@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class SchedulingConfig {

    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
                JdbcTemplateLockProvider.Configuration.builder()
                        .withJdbcTemplate(new JdbcTemplate(dataSource))
                        .usingDbTime()
                        .build());
    }
}
```

```java
@Scheduled(cron = "0 0 2 * * *", zone = "Europe/Istanbul")
@SchedulerLock(name = "dailyReport", lockAtMostFor = "30m", lockAtLeastFor = "1m")
public void sendDailyReport() {
    reportService.sendDailyReport();
}
```

| Parametre | Çözdüğü Sorun |
|---|---|
| `name` | Kilidin adı; her görev için **benzersiz** |
| **`lockAtMostFor`** | Kilidi alan sunucu **çökerse** kilit sonsuza kadar dolu kalmasın. Süre sonunda kendiliğinden düşer. |
| **`lockAtLeastFor`** | Çok kısa görev + sunucu **saat farkları** → aynı turda ikinci çalışma. Kilit en az bu süre tutulur. |
| **`usingDbTime()`** | Kilit süreleri her sunucunun kendi saatiyle değil, **veritabanının saatiyle** hesaplanır |

> ⚠️ **`lockAtMostFor` görev süresinden açıkça uzun olmalı.** Kısa olursa görev hâlâ çalışırken kilit düşer, başka sunucu aynı görevi başlatır.
> TTL "unutulan evict'e karşı", `lockAtMostFor` "çöken sunucuya karşı" güvenlik ağı.

- ShedLock bir **zamanlayıcı değildir**; zamanlamayı `@Scheduled` yapar, ShedLock "bu tetiklemeyi çalıştırayım mı?" der.
- Kilidi alamayan sunucu görevi **sıraya koymaz, atlar.** "Kaçırılırsa sonraki turda yapılır" türü (idempotent) görevler için ideal.
- `@SchedulerLock` da **proxy (AOP)** ile çalışır.

## 6. Transactional Outbox

### Sorun: Dual Write

Depoya "siparişi hazırla" bildirimi **kesinlikle kaybolmamalı.**

| Yaklaşım | Açığı |
|---|---|
| Transaction içinde API çağrısı | Rollback olursa bildirim gitmiş olur; yavaşsa bağlantı havuzu dolar |
| `@TransactionalEventListener` (senkron) | Commit sonrası, gönderim öncesi çökme → **kayıp** |
| `@Async` + `@TransactionalEventListener` | Kuyruk bellekte; deploy / çökme → **kayıp** |

> **Dual write:** İki farklı sisteme (veritabanı + API) bölünemez şekilde yazmanın yolu yok. Transaction sadece veritabanını kapsar; arada her zaman bir çökme boşluğu vardır.

### Çözüm: Her Şeyi Tek Sisteme Yaz

"Bildirim gönderilmeli" bilgisini de veritabanına, siparişle **aynı transaction'da** yaz.

```sql
-- V10__create_outbox_table.sql
CREATE TABLE outbox_events (
    id            BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    event_type    VARCHAR(100) NOT NULL,
    payload       TEXT         NOT NULL,
    status        VARCHAR(20)  NOT NULL,
    attempts      INTEGER      NOT NULL DEFAULT 0,
    created_at    TIMESTAMP    NOT NULL,
    processed_at  TIMESTAMP
);

CREATE INDEX idx_outbox_status_created ON outbox_events (status, created_at);
```

```java
@Transactional
public OrderResponse placeOrder(OrderRequest request) {
    Order order = orderRepository.save(new Order(...));

    outboxRepository.save(OutboxEvent.pending(
            "WAREHOUSE_PREPARE_ORDER",
            toJson(new WarehousePreparePayload(order.getId(), order.getItems()))));

    return toResponse(order);
}
```

- Sipariş ve outbox kaydı **aynı transaction'da** → Ya ikisi birden kaydedilir ya ikisi de geri alınır.
- "Sipariş var ama bildirim bilgisi yok" durumu **imkânsız.** Uygulama bir ms sonra çökse bile bilgi veritabanında güvende.
- **Outbox = giden kutusu.** Mektuplar önce kutuya konur, postacı düzenli olarak gelip alır.
- Index: Postacı sürekli "bekleyen en eski kayıtlar" sorgusunu çalıştıracak.

### Postacı

```java
@Component
public class OutboxRelayJob {

    private final OutboxProcessor processor;

    @Scheduled(fixedDelay = 5, timeUnit = TimeUnit.SECONDS)
    @SchedulerLock(name = "outboxRelay", lockAtMostFor = "2m")
    public void relay() {
        processor.processPendingEvents();
    }
}
```

```java
@Service
public class OutboxProcessor {

    public void processPendingEvents() {
        List<OutboxEvent> batch = outboxRepository
                .findTop50ByStatusOrderByCreatedAtAsc(OutboxStatus.PENDING);

        for (OutboxEvent event : batch) {
            try {
                handlers.get(event.getEventType()).handle(event);    // dış çağrı, transaction DIŞINDA
                outboxStatusService.markSent(event.getId());         // kısa transaction
            } catch (Exception ex) {
                log.warn("Outbox olayı işlenemedi: id={}", event.getId(), ex);
                outboxStatusService.markFailedAttempt(event.getId()); // attempts++
            }
        }
    }
}
```

| Parça | Kullanılan Konu |
|---|---|
| Her 5 sn'de kontrol | `@Scheduled` |
| Üç sunucu aynı olayı göndermesin | `@SchedulerLock` |
| Sınırlı sayıda işle (`Top50`) | Sayfalı okuma |
| Dış çağrı transaction dışında, durum güncellemesi kısa transaction'da | "Dış servisi transaction içinde çağırma" kuralı |
| Başarısız → `PENDING` kalır, `attempts++`, sonraki turda tekrar | **Retry** (uygulama yeniden başlasa bile kaybolmayan) |

**Gerçek sistemde eklenenler:**
- Denemeler arası bekleme (backoff, örn. `next_attempt_at` kolonu)
- N denemeden sonra `FAILED` durumu + uyarı (sonsuza kadar deneme yerine birinin bakması)
- Gönderilmiş eski kayıtları temizleyen başka bir zamanlanmış görev

### En Az Bir Kez Teslim

```
Postacı gönderdi → depo aldı → markSent'ten ÖNCE çöktü → kayıt hâlâ PENDING → TEKRAR gönderilir
```

| Garanti | Anlamı | Risk |
|---|---|---|
| **En fazla bir kez** (at-most-once) | Gönder ve unut | Mesaj **kaybolabilir** (`@Async`) |
| **En az bir kez** (at-least-once) | Onaylanana kadar tekrar dene | Mesaj **tekrar** gelebilir (outbox) |
| **Tam olarak bir kez** (exactly-once) | Ne kayıp ne tekrar | İki sistem arasında **pratikte sağlanamaz** |

> "Gönderdim" ile "gönderdiğimi kaydettim" arasında her zaman bir çökme olabilir.

**Tekrarlar nasıl zararsız olur?** **Idempotency:** Postacı outbox kaydının kimliğini **idempotency key** olarak gönderir; alıcı aynı anahtarı ikinci kez görünce işlemi tekrarlamaz. Kendi uygulaman mesaj alıyorsa aynı sorumluluk sende.

> **Formül:** En az bir kez teslim + idempotent alıcı = **Etkili olarak tam bir kez.**

### Outbox'ın Gittiği Yer

- Büyük sistemlerde postacı olayları **mesaj kuyruğuna** (RabbitMQ, Kafka) yayınlar. "Veritabanına yaz + kuyruğa gönder" de bir dual write sorunudur, outbox orada da kullanılır.
- **CDC (Change Data Capture):** Debezium gibi araçlar outbox tablosunu sorgulamak yerine veritabanının değişiklik günlüğünü okur. Büyük ölçekte tercih edilir.

## 7. Test Etmek

### Zamanlayıcıyı Bekleme

İş service'te olduğu için **service doğrudan çağrılır:**

```java
@Test
void cancelExpiredOrders_cancelsOnlyOrdersOlderThan30Minutes() {
    // test verisi hazırla
    expirationService.cancelExpiredOrders();
    // sonucu doğrula
}
```

### Testlerde Zamanlayıcıyı Kapatmak

Testin ortasında postacı devreye girip test verisini değiştirmesin:

```java
@Configuration
@EnableScheduling
@ConditionalOnProperty(name = "app.scheduling.enabled", havingValue = "true", matchIfMissing = true)
public class SchedulingEnablerConfig {
}
```

```yaml
# application-test.yml
app:
  scheduling:
    enabled: false
```

- `@ConditionalOnProperty`: Sınıfı sadece özellik `true` ise yükle. Auto-configuration'ın temel taşlarından biri.

### Zamanı Kontrol Etmek: Clock

`Instant.now()` testte kontrol edilemez: 30 dk beklemek gerekir veya tarihler karmaşık hesaplanır; dakika değişirken test kırılabilir.

```java
@Bean
public Clock clock() {
    return Clock.systemDefaultZone();   // production: gerçek saat
}
```

```java
// Test: zaman sabit
Clock fixedClock = Clock.fixed(Instant.parse("2026-10-07T12:00:00Z"), ZoneOffset.UTC);
OrderExpirationService service = new OrderExpirationService(orderRepository, fixedClock);
```

- 11:30 siparişi = tam 30 dk, 11:31 = 29 dk. **Sınır durumları kesin test edilir.**

> **Zaman bile bir bağımlılıktır.** `new EmailService()` yerine constructor'dan almak ile `Instant.now()` yerine `Clock` almak **aynı ilkedir** (IoC / DI).

---

## Sorular

**1. `fixedRate` ve `fixedDelay` farkı nedir?**
`fixedRate` önceki çalışmanın başlangıcından, `fixedDelay` bitişinden itibaren sayar. Uzun görevlerde `fixedRate` arasız çalışır, `fixedDelay` her zaman ara bırakır.

**2. Linux cron ifadesi Spring'de neden çalışmayabilir?**
Linux cron 5 alanlı, Spring cron başta saniye alanıyla 6 alanlıdır.

**3. 02:00 görevi production'da 05:00'te çalışıyorsa sebep ne olabilir?**
Konteyner UTC'de çalışıyordur ve `zone` belirtilmemiştir.

**4. `@Scheduled` görevlerin varsayılan thread havuzu nedir?**
Tek thread. Uzun süren görev diğerlerini bekletir; `spring.task.scheduling.pool.size` ile büyütülür.

**5. Zamanlanmış görev sınıfı neden ince tutulur?**
İş mantığı service'te kalır; doğrudan test edilebilir, başka yerden tetiklenebilir ve `@Transactional` proxy üzerinden kesin çalışır.

**6. Toplu görev neden tek transaction'da yapılmaz?**
Bir hata tüm işi geri alır ve bağlantı uzun süre meşgul kalır. Her kayıt veya parça ayrı transaction'da işlenir.

**7. Toplu görev yarıda kesilirse ne olmalı?**
Kaldığı yerden devam etmeli. Kayıtları durum alanıyla seçip işledikçe durumu değiştirmek görevi idempotent yapar.

**8. Birden fazla sunucuda `@Scheduled` hangi sorunu yaratır?**
Her sunucu görevi bağımsız çalıştırır; görev sunucu sayısı kadar tekrarlanır.

**9. ShedLock nasıl çalışır?**
Görevi çalıştırmadan önce veritabanında kilit alır; kilidi alan sunucu çalıştırır, diğerleri o turu atlar.

**10. `lockAtMostFor` ve `lockAtLeastFor` ne işe yarar?**
`lockAtMostFor` kilidi alan sunucu çökerse kilidin kendiliğinden düşmesini sağlar (görev süresinden uzun olmalı). `lockAtLeastFor` saat farkları yüzünden kısa görevlerin aynı turda tekrar çalışmasını önler.

**11. Dual write sorunu nedir?**
Veritabanına ve başka bir sisteme bölünemez şekilde yazılamamasıdır; aradaki çökme tutarsızlık yaratır.

**12. Transactional outbox nasıl çalışır?**
Gönderilecek mesaj, asıl veriyle aynı transaction'da outbox tablosuna yazılır. Zamanlanmış bir görev bekleyen kayıtları okuyup gönderir ve durumunu günceller.

**13. Outbox hangi teslim garantisini sağlar?**
En az bir kez. Mesaj kaybolmaz ama tekrar gelebilir; alıcı idempotent olmalıdır.

**14. Neden "tam olarak bir kez" teslim pratikte sağlanamaz?**
"Gönderdim" ile "gönderdiğimi kaydettim" arasında her zaman bir çökme olabilir. Pratik çözüm: en az bir kez + idempotent alıcı.

**15. Testlerde zamanlanmış görevler nasıl kapatılır?**
`@EnableScheduling` bir `@ConditionalOnProperty` yapılandırma sınıfına konur ve test profilinde özellik `false` yapılır.

**16. Zamana bağlı kod neden `Clock` ile yazılır?**
Testte `Clock.fixed(...)` ile zaman sabitlenir ve sınır durumları kesin test edilir. Zaman da bir bağımlılıktır.
