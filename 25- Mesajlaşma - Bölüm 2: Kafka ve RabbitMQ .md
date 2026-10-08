# Mesajlaşma - Bölüm 2: Kafka ve RabbitMQ ile Karşılaştırma

## 1. Farklı Model: Kuyruk Değil, Kayıt Defteri

> **Benzetme:** Muhasebe defteri. Her işlem sırayla bir sonraki satıra yazılır, **silinmez ve değiştirilmez.** Her okuyucu kendi **ayracını** kullanır ("152. satıra kadar okudum"). Biri okuyunca satırlar kaybolmaz; yeni gelen baştan okuyabilir.

| | RabbitMQ (posta kutusu) | Kafka (kayıt defteri) |
|---|---|---|
| Mesaj okununca | **Silinir** | **Kalır** (saklama süresi boyunca) |
| Nerede kalındığını kim bilir? | Aracı | Okuyucu (kendi ayracı) |
| Geçmişi tekrar okumak | ❌ Mesaj gitti | ✅ Ayracı geri al |
| Düşünce biçimi | "Bu işi yap" (**görev**) | "Bu oldu" (**gerçek kaydı**) |

> Olayları geçmiş zamanla isimlendirme fikri Kafka'da altyapının merkezinde: Gerçekler silinmez, birikir.

## 2. Temel Kavramlar

| Kavram | Açıklama |
|---|---|
| **Topic** | Mesajların yazıldığı adlandırılmış kayıt defteri (`orders`, `payments`) |
| **Partition** | Topic'in bölümleri. Her biri **ayrı, sıralı** bir kayıt defteri. Ölçeklenme ve sıralamanın anahtarı. |
| **Offset** | Mesajın partition içindeki sıra numarası. Tüketicinin "ayracı." |
| **Broker** | Bir Kafka sunucusu. Birden fazlası **küme** (cluster) oluşturur. |
| **Producer / Consumer** | Mesajı yazan / okuyan |
| **Consumer group** | Aynı işi yapan tüketicilerin grubu (bildirim servisinin 3 kopyası = `notification-service`) |
| **Retention** | Saklama süresi (varsayılan genelde 7 gün). Okunsun okunmasın süre dolunca silinir. |

```
Topic: orders

Partition 0:  [0] [1] [2] [3] [4] [5] [6] →  (yeni mesajlar sona eklenir)
Partition 1:  [0] [1] [2] [3] →
Partition 2:  [0] [1] [2] [3] [4] →
```

- Her partition'ın **kendi** offset sayacı var.

## 3. Okumak: Herkes Kendi Ayracıyla

- **Offset commit:** Tüketici en son işlediği offset'i Kafka'ya bildirir. Yeniden başlayınca oradan devam eder.
- Offset'ler **consumer group başına** tutulur.

```
Topic: orders, Partition 0:  [0] [1] [2] [3] [4] [5] [6]
                                              ▲       ▲
                    analytics-service: 3'te ──┘       │
                    notification-service: 5'te ───────┘
```

- Farklı servisler **aynı defteri** birbirinden habersiz, kendi hızlarında okur. Kopya yok, mesaj silinmez.
- RabbitMQ'daki "her servisin kendi kuyruğu" durumunun karşılığı.

### Replay (Yeniden Oynatma)

| Senaryo | Çözüm |
|---|---|
| 6 ay sonra yeni **öneri servisi** geçmiş siparişleri işlemeli | Yeni consumer group, **en baştan** okur |
| Analitikte hata, son 3 günün raporları yanlış | Hatayı düzelt, offset'i 3 gün öncesine al, **yeniden işle** |

> RabbitMQ'da imkânsız: Teslim edilen mesajlar silinmiş.

## 4. Partition'lar ve Ölçeklenme

Consumer group içinde **her partition aynı anda tam olarak bir tüketiciye** atanır.

```
Topic: orders (3 partition)        Consumer group: notification-service

Partition 0  ──────────────────►  Tüketici A
Partition 1  ──────────────────►  Tüketici B
Partition 2  ──────────────────►  Tüketici C
```

```
2 tüketici:   A ← P0, P1     B ← P2
4 tüketici:   A ← P0   B ← P1   C ← P2   D ← (BOŞTA!)
```

> ⚠️ **Consumer group'taki paralellik partition sayısıyla sınırlıdır.** 3 partition'lı topic için 10 kopya başlatmak işe yaramaz.

- Partition sayısı baştan, beklenen en yüksek paralelliğe göre pay bırakılarak seçilir.
- **Artırılabilir, azaltılamaz.** (Artırmanın sıralama bedeli var, bölüm 5.)

### Rebalance

