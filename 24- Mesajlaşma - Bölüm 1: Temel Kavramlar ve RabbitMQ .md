# Mesajlaşma - Bölüm 1: Temel Kavramlar ve RabbitMQ

## 1. Sorun: Servisler Birbirini Beklememeli

Sistem büyüdü: Bildirim, depo, analitik **ayrı servisler.** Sipariş servisi onlara HTTP ile haber verirse:

```java
notificationClient.sendOrderConfirmation(order);
warehouseClient.prepareOrder(order);
analyticsClient.trackOrder(order);
```

| Sorun | Açıklama |
|---|---|
| **Zamansal bağımlılık** | Tüm servislerin **aynı anda** ayakta olması gerekir. Biri bakımdaysa retry, circuit breaker, fallback devreye girer. |
| **Bilgi bağımlılığı** | Sipariş servisi ilgilenen **herkesi** tanır. Yeni servis = sipariş servisinde kod değişikliği. |
| **Yük dalgalanması** | Kampanyada dakikada 10.000 sipariş, e-posta servisi 1.000 kaldırabiliyor → Ya çöker ya sipariş servisi yavaşlar. |

> Spring Events'le tek uygulama içinde çözülen sorun, şimdi **servisler arasında.**

## 2. Çözüm: Mesaj Aracısı (Message Broker)

> **Benzetme:** Postane. Mektubu bırakırsın, alıcının o an evde olması gerekmez. Postane saklar, alıcı müsait olunca teslim eder.

```
                         ┌──────────────┐
Sipariş servisi  ──────► │ Mesaj aracısı│ ──────► Bildirim servisi
   (producer)            │   (broker)   │ ──────► Depo servisi
                         └──────────────┘ ──────► Analitik servisi
                                                   (consumer'lar)
```

| Rol | Görevi |
|---|---|
| **Producer** | Mesajı gönderen |
| **Broker** | Aradaki postane |
| **Consumer** | Mesajı alıp işleyen |

| Sorun | Aracıyla Çözümü |
|---|---|
| Zamansal bağımlılık | Tüketici kapalıyken mesajlar **bekler** |
| Bilgi bağımlılığı | Producer kimin dinlediğini bilmez; yeni servis kendi kendine dinler |
| Yük dalgalanması | Mesajlar aracıda birikir, tüketici kendi hızında işler (**load leveling**, baraj gibi) |

### Spring Events ile Farkı

| | Spring Events | Mesaj Aracısı |
|---|---|---|
| Nerede? | Tek uygulamanın **belleği** | **Ayrı sunucu** |
| Kapsam | Tek JVM | Farklı uygulamalar arası |
| Kalıcılık | Uygulama kapanınca kaybolur | **Diske** yazılır, yeniden başlatmada kaybolmaz |

> **Outbox'ın devamı:** "Büyük sistemlerde postacı olayları bir mesaj kuyruğuna yayınlar." İşte o kuyruk.

## 3. RabbitMQ'nun Modeli

```
Producer ──► Exchange ──(binding)──► Queue ──► Consumer
```

| Parça | Görevi | Benzetme |
|---|---|---|
| **Queue** | Mesajların tüketilene kadar beklediği yer | Posta kutusu |
| **Exchange** | Producer'ın gönderdiği yer; mesajın hangi kuyruklara gideceğine karar verir | Ayırma masası |
| **Binding** | Kuyruğu exchange'e bağlayan kural | "Şu türdeki mesajları bu kutuya koy" |
| **Routing key** | Producer'ın mesaja eklediği etiket (`order.placed`) | Zarftaki adres |

> **Producer neden doğrudan kuyruğa göndermez?** Kuyrukları bilmek zorunda kalırdı (bilgi bağımlılığı). Exchange sayesinde producer sadece mesajın türünü söyler; hangi kuyrukların alacağını **tüketicilerin** tanımladığı binding'ler belirler.

### Exchange Türleri

| Tür | Kural | Örnek |
|---|---|---|
| **Direct** | Routing key **birebir** eşleşmeli | `order.placed` → sadece `order.placed`'e bağlılar |
| **Topic** | Routing key **desene** uymalı. `*` = tam bir kelime, `#` = sıfır veya daha fazla kelime | `order.*` → `order.placed` ✅ `order.payment.failed` ❌<br>`order.#` → ikisi de ✅ |
| **Fanout** | Routing key'e bakmaz, **tüm** bağlı kuyruklara | Herkese duyuru |

