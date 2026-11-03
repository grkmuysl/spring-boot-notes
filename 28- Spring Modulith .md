# Spring Modulith: Modüler Monolit

## 0. Modüler Monolit Nedir?

Mikroservisleri konuşurken iki uç gördük: **tek parça monolit** ve **dağıtık mikroservisler.** Arada üçüncü bir yol var: **Modüler monolit.**

| Mimari | Deploy | Modül Sınırları | Karmaşıklık |
|---|---|---|---|
| **Klasik monolit** | Tek parça | Yok ya da sadece kâğıt üstünde | Düşük (başta), zamanla "big ball of mud" |
| **Modüler monolit** | Tek parça | **Net ve kontrol edilen** | Düşük-orta |
| **Mikroservis** | Her servis ayrı | Ağ ile zorlanır | Yüksek (ağ, dağıtık transaction, izleme...) |

> **Modüler monolit:** Tek bir uygulama olarak deploy edilir ama içinde **birbirine gevşek bağlı, sınırları belli modüller** vardır. Mikroservislerin **modülerliğini** dağıtık sistemin **maliyeti olmadan** almak.

### Klasik Monolitin Sorunu

```
order/OrderService ──► inventory/internal/StockRepository   (iç detaya erişim)
inventory/StockService ──► order/OrderService                (döngü!)
payment/PaymentService ──► order/Order entity'sinin alanları (sıkı bağ)
```

- Başta herkes "paketleri ayırdık, modüler" der. Ama Java'da `public` olan her şey **her yerden** çağrılabilir.
- Zamanla herkes herkese bağlanır → Bir yerde değişiklik beş yeri bozar.
- Mikroservise geçmek istendiğinde nereden keseceğini bilemezsin → **Dağıtık monolit** riski.

### Spring Modulith Nedir?

- Spring Boot uygulamalarında **modüler monolit** kurmayı kolaylaştıran **resmi Spring projesi.**
- Modül sınırlarını **paket yapısından** çıkarır, **testle doğrular**, modüller arası iletişimi **event'lerle** kurar.
- Arkada sınır kontrolleri için **ArchUnit** kullanır.

| Spring Modulith Ne Sağlar? | Bölüm |
|---|---|
| Modül tanımı ve sınırlar | 1 |
| Sınırların testle doğrulanması | 2 |
| Event tabanlı modül iletişimi | 3 |
| Kalıcı event kaydı (Event Publication Registry) | 4 |
| Event'leri dışarıya (Kafka vb.) göndermek | 5 |
| Modülleri izole test etmek | 6 |
| Otomatik dokümantasyon ve gözlemlenebilirlik | 7 |

### Sürüm Mantığı

| Spring Modulith | Spring Boot |
|---|---|
| 1.3.x | 3.4.x |
| 1.4.x | 3.5.x |
| **2.0.x** | **4.0.x** |

- **2.0** (Kasım 2025): Spring Boot 4 ve Framework 7 tabanlı. En büyük değişiklik **yenilenen Event Publication Registry** (bölüm 4).

> Spring Cloud'daki release train mantığına benzer: Spring Boot sürümü yükseltilince Modulith sürümü de yükseltilmeli.

### Bağımlılıkları Eklemek: BOM

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-bom</artifactId>
            <version>2.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Sürüm YAZILMAZ, BOM yönetir -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-starter-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-starter-jpa</artifactId>   <!-- Event kaydı için -->
    </dependency>
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

> Spring Cloud BOM'unun aynı mantığı: Uyumlu sürümler tek yerde.

---

## 1. Uygulama Modülleri

### Kural: Ana Paketin Doğrudan Alt Paketleri = Modül

```
com.example.shop
├── ShopApplication.java          ← @SpringBootApplication burada
│
├── order/                        ← MODÜL: order
│   ├── OrderService.java         ← public API (dışarıya açık)
│   ├── OrderCompleted.java       ← public API (event)
│   └── internal/                 ← İÇ DETAY (diğer modüller erişemez)
│       ├── OrderRepository.java
│       └── OrderEntity.java
│
├── inventory/                    ← MODÜL: inventory
│   ├── InventoryService.java
│   └── internal/
│
└── payment/                      ← MODÜL: payment
    └── ...
```

| Kısım | Erişim |
|---|---|
| Modülün **ana paketi** (`order`) | Diğer modüller kullanabilir → **API** |
| Modülün **alt paketleri** (`order.internal`) | Sadece modülün kendisi → **İç detay** |

