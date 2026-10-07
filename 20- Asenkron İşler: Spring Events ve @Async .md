# Asenkron İşler - Bölüm 1: Spring Events ve @Async

## 1. Sorun: Her Şeyi Yapan Metot

```java
@Transactional
public OrderResponse placeOrder(OrderRequest request) {
    Order order = orderRepository.save(new Order(...));   // 20 ms
    emailService.sendConfirmation(order);                 // 800 ms
    loyaltyService.addPoints(order);
    warehouseClient.notifyNewOrder(order);                // 500 ms
    analyticsClient.trackOrder(order);                    // 300 ms
    return toResponse(order);
}
```

| Sorun | Açıklama | Çözüm |
|---|---|---|
| **Bağımlılık** | `OrderService` e-postayı, puanları, depoyu, analitiği tanıyor. "SMS de gönderilsin" → `OrderService` değişir. Constructor 5 parametre (sınıf çok iş yapıyor işareti). | **Spring Events** |
| **Bekleme** | Sipariş 20 ms'de kaydedildi ama kullanıcı **1,6 sn** bekliyor. Analitik çökerse **sipariş geri alınır.** | **`@Async`** |

## 2. Spring Events

`OrderService` kimin ne yapacağını bilmez, sadece **"bir sipariş oluşturuldu"** diye duyurur.

### Olay

```java
public record OrderPlacedEvent(
        Long orderId,
        Long customerId,
        String customerEmail,
        BigDecimal total
) {}
```

> **Geçmiş zamanla isimlendirilir:** `OrderPlaced`, `UserRegistered`, `PaymentFailed`.
> Olay **olmuş bir gerçektir** ("sipariş oluşturuldu"). Komut değildir ("e-posta gönder"); komut kimin yapacağını bilmeyi gerektirir.

### Yayınlamak

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public OrderResponse placeOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(...));
        eventPublisher.publishEvent(new OrderPlacedEvent(
                order.getId(), order.getCustomerId(), order.getCustomerEmail(), order.getTotal()));
        return toResponse(order);
    }
}
```

- `ApplicationEventPublisher` Spring'in sağladığı bean.
- 5 bağımlılık → 2.

### Dinlemek

```java
@Component
public class OrderEmailListener {