- Tüketici gruba katılınca / ayrılınca (yeni kopya, çökme, deploy) partition'lar **yeniden dağıtılır.**
- Bu sırada grup kısa süre **okumayı durdurur.**
- Graceful shutdown burada da önemli: Düzgün kapanan tüketici gruptan temiz ayrılır.

## 5. Sıralama: Kafka'nın Gücü

> **Kafka sıralamayı bir partition içinde garanti eder.** Partition'ı grupta tek tüketici okur → Mesajlar tek tek, sırayla işlenir.

### Mesaj Anahtarı (Key)

- Producer her mesajla bir **anahtar** gönderir.
- Kafka anahtardan hash hesaplar → partition seçer.
- **Aynı anahtar her zaman aynı partition'a gider.**

```
Anahtar: "order-42"  ──hash──►  Partition 1:  [placed] [paid] [shipped]    ← sırayla
Anahtar: "order-77"  ──hash──►  Partition 0:  [placed] [cancelled]         ← sırayla
Anahtar: "order-91"  ──hash──►  Partition 1:  ... [placed] ...
```

- **Anahtar = sipariş kimliği** → Bir siparişin tüm olayları **mutlaka sırayla.** Farklı siparişler paralel.
- İhtiyaç: Global sıra değil, **her siparişin kendi içinde** sıra.

> RabbitMQ'daki "iptal oluşturmadan önce işlenebilir" sorununun cevabı.

> ⚠️ **Partition sayısını artırmak:** Anahtar → partition eşlemesi değişir. Geçiş anında o anahtarın eski olayları eski, yenileri yeni partition'da; sıralama bozulabilir.

### ⚠️ Sıcak Partition (Hot Partition)

- Anahtar = müşteri kimliği, bir kurumsal müşteri binlerce sipariş veriyor → Hepsi **tek partition'a** → O tüketici aşırı yüklü, diğerleri boşta.
- İyi anahtar: Hem **sıralama birimini** temsil etmeli (sipariş) hem yükü **dengeli** dağıtmalı.

## 6. Kafka'yı Çalıştırmak

```yaml
  kafka:
    image: apache/kafka:4.0.0
    container_name: shop-kafka
    ports:
      - "9092:9092"
```

- Resmi imaj varsayılan ayarlarla tek broker'lık yerel Kafka başlatır.

> **ZooKeeper:** Eski eğitimlerde görülür. Kafka artık küme yönetimini yerleşik **KRaft** ile yapar; **Kafka 4.0'da ZooKeeper tamamen kaldırıldı.**

### ⚠️ Advertised Listeners Tuzağı

1. İstemci broker'a bağlanır.
2. Broker "beni bundan sonra **şu adresten** bul" der (**advertised listener**).
3. Varsayılan `localhost:9092`. Uygulama da konteynerdeyse kendi konteynerinde arar, **bulamaz.**

- İlk bağlantı başarılı, sonraki her şey başarısız → Kafa karıştırıcı.
- **Çözüm:** Konteyner içi (`kafka:9093`) ve dışı (`localhost:9092`) için **iki ayrı dinleyici.**