- Kod yazmadan, sadece **paket yapısıyla** sınır tanımlanır.
- Java'nın `public` / package-private görünürlüğü tek başına yetmez: Farklı paketteki sınıfı kullanmak için `public` yapmak zorunda kalırsın. Modulith bu `public` sınıfların **modül dışından** kullanılmasını yakalar.

> **İpucu:** Mümkün olan her şeyi **package-private** yap. API'de sadece gerçekten dışarıya açık olanlar `public` olsun.

### Named Interface: İç Paketi Kısmen Açmak

Bazen bir alt paketin **bir kısmı** dışarıya açılmak istenir:

```java
// order/spi/package-info.java
@org.springframework.modulith.NamedInterface("spi")
package com.example.shop.order.spi;
```

- `order.spi` artık diğer modüllerin kullanabileceği **ikinci bir API** olur.
- Geri kalan alt paketler yine kapalı kalır.

### İzin Verilen Bağımlılıkları Sınırlamak

```java
// order/package-info.java
@org.springframework.modulith.ApplicationModule(
    displayName = "Siparişler",
    allowedDependencies = { "inventory", "customer :: api" }
)
package com.example.shop.order;
```

- `order` modülü **sadece** `inventory`'ye ve `customer`'ın `api` adlı named interface'ine bağımlı olabilir.
- `payment`'i çağırırsa doğrulama testi kırılır.

### ⚠️ Modüller Arası Veri: Entity Değil ID

```java
// ❌ Yanlış: order modülü inventory'nin entity'sine bağlanıyor
@ManyToOne
private ProductEntity product;

// ✅ Doğru: Sadece kimlik tutuluyor
private Long productId;
```

- Modüller birbirinin **tablolarına JPA ilişkisiyle bağlanmaz.**
- Her modül **kendi tablolarının sahibi.** Diğer modülün verisi gerekiyorsa API'sini çağırır veya event dinler.

> DDD'deki **aggregate'ler arası ID ile referans** kuralı. Mikroservislerdeki "her servisin kendi veritabanı" kuralının monolit içindeki karşılığı. İleride modülü servise çıkarmak bu sayede mümkün.

## 2. Sınırları Doğrulamak

### Tek Test, Tüm Mimari

```java
class ModularityTests {

    ApplicationModules modules = ApplicationModules.of(ShopApplication.class);

    @Test
    void verifiesModularStructure() {
        modules.verify();
    }

    @Test
    void printModules() {
        modules.forEach(System.out::println);   // Modülleri ve bağımlılıklarını yazdırır
    }
}
```

### `verify()` Neleri Yakalar?

| İhlal | Örnek |
|---|---|
| **Döngüsel bağımlılık** | `order → inventory → order` |
| **İç detaya erişim** | `payment`, `order.internal.OrderRepository`'yi kullanıyor |
| **İzin verilmeyen bağımlılık** | `allowedDependencies` dışında bir modüle erişim |

```
Violations:
- Module 'payment' depends on non-exposed type
  com.example.shop.order.internal.OrderRepository within module 'order'!
```

- Bu test **CI'da** çalışır → Mimariyi bozan değişiklik `main`'e giremez.

> Sözleşme testlerinin (consumer-driven contracts) mantığına benzer: Kırılma **deploy'dan önce**, CI'da yakalanır. Mimari kurallar dokümanda kalmaz, **testte** yaşar.

### Mevcut Projeye Eklemek

- Eski bir monolitte `verify()` ilk çalıştırmada muhtemelen **onlarca ihlal** verir. Bu normal.
- `printModules()` ile mevcut durumu gör → İhlalleri teker teker temizle.

> Strangler fig'deki "küçük adımlarla dönüştürme" mantığı: Büyük patlama yok, modül modül iyileştirme.

## 3. Event Tabanlı Modül İletişimi

### Sorun: Doğrudan Çağrı

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final InventoryService inventory;
    private final PaymentService payment;
    private final NotificationService notification;
    private final LoyaltyService loyalty;       // her yeni özellik = yeni bağımlılık

    @Transactional
    public void complete(Long orderId) {
        // ... siparişi tamamla
        inventory.reduceStock(orderId);
        notification.sendConfirmation(orderId);
        loyalty.addPoints(orderId);
    }
}
```

- `order` modülü **herkesi tanımak** zorunda.
- Bildirim gönderimi hata verirse **sipariş de geri alınır** (aynı transaction).
- Yeni bir modül siparişlere tepki vermek isterse `OrderService` değişir.

### Çözüm: Event Yayınla, İsteyen Dinlesin

```java
// order modülü (API'nin parçası)
public record OrderCompleted(Long orderId, Long customerId) {}
```

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orders;
    private final ApplicationEventPublisher events;

    @Transactional
    public void complete(Long orderId) {
        // ... siparişi tamamla
        events.publishEvent(new OrderCompleted(orderId, customerId));
    }
}
```