    private final EmailService emailService;

    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        emailService.sendConfirmation(event.customerEmail(), event.orderId());
    }
}
```

- Parametre tipine uyan her olay yayınlandığında çağrılır.
- **"SMS de gönderilsin"** → Yeni `OrderSmsListener` yaz. **`OrderService`'e dokunma.**

| Kavram | Açıklama |
|---|---|
| **Açık-kapalı ilkesi** | Sınıf genişlemeye açık, değişikliğe kapalı |
| **Observer deseni** | Bir nesne olay yayınlar, ilgilenenler gözlemler |

> `List<NotificationService>` enjeksiyonunun daha gelişmiş hali.

### Olayın İçinde Ne Olmalı?

**Entity değil, kimlikler ve değişmez veriler** (`record`):
- Entity değiştirilebilir → Bir dinleyicinin değişikliği diğerlerini etkiler.
- Dinleyici başka thread'de, transaction sonrası çalışabilir → Entity **detached**, lazy alan → `LazyInitializationException`.

> Caching'deki "entity değil DTO önbelleğe al" tartışmasının aynısı.

## 3. ⚠️ Events Varsayılan Olarak Senkrondur

`publishEvent` çağrılınca dinleyiciler **hemen, aynı thread'de, aynı transaction'da, sırayla** çalışır.

| Sonuç | Açıklama |
|---|---|
| Kullanıcı hâlâ bekliyor | `publishEvent` tüm dinleyiciler bitene kadar dönmez |
| Dinleyici hatası siparişi geri alır | Exception `placeOrder`'a çıkar, **rollback** |
| **Olmamış sipariş için e-posta** | Dinleyici **commit'ten önce** çalışır. E-posta gitti, sonra rollback → Veritabanında sipariş yok, müşteriye "siparişiniz alındı" e-postası gitti. |

> Events kodu **düzenledi**, davranışı **değiştirmedi.**

## 4. @TransactionalEventListener

```java
@TransactionalEventListener
public void onOrderPlaced(OrderPlacedEvent event) {
    emailService.sendConfirmation(event.customerEmail(), event.orderId());
}
```

- Olay transaction'a bağlanır, dinleyici transaction **başarıyla commit edildikten sonra** çalışır (`phase = AFTER_COMMIT`, varsayılan).
- Rollback olursa dinleyici **hiç çalışmaz.**
- Dış dünyaya dokunan işler, veritabanındaki gerçek kesinleştikten sonra yapılır.

| Aşama | Ne Zaman? |
|---|---|
| `AFTER_COMMIT` (varsayılan) | Commit başarılı olduktan sonra |
| `AFTER_ROLLBACK` | Rollback sonrası |
| `AFTER_COMPLETION` | Her iki durumda da |
| `BEFORE_COMMIT` | Commit'ten hemen önce |

### ⚠️ Transaction Yoksa Çalışmaz

Olay transaction dışında yayınlanırsa dinleyici **sessizce çağrılmaz.** ("Neden e-postalar gitmiyor?")
Çözüm: `@TransactionalEventListener(fallbackExecution = true)`

### Veritabanına Yazan Dinleyici

```java
@TransactionalEventListener
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void onOrderPlaced(OrderPlacedEvent event) {
    loyaltyRepository.addPoints(event.customerId(), calculatePoints(event.total()));
}
```

- Asıl transaction commit edilmiş, yeni değişiklik için kullanılamaz.
- **`REQUIRES_NEW`:** Mevcut olsa bile **yeni, bağımsız** transaction aç. (Varsayılan davranışın tersi: mevcut transaction'a katılmak.)

> `AFTER_COMMIT` dinleyici hâlâ **aynı thread'de** çalışır, kullanıcı hâlâ bekler.

## 5. @Async Nasıl Çalışır?

### Normal Durum: İstek Başına Bir Thread

Spring tüm işleri **tek** thread'e koymaz. Her HTTP isteği Tomcat havuzundan **kendi** thread'ini alır ve o isteğe ait **her şey baştan sona o thread'de, sırayla** çalışır:

```
nio-8080-exec-3:  [controller] → [service] → [repository] → [dinleyici] → [e-posta (800 ms)] → cevap
                  |←────────────────── kullanıcı tüm bu süre boyunca bekliyor ──────────────────→|
```

Aynı anda gelen başka istek **başka** thread alır (`nio-8080-exec-4`). Ama her istek kendi içinde tek thread'de, adım adım ilerler.

### @Async ile

```
nio-8080-exec-3:  [controller] → [service] → [repository] → [görevi kuyruğa bırak] → cevap ✅
                                                                    │
                                                                    ▼
async-1:                                                    [e-posta gönder (800 ms)]
```

İsteğin thread'i işi **kendisi yapmaz**; kuyruğa bırakıp devam eder. Kuyruktaki işi **başka bir havuzdaki** thread alıp çalıştırır. İki thread birbirini beklemez.

### Yeni Thread Oluşturmaz, Hazır Havuz Kullanır

- Her `@Async` çağrısında **sıfırdan** thread yaratılmaz.
- Uygulama başlarken `@Async` için **ayrı bir havuz** hazırlanır. İş bitince thread havuza döner.
- **Neden?** Thread oluşturmak pahalı; her işe yeni thread açılsa yoğun anlarda bellek tükenir.

| Havuz | Thread Adları | Görevi |
|---|---|---|
| **Tomcat havuzu** | `nio-8080-exec-1`, `-2`, ... | Gelen HTTP isteklerini işlemek |
| **`@Async` havuzu** | `async-1`, `async-2`, ... | Arka plana bırakılan işleri çalıştırmak |

### İşi Kuyruğa Bırakan: Proxy

`onOrderPlaced` çağrıldığında önce **proxy** devreye girer:
1. "Bu metot `@Async`, şimdi çalıştırmayacağım."
2. Çağrıyı parametreleriyle **görev** olarak paketler.
3. Görevi `@Async` havuzunun **kuyruğuna** bırakır.
4. **Hemen döner.**

Havuzdaki boşta bir thread görevi alır ve gerçek metodu **o thread'de** çalıştırır.

> **Benzetme:** Yöneticiye iş getirdin; asistan "ekibe iletiyorum" deyip işi masaya bıraktı, sen hemen çıktın. İşi ekipten biri sonra yapıyor.

### Kendi Gözünle Görmek

```java
log.info("Sipariş kaydediliyor, thread: {}", Thread.currentThread().getName());   // OrderService
log.info("E-posta gönderiliyor, thread: {}", Thread.currentThread().getName());   // Listener
```

```
# @Async YOK
Sipariş kaydediliyor, thread: nio-8080-exec-3
E-posta gönderiliyor, thread: nio-8080-exec-3     ← aynı thread