> En esnek ve yaygın: **Topic.**

```
                                    binding: order.placed
                                 ┌──────────────────────► [notification.order-placed] ──► Bildirim
                                 │
Sipariş servisi ──► shop.orders ─┤ binding: order.placed
   routing key:     (topic)      ├──────────────────────► [warehouse.order-placed]   ──► Depo
   order.placed                  │
                                 │ binding: order.#
                                 └──────────────────────► [analytics.orders]         ──► Analitik
```

- Her servisin **kendi kuyruğu**, aynı mesajın **kendi kopyası.**
- Yeni sadakat servisi: Kendi kuyruğunu oluşturup bağlar. **Sipariş servisinde tek satır değişmez.**

### İki Farklı Dağıtım

| Durum | Davranış | Model |
|---|---|---|
| **Farklı kuyruklar** | Herkes **bir kopya** alır | **Publish-subscribe** |
| **Aynı kuyruk, birden fazla tüketici** | Her mesajı **sadece biri** alır | **Competing consumers** (rekabet eden tüketiciler) |

- İkisi birlikte: Her **servis türü** kendi kuyruğu (herkes kopya), her servisin **kopyaları** o kuyruğu paylaşır (iş bölünür).
- E-posta yetişemiyorsa 2 kopya daha → Kuyruk 3 kat hızlı erir.

> ShedLock'taki "üç sunucu aynı işi üç kez yapmasın" sorunu burada **kendiliğinden yok**: Kuyruk mesajı tek tüketiciye verir.

## 4. RabbitMQ'yu Çalıştırmak

```yaml
  rabbitmq:
    image: rabbitmq:4-management
    container_name: shop-rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: shop
      RABBITMQ_DEFAULT_PASS: shop
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
```

| Port | Görevi |
|---|---|
| **5672** | Uygulamaların bağlandığı mesajlaşma portu |
| **15672** | Yönetim arayüzü (`http://localhost:15672`): Exchange'ler, kuyruklar, mesaj sayıları, tüketiciler |

> ⚠️ Varsayılan `guest` / `guest` güvenlik gereği **sadece aynı makineden** bağlanabilir. Başka konteynerdeki uygulama bağlanamaz.

## 5. Spring ile RabbitMQ

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

- **AMQP:** RabbitMQ'nun konuştuğu protokol (HTTP web için, AMQP mesajlaşma için).

```yaml
spring:
  rabbitmq:
    host: localhost       # Compose'da uygulama da konteynerdeyse: rabbitmq
    port: 5672
    username: shop
    password: ${RABBITMQ_PASSWORD}
```

### Producer Tarafı (Sipariş Servisi)

```java
@Configuration
public class RabbitConfig {

    public static final String ORDERS_EXCHANGE = "shop.orders";

    @Bean
    public TopicExchange ordersExchange() {
        return new TopicExchange(ORDERS_EXCHANGE);
    }

    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

- Producer **sadece exchange** tanımlar, kuyruk tanımlamaz (kuyruklar tüketicilerin işi).
- Spring yapıyı başlangıçta RabbitMQ'da **kendisi oluşturur** (varsa dokunmaz).
- `MessageConverter` bean'i Spring Boot tarafından gönderen ve dinleyen tarafa **otomatik** uygulanır.

> **Neden JSON?** Redis'teki serileştirme tartışmasının aynısı: Java serileştirmesi okunamaz, `Serializable` ister, kırılgandır. Mesajlaşmada daha önemli: Gönderen ve alan **farklı uygulamalar**, belki farklı diller. JSON ortak dil.

```java
@Component
public class OrderEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishOrderPlaced(OrderPlacedMessage message) {
        rabbitTemplate.convertAndSend(RabbitConfig.ORDERS_EXCHANGE, "order.placed", message);
    }
}
```

- `convertAndSend`: JSON'a çevir → exchange'e, routing key ile gönder.
- `RestClient` gibi ama **cevap beklemez**; aracıya teslim edip döner.

### Consumer Tarafı (Bildirim Servisi, Ayrı Uygulama)

```java
@Configuration
public class NotificationRabbitConfig {

    public static final String QUEUE = "notification.order-placed";