```java
// inventory modülü
@Component
class InventoryEventHandler {

    @ApplicationModuleListener
    void on(OrderCompleted event) {
        // stoktan düş
    }
}
```

- `order` artık `inventory`, `notification`, `loyalty`'yi **tanımıyor.**
- Bağımlılık yönü **tersine döndü:** Dinleyen modül, yayınlayan modüle bağımlı.
- Yeni bir modül siparişlere tepki vermek isterse **sadece dinleyici ekler**, `order` değişmez.

> Kafka / RabbitMQ ile servisler arası event mantığının **aynı uygulama içindeki** karşılığı. Fark: Ağ yok, broker yok, sadece Spring'in `ApplicationEvent` mekanizması.

### `@ApplicationModuleListener` Ne Yapar?

Aslında üç anotasyonun kısayolu:

```java
@Async
@Transactional(propagation = Propagation.REQUIRES_NEW)
@TransactionalEventListener
```

| Parça | Anlamı |
|---|---|
| `@TransactionalEventListener` | Yayınlayan transaction **commit edildikten sonra** çalışır (varsayılan `AFTER_COMMIT`) |
| `@Async` | Ayrı thread'de çalışır, yayınlayanı **bekletmez** |
| `REQUIRES_NEW` | Dinleyici **kendi** transaction'ında çalışır |

- Sipariş commit edilmeden stok düşülmez → Geri alınan sipariş için yanlış stok hareketi olmaz.
- Stok düşme hata verirse **sipariş etkilenmez.**

### ⚠️ Tuzak: Peki Event Kaybolursa?

```
1. Sipariş commit edildi ✅
2. Event yayınlandı, dinleyici asenkron başladı
3. Uygulama tam bu anda çöktü 💥
4. Stok hiç düşülmedi ❌  → Event hafızadaydı, kayboldu
```

- Bellekteki event'ler uygulama kapanınca **kaybolur.**
- Çözüm: Event'i **kalıcı olarak kaydetmek** → Bölüm 4.

## 4. Event Publication Registry

### Çözüm: Event'leri Veritabanına Yaz

```
@Transactional complete()
├── orders tablosuna UPDATE
├── event_publication tablosuna INSERT  ← AYNI transaction
└── COMMIT
       │
       ▼
Dinleyici çalışır → başarılıysa kayıt "tamamlandı" olarak işaretlenir
                  → hata verirse / uygulama çökerse kayıt "tamamlanmamış" kalır
```

- Event kaydı **iş verisiyle aynı transaction'da** yazılır → Sipariş commit edildiyse event kaydı da kesin var.
- Tamamlanmamış kayıtlar **tekrar işlenebilir.**

> Mikroservislerdeki **Transactional Outbox** deseninin **hazır** hali. Kendi outbox tablonu ve zamanlanmış görevini yazmana gerek yok.

### Kurulum

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-jpa</artifactId>   <!-- veya -jdbc, -mongodb -->
</dependency>
```

```yaml
spring:
  modulith:
    events:
      jdbc:
        schema-initialization:
          enabled: true          # Geliştirmede tabloyu otomatik oluştur
      republish-outstanding-events-on-restart: true   # Açılışta yarım kalanları tekrar dene
      completion-mode: update    # update / delete / archive
```

> ⚠️ Production'da tablo **Flyway** ile oluşturulmalı; `schema-initialization` geliştirme kolaylığı. (Hibernate `ddl-auto` tartışmasıyla aynı mantık.)

### Tamamlanma Modları

| Mod | Tamamlanan Kayda Ne Olur? |
|---|---|
| `update` (varsayılan) | Tamamlanma zamanı yazılır, kayıt kalır |
| `delete` | Kayıt silinir → Tablo küçük kalır |
| `archive` | Ayrı arşiv tablosuna taşınır |

### Yarım Kalanları Elle Tekrar Denemek

```java
@Component
@RequiredArgsConstructor
class RetryIncompleteEvents {

    private final IncompleteEventPublications incomplete;

