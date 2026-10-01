# Bean Scope'ları ve Yaşam Döngüsü

## Singleton (Varsayılan)

Container başına **tek nesne** oluşturulur. Bean'i isteyen herkes aynı nesneyi alır.

- Java'daki Singleton tasarım deseniyle aynı şey değildir. Spring singleton'ı "bu container içinde bir tane" demektir; sınıf sıradan bir sınıftır.
- Servisler, repository'ler ve controller'lar genelde durumsuz (stateless) olduğu için varsayılan budur.

### Singleton'da State Tutmak

Her HTTP isteği ayrı thread'de çalışır ama hepsi **aynı nesneyi** kullanır.

```java
@Service
public class OrderService {
    private String currentCustomer; // YANLIŞ: thread'ler birbirinin verisini ezer

    public void placeOrder(String customer) {
        this.currentCustomer = customer;
        sendConfirmation(currentCustomer); // Ali'nin onayı Ayşe'ye gidebilir
    }
}
```

**Kurallar:**
- İsteğe özel veri field'da tutulmaz, metot parametreleri ve yerel değişkenlerle taşınır.
- Enjekte edilen bağımlılıklar ve değişmeyen ayarlar field olabilir.
- Gerçekten paylaşılan veri tutuluyorsa thread-safe olmalıdır:

```java
private int count = 0;
count++;               // YANLIŞ: oku-ekle-yaz, istekler kaybolabilir

private final AtomicInteger count = new AtomicInteger();
count.incrementAndGet(); // DOĞRU: bölünemez tek işlem
```

> Bu hatalar tek istekle test ederken ortaya çıkmaz, gerçek trafik altında rastgele patlar.

## Prototype

Bean her istendiğinde **yeni nesne** oluşturulur.

```java
@Component
@Scope("prototype")
public class ShoppingCart { ... }
```

### Tuzak: Singleton İçinde Prototype

Singleton bir kez oluşturulduğu için içine enjekte edilen prototype da **bir kez** enjekte edilir. Herkes aynı nesneyi paylaşır.

**Çözüm:** Nesneyi değil, üreticisini enjekte et:

```java
private final ObjectProvider<ShoppingCart> cartProvider;

ShoppingCart cart = cartProvider.getObject(); // her çağrıda yeni nesne
```

> Pratikte prototype'a nadiren gerek olur. Genelde metot içinde `new` ile oluşturmak veya veriyi veritabanında tutmak daha doğrudur.

## Web Scope'ları

| Scope | Ömür |
|---|---|
| `request` | Her HTTP isteği için yeni nesne |
| `session` | Her kullanıcı oturumu için bir nesne |
| `application` | Tüm uygulama için bir nesne |

## Yaşam Döngüsü

1. **Oluşturma:** Constructor çağrılır (constructor injection burada olur)
2. **Enjeksiyon:** Setter ve field enjeksiyonları yapılır
3. **Başlatma:** `@PostConstruct` çalışır
4. **Kullanım:** Bean hazırdır
5. **Yok etme:** Uygulama kapanırken `@PreDestroy` çalışır

```java
@PostConstruct   // kurulumdan sonra: cache yükleme, başlangıç kontrolleri
public void init() { ... }

@PreDestroy      // yok edilmeden önce: bağlantıları kapatma, kaynakları bırakma
public void cleanup() { ... }
```

- Spring Boot 3'te bu anotasyonlar `jakarta.annotation` paketindedir (eskiden `javax.annotation`).
- `@PostConstruct` **tüm** enjeksiyonlar bittikten sonra çalışır. Field injection kullanılan kodda constructor'da bağımlılıklar henüz `null`'dır.
- **Prototype bean'lerde `@PreDestroy` çağrılmaz.** Spring nesneyi verdikten sonra takibini bırakır.
- Kütüphane sınıflarında aynı iş için: `@Bean(initMethod = "start", destroyMethod = "close")`

## Eager ve Lazy Oluşturma

- Singleton'lar **uygulama başlarken** oluşturulur. Hata varsa uygulama hiç ayağa kalkmaz (fail-fast). Bu iyi bir şeydir: Hata deploy anında görülür.
- `@Lazy` ile bean ilk kullanıldığı ana kadar oluşturulmaz, ama bu hataları geç yakalamaya yol açar.