# @Async VAR
Sipariş kaydediliyor, thread: nio-8080-exec-3
E-posta gönderiliyor, thread: async-1             ← farklı thread
```

> Varsayılan log formatı thread adını zaten gösterir.

### Kullanım

```java
@SpringBootApplication
@EnableAsync
public class ShopApplication { ... }
```

```java
@Component
public class OrderEmailListener {

    @Async
    @TransactionalEventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        emailService.sendConfirmation(event.customerEmail(), event.orderId());
    }
}
```

**Akış:**
1. `placeOrder` siparişi kaydeder, olayı yayınlar.
2. Transaction **commit** edilir.
3. Dinleyici `@Async` kuyruğuna verilir.
4. `placeOrder` **hemen döner** → Kullanıcı 20 ms'de cevap alır.
5. E-posta arka planda gönderilir. E-posta servisi çökse de sipariş etkilenmez.

| Araç | Çözdüğü |
|---|---|
| Events | Bağımlılık |
| `@TransactionalEventListener` | Olmamış sipariş için e-posta |
| `@Async` | Bekleme |

### ⚠️ Proxy Tuzakları

- **`@EnableAsync` unutulursa** → Metot sessizce **senkron** çalışır.
- **Aynı sınıftan çağrılırsa** → `this` proxy'ye uğramaz, çağıranın thread'inde çalışır.
- **`private` metotlarda** → Çalışmaz.

> Dinleyicileri yayınlayan sınıftan **ayrı bean** olarak yazmak bu tuzaktan korur.

- Dönüş tipi: `void` veya sonuç gerekiyorsa `CompletableFuture<T>`.

### Havuz Ayarları

```yaml
spring:
  task:
    execution:
      pool:
        core-size: 4          # normalde hazır tutulan thread sayısı
        max-size: 16
        queue-capacity: 500   # hepsi meşgulse kuyrukta en fazla 500 iş
      thread-name-prefix: async-
```

> ⚠️ Varsayılan kuyruk kapasitesi **sınırsız.** E-posta servisi yavaşlarsa görevler birikir, **bellek dolar.** (Caffeine'deki "sınırsız önbellek" sorununun aynısı.)

- `thread-name-prefix`: Logda satırın arka plan görevinden geldiği anında görülür.

### Sanal Thread'ler (Java 21)

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

- Spring Boot 3.2+ her görev için **yeni sanal thread** oluşturur (çok ucuz, havuza gerek yok).
- **Mantık değişmez:** İş yine ayrı thread'de çalışır, aşağıdaki tuzakların hepsi geçerlidir.

## 6. ⚠️ @Async'in Tuzakları

> **Tek cümle:** İş başka thread'e geçtiğinde, **ilk thread'e bağlı olan her şey geride kalır.**

### Hatalar Kaybolur

- `void` `@Async` metottaki exception **çağırana ulaşmaz** (çoktan dönmüştür).
- `GlobalExceptionHandler` devreye girmez (HTTP isteği yok).
- E-postalar sessizce gitmemeye başlar, müşteri şikâyet edince öğrenilir.

**Çözüm:**
- Hata **görevin içinde** yakalanıp anlamlı şekilde loglanır.
- Metrik ile izlenir (`email.failures` sayacı).
- Genel yakalayıcı: `AsyncConfigurer` → `AsyncUncaughtExceptionHandler`

### ThreadLocal'lar Taşınmaz

| ThreadLocal | `async-1`'de Ne Olur? |
|---|---|
| **MDC** (`requestId`, `traceId`) | **Boş.** Log satırlarında görünmez, hangi isteğe ait olduğu bulunamaz. |
| **SecurityContextHolder** | **null.** `@PreAuthorize`'lı metot reddedilir. |
| **Transaction** | **Yok.** Ana thread'in transaction'ı görünmez. |

> `async-1`, sanki hiç istek gelmemiş, kimse giriş yapmamış gibi **temiz** bir thread.

**MDC için çözüm: TaskDecorator**

```java
@Component
public class MdcTaskDecorator implements TaskDecorator {