    @Scheduled(fixedDelay = 60_000)
    void retry() {
        incomplete.resubmitIncompletePublicationsOlderThan(Duration.ofMinutes(5));
    }
}
```

### ⚠️ Dinleyiciler Idempotent Olmalı

- Tekrar deneme = Aynı event **birden fazla kez** işlenebilir.
- "Stoktan 1 düş" iki kez çalışırsa stok yanlış olur.
- **Çözüm:** İşlenmiş event'leri kaydet veya işlemi tekrar çalışınca aynı sonucu verecek şekilde tasarla.

> Kafka'daki **at-least-once** teslim ve idempotent tüketici konusunun aynısı.

### 2.0 ile Gelen Değişiklik

- 1.x'te bir kayıt sadece **tamamlandı / tamamlanmadı** idi. "İşleniyor mu, başarısız mı oldu?" ayrımı yoktu.
- 2.0'da **durum modeli yenilendi:** Kayıtlar işleniyor, başarısız, tekrar gönderildi gibi ara durumlarla takip edilir.
- 1.x'ten yükseltirken **tablo yapısı değişir** → Migration rehberine bakılmalı.

## 5. Event'leri Dışarıya Göndermek (Externalization)

### Sorun

- Modüller arası event'ler uygulamanın içinde. Başka bir sistem (ör. ayrı bir analitik servisi) de `OrderCompleted`'ı duymak istiyor.

### Çözüm: `@Externalized`

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-events-kafka</artifactId>   <!-- veya -amqp, -jms, -aws-sqs, -aws-sns -->
</dependency>
```

```java
@Externalized("orders.completed::#{#this.customerId()}")
public record OrderCompleted(Long orderId, Long customerId) {}
```

| Parça | Anlamı |
|---|---|
| `orders.completed` | Hedef Kafka topic'i |
| `::#{...}` | Mesaj anahtarı (SpEL) → Aynı müşterinin event'leri aynı partition'a, **sırası korunur** |

- Event hem **içerideki dinleyicilere** hem **Kafka'ya** gider.
- Kafka'ya gönderim de Event Publication Registry üzerinden → Broker geçici olarak kapalıysa event kaybolmaz.

> **Mikroservise geçiş köprüsü:** Bir modülü ileride ayrı servise çıkarınca, o servis aynı topic'i dinlemeye başlar. Yayınlayan modülün kodu **değişmez.**

## 6. Modülleri İzole Test Etmek

### Sorun

- `@SpringBootTest` **tüm uygulamayı** ayağa kaldırır → Yavaş.
- Hangi modülün hangi modüle gerçekten ihtiyaç duyduğu görünmez.

### Çözüm: `@ApplicationModuleTest`

```java
@ApplicationModuleTest     // Sadece bu test sınıfının paketindeki modül ayağa kalkar
class InventoryModuleTests {

    @Autowired InventoryService inventory;

    @Test
    void reducesStockOnOrderCompleted(Scenario scenario) {
        scenario.publish(new OrderCompleted(1L, 42L))
                .andWaitForStateChange(() -> inventory.stockOf(100L), stock -> stock == 9)
                .andVerify(stock -> assertThat(stock).isEqualTo(9));
    }
}
```

### Bootstrap Modları

| Mod | Ne Ayağa Kalkar? |
|---|---|
| `STANDALONE` (varsayılan) | Sadece modülün kendisi; dış bağımlılıklar `@MockitoBean` ile verilir |
| `DIRECT_DEPENDENCIES` | Modül + doğrudan bağımlı olduğu modüller |
| `ALL_DEPENDENCIES` | Modül + tüm bağımlılık ağacı |

```java
@ApplicationModuleTest(mode = BootstrapMode.DIRECT_DEPENDENCIES)
```

- `STANDALONE`'da test başlamıyorsa → Modül sandığından **daha fazla** şeye bağımlı. Testin kendisi bir mimari geri bildirim.

> `@WebMvcTest` / `@DataJpaTest` gibi **slice test** mantığı: Her şeyi değil, sadece ilgili dilimi yükle. Fark: Dilim katmana göre değil **modüle** göre.

### Scenario API

```java
@Test
void publishesEventWhenOrderCompleted(Scenario scenario) {
    scenario.stimulate(() -> orderService.complete(1L))
            .andWaitForEventOfType(OrderCompleted.class)
            .matching(e -> e.orderId().equals(1L))
            .toArriveAndVerify(e -> assertThat(e.customerId()).isEqualTo(42L));
}
```