> Bağlantı sorununda ilk bakılacak yer: **Advertised listener ayarı.** (Compose'daki `localhost` tuzağının bir adım ileri versiyonu.)

- Görsel araçlar: **Kafka UI**, **AKHQ** (topic, partition, mesaj, consumer group offset'leri).

## 7. Spring ile Kafka

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### Topic Tanımlamak

```java
@Configuration
public class KafkaTopicConfig {

    @Bean
    public NewTopic ordersTopic() {
        return TopicBuilder.name("orders")
                .partitions(6)
                .replicas(1)   // production'da genelde 3
                .build();
    }
}
```

- Spring Boot `NewTopic` bean'lerini başlangıçta oluşturur (varsa dokunmaz).

### Producer

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
```

```java
@Component
public class OrderEventPublisher {

    private final KafkaTemplate<String, OrderPlacedMessage> kafkaTemplate;

    public CompletableFuture<SendResult<String, OrderPlacedMessage>> publish(OrderPlacedMessage message) {
        return kafkaTemplate.send("orders", message.orderId().toString(), message);
        //                         topic     ANAHTAR (sıralama)          mesaj
    }
}
```

- `send` bir **`CompletableFuture`** döner (gönderim arka planda).
- **Outbox postacısı** sonucu beklemeli, kaydı ancak başarıdan **sonra** "gönderildi" yapmalı. (RabbitMQ'daki publisher confirms'ün karşılığı.)

| `acks` | Onay Ne Zaman? | Risk |
|---|---|---|
| `0` | Hiç beklenmez | Mesaj kaybolabilir |
| `1` | Lider kopya yazınca | Lider aktarmadan çökerse kayıp |
| **`all`** | **Tüm güncel kopyalar** yazınca | En güvenli, biraz yavaş |

> **Idempotent producer:** Güncel Kafka'da varsayılan. Ağ sorunu yüzünden tekrar gönderen producer mesajı partition'a **iki kez yazmaz.**
> ⚠️ Sadece producer'ın **kendi** tekrarlarını önler. Outbox postacısının farklı zamanda tekrar göndermesine karşı **tüketici yine idempotent** olmalı.

### Consumer

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: notification-service
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.use.type.headers: false
        spring.json.value.default.type: com.example.notification.messaging.OrderPlacedMessage
```

```java
@Component
public class OrderPlacedConsumer {

    @KafkaListener(topics = "orders", groupId = "notification-service")
    public void handle(OrderPlacedMessage message) {
        // idempotency kontrolü + e-posta gönder
    }
}
```

- `@KafkaListener` = `@RabbitListener` / `@EventListener` karşılığı.
- **Aynı `groupId`** → Kopyalar partition'ları paylaşır. **Farklı `groupId`** → Tüm mesajlar ayrıca okunur.

| `auto-offset-reset` | Grubun kayıtlı offset'i yoksa (ilk çalışma) |
|---|---|
| `earliest` | **En baştan** (saklama süresindeki tüm geçmiş) |
| `latest` | Sadece bundan sonra gelenler |

### ⚠️ Tip Başlığı Tuzağı

- `JsonSerializer` mesaja **producer'daki** sınıfın tam adını başlık olarak ekler.
- Tüketici başlığa güvenirse kendi projesinde o sınıf yok → **okuyamaz.**
- `use.type.headers: false` + `default.type` → "Başlığa bakma, **benim** sınıfıma çevir."

> Redis'teki `@class` ve RabbitMQ'daki sözleşme tartışmasının Kafka versiyonu: **Sözleşme JSON yapısıdır, Java sınıfı değil.**

> Spring Kafka'nın yeni sürümlerinde JSON sınıf adları ve ayarlar değişebilir; sürüm belgelerine bakılmalı.

## 8. Güvenilirlik

### Offset Commit ve Tekrarlar

- Spring, listener metodu **başarıyla bittikten sonra** offset'i bildirir.
- İşlenip offset bildirilmeden çökülürse → Son mesajlar **tekrar** işlenir.

> RabbitMQ'daki "işledi, onay vermeden çöktü" senaryosunun aynısı. **En az bir kez teslim → Idempotent tüketici.** Hangi araç olursa olsun formül değişmez.

### ⚠️ Zehirli Mesaj Partition'ı Kilitler

- Kafka sıralamayı garanti ettiği için offset 5 işlenemezse **offset 6'ya geçilemez.**
- Zehirli mesaj **partition'ın tamamını** kilitler, arkasındaki tüm siparişler bekler.

```java
@Bean
public DefaultErrorHandler kafkaErrorHandler(KafkaTemplate<Object, Object> kafkaTemplate) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate);
    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 2));
    handler.addNotRetryableExceptions(BusinessException.class);
    return handler;
}
```

- Spring Boot bu bean'i dinleyicilere **otomatik** uygular.
- 1 sn aralıkla 2 tekrar (toplam 3) → Hâlâ başarısızsa **`orders.DLT`** topic'ine → Tüketici devam eder, kilit açılır.
- `addNotRetryableExceptions`: Tekrar denemenin işe yaramayacağı hatalar doğrudan DLT'ye.

> DLT = RabbitMQ'daki DLQ'nun karşılığı. **İzlenmeli.**

### Takas: Sıralama mı, Akış mı?

| Yaklaşım | Davranış | Bedel |
|---|---|---|
| **`DefaultErrorHandler`** (bloklayan) | Tekrarlar sürerken partition'daki diğer mesajlar **bekler** | Akış yavaşlar, **sıralama korunur** |
| **`@RetryableTopic`** (bloklamayan) | Başarısız mesaj ayrı "tekrar deneme" topic'lerine taşınır, partition hemen serbest | O mesaj arkasındakilerden **sonra** işlenir, **sıralama bozulur** |

### Ters Serileştirme Hataları

- Mesaj JSON olarak bile çözülemiyorsa hata **listener'a gelmeden** oluşur → Sonsuz döngü.
- **`ErrorHandlingDeserializer`**: Asıl deserializer'ı sarar, hatayı hata işleyiciye iletir → Mesaj DLT'ye gider.
- Production'da kullanılması iyi alışkanlık.

### Kopyalar (Replication)

- Production'da her partition genelde **3 broker'da** kopyalanır (`replicas(3)`).
- Bir kopya **lider** (yazma / okuma), diğerleri **takipçi.**
- Lider çökerse takipçi lider olur, sistem devam eder.
- `acks: all` → Onaylanmış mesaj lider çökse de **kaybolmaz.**

> "Verinin kaybolabileceği varsayımına" Kafka'nın cevabı: **Veri birden fazla makinede.**

## 9. Saklama ve Log Compaction

| Mod | Davranış |
|---|---|
| **Zamana dayalı** (varsayılan) | 7 gün sonra silinir |
| **Boyuta dayalı** | Belirli boyuttan sonra eskiler silinir |
| **Log compaction** | Her **anahtar** için sadece **en son** mesaj tutulur |

**Log compaction örneği:** Ürün kataloğu topic'i, anahtar = ürün kimliği.
- Ürün 42'nin fiyatı 10 kez değişti → Compaction sonrası sadece **son fiyat.**
- Topic bir **tabloya** dönüşür: "Her ürünün güncel hali."
- Başka servis baştan okuyup tüm ürünlerin güncel halini kendi veritabanına / önbelleğine yükleyebilir.

## 10. RabbitMQ mı, Kafka mı?

| | RabbitMQ | Kafka |
|---|---|---|
| **Model** | Kuyruk (posta kutusu) | Kayıt defteri (log) |
| **Okunan mesaj** | Silinir | Saklama süresince kalır |
| **Replay** | ❌ | ✅ |
| **Sıralama** | Rekabet eden tüketicilerde garanti yok | Partition içinde garanti, anahtarla kontrol |
| **Yönlendirme** | Esnek (direct, topic, fanout, desenler) | Basit (topic + anahtar) |
| **Paralellik sınırı** | Tüketici ekledikçe artar | Partition sayısı |
| **Hata yönetimi** | Mesaj bazında ack/nack, DLQ | Zehirli mesaj partition'ı bekletir, DLT |
| **Kapasite** | Yüksek | Çok yüksek (saniyede milyonlarca) |
| **İşletme** | Daha basit | Daha karmaşık (partition, replication, rebalance) |

### RabbitMQ'yu Tercih Et

- Mesajlar **görev** (e-posta gönder, rapor üret, resim küçült)
- İşi tüketiciler arasında **esnekçe dağıtmak**
- **Karmaşık yönlendirme** kuralları
- **Mesaj bazında** onay ve tekrar deneme
- Daha **basit altyapı** işletmek

### Kafka'yı Tercih Et

- Mesajlar **olay akışı**, birçok servis **bağımsız** okuyacak
- **Replay** ihtiyacı
- Bir varlığın olaylarının **sırayla** işlenmesi kritik (siparişin yaşam döngüsü)
- **Çok yüksek hacim:** Tıklama akışları, log toplama, analitik boru hatları, IoT

> Birbirinin alternatifi olmak zorunda değil. Büyük sistemler genelde **ikisini birlikte** kullanır: Olay akışları için Kafka, görev kuyrukları için RabbitMQ.

### ⚠️ Dürüst Not

- Her yeni altyapı parçası = İşletilecek, izlenecek, güncellenecek, anlaşılacak yeni bir şey.
- Retry, DLQ, idempotency, sözleşme evrimi, sıralama, rebalance → Gerçek karmaşıklıklar.

> **Tek bir Spring Boot uygulamasında** Spring Events + transactional outbox çoğu durumda aracıya gerek kalmadan aynı güvenilirliği sağlar.
> Mesaj aracısı, **birden fazla bağımsız servisin** konuşması gerektiğinde gerçek değerini gösterir.

## 11. Test Etmek

```java
@SpringBootTest
@Testcontainers
class OrderPlacedConsumerTest {

    @Container
    @ServiceConnection
    static KafkaContainer kafka = new KafkaContainer("apache/kafka:4.0.0");

    @Autowired private KafkaTemplate<String, OrderPlacedMessage> kafkaTemplate;
    @MockitoBean private EmailService emailService;

    @Test
    void consumesOrderPlacedMessage() {
        kafkaTemplate.send("orders", "42", new OrderPlacedMessage("msg-1", 42L, "ali@example.com"));

        await().atMost(Duration.ofSeconds(10))
                .untilAsserted(() -> verify(emailService).sendConfirmation("ali@example.com", 42L));
    }
}
```

- Awaitility süresi biraz uzun: İlk **rebalance** (gruba katılıp partition alma) birkaç saniye sürebilir.
- **`@EmbeddedKafka`** (`spring-kafka-test`): Docker'sız, bellekte Kafka. Daha hızlı ama gerçek ortama Testcontainers kadar yakın değil (H2 tartışmasının aynısı).

---

## Sorular

**1. RabbitMQ'nun kuyruk modeli ile Kafka'nın log modeli farkı nedir?**
RabbitMQ'da mesaj onaylanınca silinir ve aracı teslimatı takip eder. Kafka'da mesajlar saklama süresince kalır; her consumer group kendi offset'ini tutar ve aynı mesajları bağımsız okur.

**2. Offset nedir, neden consumer group başına tutulur?**
Mesajın partition içindeki sıra numarasıdır. Grup başına tutulduğu için farklı servisler aynı topic'i birbirinden habersiz okuyabilir.

**3. Replay nedir? RabbitMQ'da neden yoktur?**
Offset'i geri alarak geçmiş mesajları yeniden işlemektir. RabbitMQ'da teslim edilen mesajlar silindiği için geçmiş yoktur.

**4. 3 partition'lı topic'i okuyan gruba 5 tüketici eklenirse ne olur?**
3'ü birer partition alır, 2'si boşta kalır. Paralellik partition sayısıyla sınırlıdır.

**5. Rebalance nedir?**
Tüketici gruba katılınca veya ayrılınca partition'ların yeniden dağıtılmasıdır; bu sırada grup kısa süre okumayı durdurur.

**6. Kafka sıralamayı nasıl garanti eder?**
Partition içinde garanti eder. Aynı anahtarlı mesajlar aynı partition'a gider; sipariş kimliği anahtar olursa bir siparişin olayları sırayla işlenir.

**7. Sıcak partition nedir?**
Dengesiz dağılan bir anahtar yüzünden bir partition'ın aşırı yüklenmesidir (örn. çok sipariş veren tek müşterinin kimliği anahtar olunca).

**8. Partition sayısını artırmanın sakıncası nedir?**
Anahtar-partition eşlemesi değişir; geçiş anında o anahtarın sıralama garantisi bozulabilir. Sayı azaltılamaz.

**9. Advertised listener tuzağı nedir?**
Broker istemciye ulaşılacak adresi bildirir. Varsayılan `localhost` ise başka konteynerdeki istemci kendi içinde arar ve bulamaz. İç ve dış için ayrı dinleyiciler gerekir.

**10. `acks: all` ne sağlar?**
Mesaj tüm güncel kopyalara yazılmadan gönderim başarılı sayılmaz; lider çökse bile onaylanmış mesaj kaybolmaz.

**11. Idempotent producer tüketici idempotency'sini gereksiz kılar mı?**
Hayır. Sadece producer'ın kendi ağ kaynaklı tekrarlarını önler; outbox'ın farklı zamanda tekrar göndermesine karşı tüketici idempotent olmalıdır.

**12. `spring.json.use.type.headers: false` hangi sorunu önler?**
Producer'daki sınıf adını taşıyan başlık yüzünden tüketicinin mesajı okuyamamasını; mesaj tüketicinin kendi sınıfına çevrilir.

**13. Kafka'da zehirli mesaj neden daha ciddidir?**
Sıralama garantisi yüzünden işlenemeyen mesaj atlanamaz ve partition'ı kilitler. `DefaultErrorHandler` sınırlı tekrar sonrası `DeadLetterPublishingRecoverer` ile DLT'ye gönderir.

**14. Bloklayan ve bloklamayan retry farkı nedir?**
Bloklayan (`DefaultErrorHandler`) sıralamayı korur ama partition bekler. Bloklamayan (`@RetryableTopic`) partition'ı serbest bırakır ama o mesaj için sıralama bozulur.

**15. `ErrorHandlingDeserializer` neden gereklidir?**
JSON çözülemeyen mesajlarda hata listener'a gelmeden oluşur ve sonsuz döngüye yol açar; bu sarmalayıcı hatayı hata işleyiciye iletir.

**16. Log compaction nedir?**
Her anahtar için sadece en son mesajın tutulduğu saklama modudur; topic güncel değerlerin tablosuna dönüşür.

**17. RabbitMQ ve Kafka ne zaman tercih edilir?**
Görevler, esnek yönlendirme, mesaj bazında hata yönetimi ve basit işletme için RabbitMQ. Bağımsız okunan olay akışları, replay, anahtar bazlı sıralama ve çok yüksek hacim için Kafka.

**18. Tek bir uygulamada mesaj aracısı gerekli midir?**
Çoğu durumda hayır; Spring Events + transactional outbox aynı güvenilirliği sağlar. Aracı, birden fazla bağımsız servis konuştuğunda değer kazanır.