    @Bean
    public Queue orderPlacedQueue() {
        return QueueBuilder.durable(QUEUE).build();
    }

    @Bean
    public Binding orderPlacedBinding(Queue orderPlacedQueue) {
        return BindingBuilder
                .bind(orderPlacedQueue)
                .to(new TopicExchange("shop.orders"))
                .with("order.placed");
    }
}
```

```java
@Component
public class OrderPlacedConsumer {

    private final EmailService emailService;

    @RabbitListener(queues = NotificationRabbitConfig.QUEUE)
    public void handle(OrderPlacedMessage message) {
        emailService.sendConfirmation(message.customerEmail(), message.orderId());
    }
}
```

- Kuyruğu ve binding'i **tüketici** tanımlar. Sipariş servisi bu kuyruktan habersiz.
- `@RabbitListener` = `@EventListener`'ın mesajlaşma karşılığı.

| Kalıcılık | Anlamı |
|---|---|
| **Durable kuyruk** | RabbitMQ yeniden başlayınca silinmez |
| **Persistent mesaj** (Spring varsayılanı) | Diske yazılır |

> İkisi birlikte: Aracının yeniden başlaması mesaj kaybına yol açmaz.

### Mesaj Sözleşmesi

> ⚠️ Mesaj sınıfını **ortak kütüphane** olarak paylaşmak: İki servis **derleme zamanında** bağlanır, birlikte derlenip deploy edilmeli. Servisleri ayırmanın tam tersi.

- **Sözleşme JSON'un yapısıdır, Java sınıfı değil.**
- Her servis **kendi record'unu** tanımlar (anti-corruption layer).

| Değişiklik | Etkisi |
|---|---|
| **Alan eklemek** | ✅ Güvenli. `ObjectMapper` tanımadığı alanları yok sayar. |
| **Alan silmek / adını değiştirmek** | ⚠️ **Sözleşme değişikliği.** O alanı okuyan tüketiciler bozulur. Yeni mesaj türü veya sürüm gerekebilir. |

> Redis'teki "yeni deploy, eski önbellek" sorununun mesajlaşma versiyonu: Kuyrukta bekleyen **eski mesajlar** ve yeni deploy edilmiş tüketiciler.

## 6. Güvenilirlik

### Onaylama (Acknowledgement)

- RabbitMQ mesajı verince **silmez**, tüketicinin **onayını (ack)** bekler.
- Onay gelirse → Silinir.
- Onay gelmeden bağlantı koparsa → **Tekrar kuyruğa** konur, başka tüketiciye verilir.
- Spring varsayılanı: Metot normal biterse onaylar, exception fırlatırsa onaylamaz.

### ⚠️ Tekrar Gelen Mesaj

```
E-posta gönderildi → onay vermeden ÇÖKTÜ → RabbitMQ tekrar verdi → E-posta İKİ KEZ
```

> Mesajlaşma **en az bir kez** (at-least-once) teslim garanti eder. Çözüm: **Idempotent tüketici.**

```java
@RabbitListener(queues = NotificationRabbitConfig.QUEUE)
@Transactional
public void handle(OrderPlacedMessage message) {
    if (processedMessageRepository.existsById(message.messageId())) {
        log.debug("Mesaj zaten işlendi, atlanıyor: {}", message.messageId());
        return;
    }
    emailService.sendConfirmation(message.customerEmail(), message.orderId());
    processedMessageRepository.save(new ProcessedMessage(message.messageId()));
}
```

- `messageId`: Producer'ın ürettiği benzersiz kimlik. **Outbox kaydının kimliği** ideal aday.
- **En az bir kez + idempotent alıcı = Etkili olarak tam bir kez.**

> Küçük boşluk: E-posta gönderilip `save` commit edilmeden çökülürse yine tekrar gider (dış yan etki geri alınamaz). E-posta sağlayıcısına idempotency key veya birkaç tekrarı kabul etmek. Mükemmel çözüm yok, olasılık çok düşük.

### Zehirli Mesaj (Poison Message)

Her denemede hata veren mesaj (bozuk e-posta adresi):

```
Exception → onay yok → tekrar kuyruğa → hemen tekrar alındı → exception → ... SONSUZA KADAR
```

- CPU %100, loglar dolar, diğer mesajlar işlenemeyebilir.

**Çözüm 1: Retry ile sınırla**

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1s
          multiplier: 2
        default-requeue-rejected: false
```

