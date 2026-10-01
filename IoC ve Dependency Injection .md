# IoC ve Dependency Injection

## Dependency Injection (DI)

Sınıfın ihtiyaç duyduğu nesneyi kendisinin oluşturması yerine dışarıdan verilmesidir.

Bağımlılığın dışarıdan verilmemesinin birkaç sorunu var:

- **Sıkı bağlılık (tight coupling):** Sınıf, bağımlılığını kendisi seçip oluşturduğu için bağımlılık değiştiğinde (örneğin e-posta yerine SMS) sınıfın kodunu açıp değiştirmek gerekir.
- **Test edilmesi zor:** Sınıf nesneyi kendi içinde oluşturduğu için testte sahte (mock) bir nesne vermenin temiz bir yolu yoktur.
- **Gizli bağımlılık:** Sınıfın neye ihtiyaç duyduğu dışarıdan görünmez, ancak kodun içi okunarak anlaşılır.

### Önce: Bağımlılığı kendisi oluşturuyor

```java
public class OrderService {
    private EmailService emailService = new EmailService();

    public void placeOrder(String customer) {
        emailService.send(customer, "Siparişiniz alındı");
    }
}
```

Burada `OrderService` iki kararı kendisi veriyor:
1. **Hangi implementasyonun** kullanılacağı (e-posta mı, SMS mi)
2. Nesnenin **ne zaman ve nasıl oluşturulacağı**

Spring kullanıldığında bu iki karar da container'a geçer.

### Sonra: Bağımlılık dışarıdan geliyor

```java
public class OrderService {
    private final NotificationService notificationService;

    public OrderService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    public void placeOrder(String customer) {
        notificationService.send(customer, "Siparişiniz alındı");
    }
}
```

`OrderService` artık sadece "bildirim gönderebilen bir şeye" ihtiyaç duyuyor. E-posta mı SMS mi olduğunu bilmiyor.

## Inversion of Control (IoC)

Nesneleri oluşturup birbirlerine bağlama kontrolünü bir container'a (Spring'de `ApplicationContext`) devretmektir. ApplicationContext'in oluşturup yönettiği nesnelere **Bean** denir.

Kısaca: **IoC kontrolü framework'e verme prensibi, DI ise bu prensibin uygulanma yöntemidir (bağımlılıkların dışarıdan verilmesi).**

> Not: DI bir tasarım prensibidir, Spring'e özel değildir. Spring olmadan da constructor'dan bağımlılık vererek DI uygulanabilir.

## Spring Bunu Nasıl Yapıyor?

Uygulama ayağa kalkarken Spring:

1. `@SpringBootApplication` anotasyonunun olduğu pakete bakar.
2. Alt paketleri tarayarak `@Component`, `@Service`, `@Repository` ve `@Controller` anotasyonlu sınıfları bulur. (Bu anotasyonlar `@Component` anotasyonunun özelleşmiş halleridir.)
3. Bulduğu sınıflardan **bean'ler oluşturur**.
4. Her bean'in constructor'ına bakıp neye ihtiyaç duyduğunu görür, uygun bean'i bulup **içeri verir**. DI'ın Spring'de gerçekleştiği yer bu adımdır.

> Sınıfta **tek bir constructor** varsa Spring onu otomatik kullanır, `@Autowired` yazmaya gerek yoktur (Spring 4.3'ten beri). Eski kodlarda her yerde `@Autowired` görülmesinin sebebi budur.

## Enjeksiyon Türleri

Spring'de bağımlılık eklemenin 3 yolu var:

```java
// 1. Constructor injection (önerilen)
public OrderService(NotificationService ns) { this.ns = ns; }

// 2. Setter injection
@Autowired
public void setNotificationService(NotificationService ns) { this.ns = ns; }

// 3. Field injection (kaçınılmalı)
@Autowired
private NotificationService ns;
```

### Neden Constructor Injection?

- Bağımlılıklar `final` olabilir, nesne oluştuktan sonra değiştirilemez.
- Sınıfın neye ihtiyaç duyduğu constructor'da açıkça görünür.
- Spring olmadan test edilebilir: `new OrderService(mockNotification)`
- Constructor çok fazla parametre almaya başlarsa bu, sınıfın çok fazla iş yaptığının işaretidir.

### Neden Field Injection'dan Kaçınılmalı?

Alan `final` olamaz, bağımlılıklar gizli kalır ve sınıfı Spring olmadan test etmek için reflection gerekir.

**Setter injection** ise bağımlılık isteğe bağlı olduğunda veya sonradan değiştirilebilmesi gerektiğinde kullanılır, pratikte nadirdir.
