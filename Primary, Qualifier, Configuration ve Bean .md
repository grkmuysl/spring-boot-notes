# Birden Fazla Bean: @Primary, @Qualifier, @Configuration ve @Bean

## Sorun: Aynı Tipte İki Bean

```java
@Service
public class EmailService implements NotificationService { ... }

@Service
public class SmsService implements NotificationService { ... }

@Service
public class OrderService {
    public OrderService(NotificationService notificationService) { ... }
}
```

Spring `NotificationService` için iki aday bulur, hangisini seçeceğini bilemez ve uygulama **ayağa kalkmaz**:

```
NoUniqueBeanDefinitionException: expected single matching bean but found 2: emailService,smsService
```

- **Bean ismi:** Spring her bean'e varsayılan olarak sınıf adının ilk harfi küçültülmüş halini isim olarak verir (`EmailService` → `emailService`).
- **Kardeş hata:** Hiç aday bulunamazsa `NoSuchBeanDefinitionException` alınır. Genelde `@Service` yazmayı unutmaktan veya sınıfın component scan dışındaki bir pakette kalmasından kaynaklanır.

## @Primary: Varsayılanı Belirlemek

```java
@Service
@Primary
public class EmailService implements NotificationService { ... }
```

Belirsizlik durumunda Spring `@Primary` olanı seçer. "Aksi belirtilmedikçe bunu kullan" demektir.

- Kararı **bean'i tanımlayan taraf** verir.
- Uygulamanın çoğu bir implementasyonu kullanıyorsa mantıklıdır.

## @Qualifier: Hangisini İstediğini Açıkça Söylemek

```java
@Service
public class PasswordResetService {
    public PasswordResetService(@Qualifier("smsService") NotificationService ns) { ... }
}
```

- Kararı **bean'i kullanan taraf** verir.
- `@Primary` ve `@Qualifier` farklı bean'leri gösterirse **`@Qualifier` kazanır**, çünkü daha spesifiktir.
- Yaygın kalıp: `@Primary` ile genel varsayılanı belirle, istisnai yerlerde `@Qualifier` ile özelleştir.

> **Dikkat:** `@Qualifier` içindeki isim düz bir String'dir. Sınıf adı değişirse bean ismi de değişir ve derleyici uyarmaz, hata uygulama başlarken alınır. Sabit isim vermek için: `@Service("sms")`

### Primary Varken Qualifier Yazmak Gereksiz mi?

```java
@Service @Primary
public class EmailService implements NotificationService { ... }

@Service
public class PasswordResetService {
    public PasswordResetService(@Qualifier("emailService") NotificationService ns) { ... }
}
```

`@Qualifier` silinirse **şu anki sonuç** değişmez, yine `EmailService` gelir. Ama ileride `@Primary` başka bir sınıfa taşınırsa `PasswordResetService` de sessizce değişir, hata alınmaz.

Asıl soru: Bu sınıfın e-posta kullanması **tesadüf** mü, **bilinçli bir karar** mı? Bilinçli bir karar ise `@Qualifier` ile koda yazmak daha güvenlidir.

## Hepsini Birden İstemek

```java
@Service
public class OrderService {
    private final List<NotificationService> notificationServices;

    public OrderService(List<NotificationService> notificationServices) {
        this.notificationServices = notificationServices;
    }

    public void placeOrder(String customer) {
        notificationServices.forEach(n -> n.send(customer, "Siparişiniz alındı"));
    }
}
```

Spring o tipteki **tüm** bean'leri listeye koyar. Yeni bir implementasyon eklendiğinde `OrderService`'e dokunmadan listeye dahil olur.

## @Configuration ve @Bean: Kendi Yazmadığın Sınıflar

Kütüphaneden gelen bir sınıfın (`ObjectMapper`, `RestTemplate` vb.) kaynak koduna `@Component` yazamayız. Bu durumda bean bir konfigürasyon sınıfında metotla tanımlanır:

```java
@Configuration
public class AppConfig {

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        return mapper;
    }
}
```

- `@Configuration`: Bu sınıf bean tanımları içerir.
- `@Bean`: Metodun döndürdüğü nesne container'a bean olarak konur. Metodu Spring çağırır.
- **Bean ismi metodun adıdır** (`objectMapper`).
- Burada `new` kullanmak DI'a aykırı değildir: Nesneyi yine container yönetir, biz sadece **nasıl oluşturulacağının tarifini** veririz. Kullanan sınıflar hâlâ constructor'dan alır.

`@Bean` metotları parametre alabilir, Spring bunları container'dan bulup verir:

```java
@Bean
public ReportService reportService(ObjectMapper objectMapper) {
    return new ReportService(objectMapper);
}
```

`@Primary` ve `@Qualifier`, `@Bean` metotlarıyla da aynı şekilde çalışır.

## Hangisi Ne Zaman?

| Durum | Kullanılacak |
|---|---|
| Kendi yazdığın sınıf | `@Component`, `@Service`, `@Repository`, `@Controller` |
| Kütüphaneden gelen sınıf | `@Configuration` + `@Bean` |
| Oluşturulması ayar gerektiren nesne | `@Configuration` + `@Bean` |