- En fazla 3 deneme, artan aralıklarla (geçici hatalar düzelir).
- `default-requeue-rejected: false`: Denemeler tükenince **tekrar kuyruğa konmaz.** Sonsuz döngü kırılır.

> ⚠️ Tüketici retry'ı denemeler arasında **thread'i bekletir.** Uzun beklemeler için gecikmeli mesaj desenleri var.

**Çözüm 2: Dead Letter Queue (DLQ)**

```java
@Bean
public Queue orderPlacedQueue() {
    return QueueBuilder.durable("notification.order-placed")
            .deadLetterExchange("shop.dlx")
            .deadLetterRoutingKey("notification.order-placed.dlq")
            .build();
}

@Bean
public DirectExchange deadLetterExchange() {
    return new DirectExchange("shop.dlx");
}

@Bean
public Queue orderPlacedDlq() {
    return QueueBuilder.durable("notification.order-placed.dlq").build();
}

@Bean
public Binding dlqBinding() {
    return BindingBuilder.bind(orderPlacedDlq())
            .to(deadLetterExchange())
            .with("notification.order-placed.dlq");
}
```

- İşlenemeyen mesaj **kaybolmaz**, DLQ'da bekler. Asıl kuyruk akmaya devam eder.
- İncele → düzelt → yönetim arayüzünden asıl kuyruğa geri gönder.

> Outbox'taki "N denemeden sonra `FAILED` + uyarı" fikrinin mesajlaşma karşılığı.

> ⚠️ **DLQ'daki mesaj sayısı izlenmeli, sıfırdan büyükse uyarı üretmeli.** İzlenmeyen DLQ, mesajların sessizce kaybolmasından farksızdır.

### Producer Tarafı: Publisher Confirms

`convertAndSend` döndüğünde mesajın RabbitMQ'ya ulaştığı **varsayılan olarak garanti değil.**

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated
```

- RabbitMQ mesajı güvenle kaydedince producer'a **onay** gönderir.
- **Outbox ile:** Postacı kaydı ancak onay geldikten **sonra** "gönderildi" yapar. Onay yoksa `PENDING` kalır, tekrar gönderilir (tüketici idempotent olduğu için zararsız).

### Büyük Resim

```
[Sipariş servisi]
   │ 1. Sipariş + outbox kaydı, aynı transaction'da
   ▼
[outbox_events tablosu]
   │ 2. Postacı (@Scheduled + ShedLock) okur
   ▼
[RabbitMQ] ◄── 3. Publisher confirm gelince outbox kaydı "gönderildi"
   │ 4. Exchange ilgili kuyruklara kopyalar (durable kuyruk, persistent mesaj)
   ▼
[Bildirim servisi]
   │ 5. İşler, işlenmiş mesaj kimliğini kaydeder (idempotent)
   │ 6. Onaylar (ack); hata → retry, tükenince → DLQ
   ▼
[E-posta gönderildi]
```

| Adım | Çökmeye Karşı Önlem |
|---|---|
| 1 | Transaction |
| 2 | Outbox + ShedLock |
| 3 | Publisher confirm |
| 4 | Durable kuyruk + persistent mesaj |
| 5 | Idempotency + ack |
| 6 | Retry + DLQ |

> Mesaj hiçbir noktada **kaybolmaz**, tekrarlar **zararsızdır.**

### ⚠️ Sıralama Garanti Değil

- Rekabet eden tüketicilerde `order.placed` ve ardından `order.cancelled` **farklı kopyalara** gidebilir → İptal, oluşturmadan **önce** işlenebilir.
- Retry'lar da sırayı bozar.
- **Tasarım sıranın garanti olmadığını varsaymalı:** Mesajlara zaman damgası / sürüm numarası koy; tüketici eski mesajı yeni durumun üzerine yazmasın.
- Sıralamanın önemli olduğu durumlar **Kafka'nın** güçlü olduğu yerlerden biri.

## 7. Test Etmek

```java
@SpringBootTest
@Testcontainers
class OrderPlacedConsumerTest {

    @Container
    @ServiceConnection
    static RabbitMQContainer rabbit = new RabbitMQContainer("rabbitmq:4-management");