| Metot | Anlamı |
|---|---|
| `stimulate(...)` | Bir işlem tetikle |
| `publish(...)` | Doğrudan bir event yayınla |
| `andWaitForEventOfType(...)` | Belirli bir event'i **bekle** (asenkron dinleyiciler için) |
| `andWaitForStateChange(...)` | Bir değerin değişmesini **bekle** |

- Asenkron dinleyicileri test ederken `Thread.sleep()` yazmaya gerek kalmaz.

## 7. Dokümantasyon ve Gözlemlenebilirlik

### Koddan Otomatik Dokümantasyon

```java
@Test
void writeDocumentation() {
    new Documenter(ApplicationModules.of(ShopApplication.class))
            .writeDocumentation();
}
```

- `target/spring-modulith-docs/` altına üretir:

| Çıktı | İçerik |
|---|---|
| **Modül diyagramları** | C4 / PlantUML formatında, modüller ve aralarındaki bağımlılıklar |
| **Application Module Canvas** | Her modül için özet: API'si, Spring bean'leri, yayınladığı ve dinlediği event'ler |

- Diyagram **koddan** üretildiği için her zaman güncel.

> Elle çizilen mimari diyagramı ilk refactoring'de eskir. Bu diyagram CI'da her build'de yeniden üretilebilir.

### Çalışma Anında Gözlem

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-insight</artifactId>
</dependency>
```

| Özellik | Açıklama |
|---|---|
| `/actuator/modulith` | Modül yapısını JSON olarak sunar |
| **Modül span'leri** | Micrometer Tracing ile her modül geçişi ayrı span olarak görünür |

```
traceId: 7ac1...
├── http: POST /api/orders/1/complete
│   └── module: order     → OrderService.complete
│       └── module: inventory → InventoryEventHandler.on(OrderCompleted)
│       └── module: notification → ...
```

> Dağıtık izlemedeki trace / span mantığı, ama servisler arası değil **modüller arası.** Monolitte bile "zaman hangi modülde harcandı?" sorusu cevaplanır.

### Ek: Moments (Zaman Event'leri)

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-moments</artifactId>
</dependency>
```

```java
@ApplicationModuleListener
void on(DayHasPassed event) {
    // gün sonu raporu
}
```

- `HourHasPassed`, `DayHasPassed`, `WeekHasPassed`, `MonthHasPassed`... gibi event'ler yayınlar.
- `@Scheduled` cron ifadeleri yerine **iş diline yakın** event'ler. Testte zaman ilerletilebilir (`TimeMachine`).

## 8. Ne Zaman Kullanmalı?

### Kullanmak Mantıklı

| Durum | Neden? |
|---|---|
| Monolit büyüyor, paketler birbirine dolanıyor | Sınırları görünür ve zorunlu kılar |
| Mikroservise geçiş düşünülüyor ama ekip / operasyon hazır değil | Önce sınırları **ucuza** doğru çiz; yanlış kesilmiş mikroservis çok pahalı |
| DDD ve bounded context'lerle çalışılıyor | Bounded context = Modül, kodda doğrulanabilir hale gelir |
| Birden fazla ekip aynı kod tabanında | Modül sahipliği ve sınırları net |
| Yeni proje | Baştan eklemenin maliyeti neredeyse sıfır |

### Gerek Olmayabilir

| Durum | Neden? |
|---|---|
| Küçük, tek amaçlı CRUD uygulaması | Bölünecek alan yok |
| Kısa ömürlü prototip | Mimari yatırımın geri dönüşü yok |
| Zaten küçük, iyi ayrılmış mikroservisler | Her servis tek bir modül gibi; getirisi az |

### "Önce Monolit" Yaklaşımı

```
Modüler monolit (Spring Modulith)
   → Sınırlar netleşti, verify() yeşil, modüller event'lerle konuşuyor
   → Bir modülün gerçekten AYRI ölçeklenmesi / deploy edilmesi gerekti
   → O modülü servise çıkar (event'ler zaten @Externalized ile Kafka'da)
```

> **Pratik bakış:**
> - Mikroservis bir **amaç** değil, belirli sorunların (bağımsız ölçekleme, bağımsız deploy, ekip özerkliği) **çözümü.**
> - Bu sorunlar yoksa modüler monolit genelde **daha doğru** seçim.
> - Modüler monolit iyi kurulduysa mikroservise geçiş bir **yeniden yazma** değil, **taşıma** olur.

---

## Sorular