    @Override
    public Runnable decorate(Runnable task) {
        Map<String, String> context = MDC.getCopyOfContextMap();   // ANA thread'de
        return () -> {
            if (context != null) MDC.setContextMap(context);       // ARKA PLAN thread'inde
            try {
                task.run();
            } finally {
                MDC.clear();   // havuzdaki thread yeniden kullanılır, sızıntıyı önle
            }
        };
    }
}
```

- Spring Boot, `TaskDecorator` bean'ini `@Async` havuzuna **otomatik** uygular.
- `decorate` ana thread'de çağrılır (kopyayı alır), döndürdüğü `Runnable` arka planda çalışır (kopyayı koyar).
- `finally` temizliği: `RequestIdFilter`'daki ile aynı sebep.

**Transaction için çözüm:** Olaya **kimlik** koy, dinleyicide veriyi **kendi transaction'ında yeniden yükle.**
> Arka plan thread'ine detached entity gönderip lazy alana erişmek = en klasik `LazyInitializationException`.

### En Önemlisi: Kaybolabilir

```
Sipariş commit → e-posta görevi kuyrukta → deploy → uygulama yeniden başladı → GÖREV KAYBOLDU
```

- `@Async` kuyruğu **uygulamanın belleğinde.** Kapanma, çökme, yeniden başlama → Çalışmamış görevler **iz bırakmadan** yok olur.

> **Sor:** "Bu görev kaybolursa ne olur?"

| Görev | Kaybolursa |
|---|---|
| Analitik kaydı | Kabul edilebilir |
| Önbellek ısıtma | Önemsiz |
| Onay e-postası | Kötü, ama dünyanın sonu değil |
| Depo hazırlık bildirimi | **Sipariş hiç gönderilmez. Kabul edilemez.** |

> Kaybolmaması gereken işler için görev **kalıcı bir yerde (veritabanı)** saklanmalı → **Transactional outbox** (Bölüm 2), mesaj kuyrukları (RabbitMQ, Kafka).

## 7. Test Etmek

### ⚠️ Kararsız (Flaky) Test

```java
orderService.placeOrder(request);
verify(emailService).sendConfirmation(any(), any());   // bazen geçer, bazen kırılır
```

- `placeOrder` döndüğünde dinleyici henüz çalışmamış olabilir.
- Rastgele kırılan testler test paketine güveni yok eder; insanlar kırmızı testleri görmezden gelmeye başlar.

### Olayın Yayınlandığını Test Etmek

```java
@SpringBootTest
@RecordApplicationEvents
class OrderServiceEventTest {

    @Autowired private OrderService orderService;
    @Autowired private ApplicationEvents events;

    @Test
    void placeOrder_publishesOrderPlacedEvent() {
        OrderResponse response = orderService.placeOrder(request);

        assertThat(events.stream(OrderPlacedEvent.class))
                .singleElement()
                .satisfies(event -> assertThat(event.orderId()).isEqualTo(response.id()));
    }
}
```

- `@RecordApplicationEvents`: Test sırasında yayınlanan tüm olayları kaydeder.
- **`OrderService`'in sorumluluğu:** Olayı yayınlamak. **Dinleyicinin sorumluluğu:** E-postayı göndermek (ayrı test edilir).

### Asenkron Sonucu Beklemek: Awaitility

```java
await().atMost(Duration.ofSeconds(2))
        .untilAsserted(() -> verify(emailService).sendConfirmation(any(), any()));