    @Autowired private RabbitTemplate rabbitTemplate;
    @MockitoBean private EmailService emailService;

    @Test
    void consumesOrderPlacedMessage() {
        rabbitTemplate.convertAndSend("shop.orders", "order.placed",
                new OrderPlacedMessage("msg-1", 42L, "ali@example.com"));

        await().atMost(Duration.ofSeconds(5))
                .untilAsserted(() -> verify(emailService).sendConfirmation("ali@example.com", 42L));
    }
}
```

- Tüketici başka thread'de çalışır → **Awaitility** vazgeçilmez.
- Değerli testler: Aynı mesajı iki kez gönderip `times(1)` ile **idempotency**, bozuk mesajın **DLQ'ya düştüğü.**

---

## Sorular

**1. Servisler arasında HTTP yerine mesaj aracısı hangi sorunları çözer?**
Zamansal bağımlılık (mesajlar bekler), bilgi bağımlılığı (producer tüketicileri tanımaz) ve yük dalgalanması (aracı yükü emer).

**2. Spring Events ile mesaj aracısı farkı nedir?**
Spring Events tek uygulamanın belleğinde çalışır ve kapanınca kaybolur. Mesaj aracısı ayrı sunucudur, diske yazar ve uygulamalar arası çalışır.

**3. Producer neden doğrudan kuyruğa göndermez?**
Kuyrukları bilmek zorunda kalırdı. Exchange'e routing key ile gönderir; hangi kuyrukların alacağını tüketicilerin binding'leri belirler.

**4. Direct, topic ve fanout exchange farkı nedir?**
Direct birebir anahtar eşleşmesi, topic desen eşleşmesi (`*` bir kelime, `#` sıfır veya daha fazla), fanout tüm bağlı kuyruklara gönderir.

**5. Publish-subscribe ile competing consumers farkı nedir?**
Farklı kuyruklarda her servis mesajın kopyasını alır. Aynı kuyruğu dinleyen tüketicilerde her mesajı sadece biri alır, iş bölünür.

**6. Mesajlarda neden JSON kullanılır?**
Gönderen ve alan farklı uygulamalardır; Java serileştirmesi okunamaz ve kırılgandır, JSON ortak formattır.

**7. Mesaj sınıfını ortak kütüphanede paylaşmanın sakıncası nedir?**
Servisler derleme zamanında bağlanır. Sözleşme JSON yapısı olmalı, her servis kendi sınıfını tanımlamalıdır.

**8. Mesaj sözleşmesinde hangi değişiklikler güvenlidir?**
Alan eklemek güvenlidir. Alan silmek veya adını değiştirmek tüketicileri bozar ve sözleşme değişikliğidir.

**9. Durable kuyruk ve persistent mesaj ne sağlar?**
Kuyruk ve mesajlar aracı yeniden başladığında kaybolmaz.

**10. Ack mekanizması nasıl çalışır?**
Aracı mesajı tüketicinin onayına kadar saklar; onay gelmeden bağlantı koparsa mesajı tekrar kuyruğa koyar.

**11. Mesajlar neden tekrar gelebilir, nasıl zararsız hale getirilir?**
İşlenip onay verilmeden çökülürse mesaj tekrar verilir (at-least-once). Tüketici işlenen mesaj kimliklerini saklayarak idempotent yapılır.

**12. Zehirli mesaj nedir, nasıl ele alınır?**
Her denemede hata veren mesajdır; varsayılan davranışta sonsuz döngüye girer. Retry ile denemeler sınırlanır, `default-requeue-rejected: false` ile tekrar kuyruğa konmaz ve DLQ'ya gönderilir.

**13. DLQ neden izlenmelidir?**
İzlenmeyen DLQ'daki mesajlar fiilen kaybolmuş sayılır; mesaj sayısı sıfırdan büyükse uyarı üretilmelidir.

**14. Publisher confirms ne işe yarar?**
Aracı mesajı güvenle kaydettiğinde producer'a onay verir. Outbox postacısı kaydı ancak onaydan sonra "gönderildi" yapar.

**15. Mesajların sırayla işleneceği varsayılabilir mi?**
Hayır. Rekabet eden tüketiciler ve retry'lar sırayı bozabilir; tasarım sıranın garanti olmadığını varsaymalıdır.