**1. Modüler monolit nedir? Klasik monolitten ve mikroservislerden farkı nedir?**
Tek parça deploy edilen ama içinde net, kontrol edilen modül sınırları olan uygulamadır. Klasik monolitte sınırlar zorunlu değildir; mikroservislerde sınırlar ağ ile zorlanır ama dağıtık sistem maliyeti gelir.

**2. Spring Modulith bir modülü nasıl tanır?**
Ana uygulama paketinin doğrudan alt paketlerini modül kabul eder. Modülün ana paketi API'dir, alt paketleri iç detaydır.

**3. Java'nın `public` / package-private görünürlüğü neden tek başına yetmez?**
Farklı paketteki sınıfı kullanmak için sınıfı `public` yapmak gerekir ve `public` sınıf her yerden kullanılabilir hale gelir. Modulith bu sınıfların modül dışından kullanımını yakalar.

**4. `@NamedInterface` ne işe yarar?**
Bir modülün alt paketini, diğer modüllerin kullanabileceği ek bir API olarak dışarıya açar.

**5. `ApplicationModules.verify()` neleri yakalar?**
Modüller arası döngüsel bağımlılıkları, başka modülün iç detayına erişimi ve `allowedDependencies` dışındaki bağımlılıkları.

**6. Modüller birbirinin entity'lerine neden JPA ilişkisiyle bağlanmamalı?**
Sıkı bağ oluşur ve modül ileride ayrılamaz. Her modül kendi tablolarının sahibidir; diğer modüllere sadece ID ile referans verilir.

**7. Modüller arası doğrudan çağrı yerine event kullanmanın faydası nedir?**
Yayınlayan modül dinleyenleri tanımaz, bağımlılık yönü tersine döner. Yeni bir tepki eklemek için yayınlayan modül değişmez.

**8. `@ApplicationModuleListener` hangi anotasyonların kısayoludur?**
`@Async`, `@Transactional(propagation = REQUIRES_NEW)` ve `@TransactionalEventListener`. Dinleyici, yayınlayan transaction commit edildikten sonra ayrı thread'de ve kendi transaction'ında çalışır.

**9. Event Publication Registry hangi sorunu çözer?**
Asenkron dinleyici çalışırken uygulama çökerse bellekteki event kaybolur. Registry event'i iş verisiyle aynı transaction'da veritabanına yazar; tamamlanmamış kayıtlar tekrar işlenebilir.

**10. Event Publication Registry hangi mikroservis deseninin karşılığıdır?**
Transactional Outbox deseninin hazır halidir.

**11. Dinleyiciler neden idempotent olmalıdır?**
Tekrar denemelerde aynı event birden fazla kez işlenebilir; idempotent olmayan işlem yanlış sonuç üretir.

**12. `@Externalized` ne işe yarar?**
Event'i içerideki dinleyicilerin yanında Kafka, RabbitMQ, JMS veya SQS/SNS gibi dış sistemlere de gönderir. Modül servise çıkarıldığında yayınlayan kod değişmez.

**13. `@ApplicationModuleTest` ile `@SpringBootTest` farkı nedir?**
`@SpringBootTest` tüm uygulamayı ayağa kaldırır; `@ApplicationModuleTest` sadece ilgili modülü (ve seçilen moda göre bağımlılıklarını) yükler.

**14. `STANDALONE` modunda test başlamıyorsa bu ne anlama gelir?**
Modülün sanıldığından daha fazla modüle bağımlı olduğunu gösterir; testin kendisi mimari geri bildirimdir.

**15. Scenario API asenkron testlerde neyi kolaylaştırır?**
Bir işlemi tetikleyip belirli bir event'in gelmesini veya bir durumun değişmesini bekler; `Thread.sleep()` gerekmez.

**16. `Documenter` neden elle çizilen diyagramdan daha iyidir?**
Diyagramlar koddan üretildiği için her build'de güncel kalır.

**17. Spring Modulith 2.0'ın en önemli değişikliği nedir?**
Spring Boot 4 / Framework 7 tabanına geçiş ve ara durumları takip eden yenilenmiş Event Publication Registry.

**18. Spring Modulith ne zaman gereksizdir?**
Küçük CRUD uygulamalarında, kısa ömürlü prototiplerde ve zaten küçük, iyi ayrılmış mikroservislerde.

**19. "Önce modüler monolit, sonra mikroservis" yaklaşımı neden önerilir?**
Sınırlar ucuza ve doğru çizilir; gerçekten ayrı ölçekleme veya deploy ihtiyacı doğan modül, event'leri zaten dışarıda olduğu için yeniden yazılmadan servise taşınabilir.