```

- "En fazla 2 sn içinde geçmeli"; daha erken geçerse hemen devam eder.
- **`Thread.sleep(2000)` yerine:** Sabit bekleme ya testi gereksiz yavaşlatır ya da yavaş makinede yine kararsız kalır.

---

## Sorular

**1. Spring Events ve `@Async` hangi sorunları çözer?**
Events bağımlılığı çözer (yayınlayan, dinleyicileri tanımaz). `@Async` beklemeyi çözer (iş ayrı thread'de çalışır).

**2. Olaylar neden geçmiş zamanla isimlendirilir?**
Olay olmuş bir gerçeği anlatır, komut değildir. Komut kimin yapacağını bilmeyi gerektirir.

**3. `@EventListener` dinleyicisi exception fırlatırsa sipariş ne olur?**
Dinleyici aynı thread ve transaction'da senkron çalışır; exception yayınlayan metoda çıkar ve sipariş geri alınır.

**4. `@EventListener` ile e-posta göndermek neden "olmamış sipariş için e-posta" sorununa yol açar?**
Dinleyici commit'ten önce çalışır; sonradan rollback olursa e-posta gitmiş ama sipariş yoktur. `@TransactionalEventListener` sadece commit sonrası çalışır.

**5. `@TransactionalEventListener` dinleyicisi neden hiç çağrılmayabilir?**
Olay transaction dışında yayınlanıyorsa. `fallbackExecution = true` kullanılabilir.

**6. `AFTER_COMMIT` dinleyicisi veritabanına yazacaksa neden `REQUIRES_NEW` gerekir?**
Asıl transaction commit edilmiştir; yeni, bağımsız bir transaction gerekir.

**7. Spring normalde bir isteği nasıl işler?**
Her istek Tomcat havuzundan kendi thread'ini alır ve o isteğe ait her şey o thread'de sırayla çalışır.

**8. `@Async` her çağrıda yeni thread mi oluşturur?**
Hayır (platform thread'lerde). Ayrı bir havuz hazırlanır; işler kuyruğa bırakılır ve havuzdaki hazır thread'ler tarafından çalıştırılır. Sanal thread'ler açıksa her görev için yeni sanal thread oluşturulur.

**9. `@Async` arka planda nasıl çalışır?**
Proxy çağrıyı yakalar, görev olarak paketleyip kuyruğa bırakır ve hemen döner. Havuzdaki bir thread görevi alıp gerçek metodu çalıştırır.

**10. `@Async` hangi durumlarda sessizce senkron çalışır?**
`@EnableAsync` yoksa, metot aynı sınıftan çağrılıyorsa veya `private` ise.

**11. Varsayılan `@Async` havuzunun production riski nedir?**
Kuyruk kapasitesi sınırsızdır; yavaşlayan bir işte görevler birikip belleği doldurabilir.

**12. `void` `@Async` metottaki exception'a ne olur?**
Çağırana ulaşmaz, `GlobalExceptionHandler` devreye girmez. Görev içinde ele alınmalı, loglanmalı ve izlenmelidir.

**13. `@Async` metotta `requestId` neden loglarda görünmez?**
MDC bir `ThreadLocal`'dır, arka plan thread'inde boştur. `TaskDecorator` ile kopyalanır.

**14. Arka plan thread'inde hangi `ThreadLocal`'lar yoktur?**
MDC, `SecurityContextHolder` ve transaction.

**15. Olaya neden entity yerine kimlik konur?**
Entity değiştirilebilir ve arka plan thread'inde detached olur; lazy alanlara erişim `LazyInitializationException` verir.

**16. `@Async` görevi uygulama yeniden başlarsa ne olur?**
Kuyruk bellekte olduğu için kaybolur. Kaybolmaması gereken işler için transactional outbox veya mesaj kuyruğu gerekir.

**17. Asenkron testlerde `Thread.sleep` yerine neden Awaitility kullanılır?**
Sabit bekleme ya yavaşlatır ya kararsız kalır; Awaitility koşul sağlanınca devam eder, üst süre içinde sağlanmazsa testi başarısız sayar.
