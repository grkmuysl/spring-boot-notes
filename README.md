# 🌱 Spring Boot Öğrenme Notları

Spring Boot öğrenirken tuttuğum kişisel notlarımın bulunduğu repo.

## Bu Repo Ne?

Spring Boot'u kendi kendime, projeler geliştirerek öğreniyorum. Öğrendiğim her konuyu önce detaylıca çalışıyor, sonra **kendi anladığım şekilde** not alıyorum. Bu repo, o notların bir araya geldiği yer.

Amacım sadece "nasıl yapılır" sorusunun cevabını kaydetmek değil, **"neden böyle yapılır"** sorusunu da anlamak. Bu yüzden notlarda kod örneklerinin yanında, bir özelliğin hangi sorunu çözdüğünü ve arka planda nasıl çalıştığını da bulacaksın.

Notlar, Spring'in çekirdeğinden (IoC, bean'ler) başlayıp production'a hazır bir uygulamanın ihtiyaç duyduğu konulara (test, Docker, gözlemlenebilirlik, mesajlaşma) ve en sonunda mikroservis mimarisine kadar uzanıyor.

## İçindekiler

### 🧱 Temeller

| Konu | İçerik |
|---|---|
| IoC ve Dependency Injection | Sıkı bağlılık, DI, IoC container, enjeksiyon türleri |
| @Primary, @Qualifier, @Configuration | Aynı tipte birden fazla bean, kütüphane sınıflarını bean yapmak |
| Bean Scope ve Yaşam Döngüsü | Singleton, prototype, thread güvenliği, `@PostConstruct` |
| REST Controller ve Katmanlı Mimari | Controller, service, repository, DTO, REST kuralları |
| Exception Handling ve Validation | `@RestControllerAdvice`, Bean Validation, hata formatı |

### 🗄️ Veri Erişimi

| Konu | İçerik |
|---|---|
| Spring Data JPA | Entity, repository, sorgular, sayfalama, transaction, dirty checking |
| Entity İlişkileri ve Proxy | İlişkiler, cascade, lazy loading, N+1 problemi, proxy kavramı |
| PostgreSQL, Docker ve Flyway | Docker temelleri, Compose, migration ile şema yönetimi |

### 🔐 Güvenlik

| Konu | İçerik |
|---|---|
| Spring Security: Temeller | Filter chain, authentication, authorization, BCrypt, roller |
| Spring Security: JWT | Stateless kimlik doğrulama, token üretimi ve doğrulama filtresi |

### 🧪 Test

| Konu | İçerik |
|---|---|
| Test: Temeller | Test piramidi, JUnit 5, Mockito, test double'lar |
| Test: Controller | `@WebMvcTest`, MockMvc, security testleri |
| Test: Repository ve Entegrasyon | `@DataJpaTest`, Testcontainers, `@SpringBootTest` |

### 🚀 Production'a Hazırlık

| Konu | İçerik |
|---|---|
| Actuator | Health check, liveness / readiness, metrikler, Micrometer |
| API Dokümantasyonu | OpenAPI, Swagger UI, springdoc |
| Caching: Spring Cache | `@Cacheable`, cache invalidation, Caffeine |
| Caching: Redis | Dağıtık önbellek, serileştirme, graceful degradation |
| Dış Servis Çağırma | `RestClient`, hata yönetimi, timeout'lar |
| Dayanıklılık | Retry, idempotency, circuit breaker, fallback |
| Asenkron İşler | Spring Events, `@TransactionalEventListener`, `@Async` |
| Zamanlanmış Görevler | `@Scheduled`, ShedLock, transactional outbox |

### 🏗️ Dağıtım ve Mimari

| Konu | İçerik |
|---|---|
| Docker ile Paketleme | Dockerfile, multi-stage build, konteynerde JVM |
| GitHub Actions ile CI | Otomatik test, branch protection, imaj yayınlama |
| Mesajlaşma: RabbitMQ | Exchange, kuyruk, ack, dead letter queue |
| Mesajlaşma: Kafka | Topic, partition, offset, sıralama, RabbitMQ ile karşılaştırma |
| Mikroservisler: Mimari | Ne zaman bölünür, modüler monolit, bounded context, saga |
| Mikroservisler: Altyapı | Spring Cloud, API gateway, servis keşfi, dağıtık izleme |

## Notların Yapısı

Her not genel olarak şu sırayı takip eder:

- **Sorun:** Bu konu neden var, hangi problemi çözüyor?
- **Çözüm ve çalışma mantığı:** Spring bunu arka planda nasıl yapıyor?
- **Kod örnekleri:** Kavramı somutlaştıran kısa, odaklı örnekler
- **Sık yapılan hatalar:** Gözden kaçan tuzaklar ve dikkat edilmesi gerekenler
- **Bilgilendirici sorular:** Konuyu ne kadar anladığımı test etmek için sorular ve cevapları

## Nasıl Kullanılır?

Notlar, konular birbirinin üzerine inşa edilecek şekilde sıralanmıştır. Spring'e yeni başlıyorsan baştan sona sırayla okumanı öneririm; belirli bir konuya bakıyorsan doğrudan ilgili dosyaya gidebilirsin.

Notlarda geçen örnekler büyük ölçüde aynı senaryo üzerinden ilerler: ürün, kategori, sipariş ve kullanıcılardan oluşan bir e-ticaret API'si. Böylece her yeni konu, bir öncekinin üzerine eklenmiş olur.

## Tekrar Tekrar Karşılaşılan Fikirler

Notları okurken bazı fikirlerin farklı konularda, farklı kılıklarla yeniden ortaya çıktığını göreceksin. Bunları anlamak, tek tek özellikleri ezberlemekten çok daha değerli:

- **Proxy:** `@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize`, Spring Data repository'leri ve lazy loading aynı mekanizmayla çalışır. Self-invocation tuzağı da hepsinde ortaktır.
- **ThreadLocal:** Transaction, `SecurityContextHolder` ve MDC, bilgiyi thread'e bağlı tutar. İş başka bir thread'e geçtiğinde bu bilgiler geride kalır.
- **Arayüz + değiştirilebilir implementasyon:** JPA / Hibernate, SLF4J / Logback, Micrometer, Spring Cache, OpenTelemetry.
- **Varsayılan olarak kapalı:** Spring Security, Actuator endpoint'leri ve güvenli yapılandırma varsayılanları.
- **Idempotency:** Retry, outbox, mesaj tüketicileri ve sagaların güvenle çalışabilmesinin ön koşulu.
- **Dışarıdan verilen bağımlılıklar:** Constructor injection'dan ortam değişkenlerine, hatta enjekte edilen `Clock`'a kadar; test edilebilirliğin temeli.

## Teknolojiler

- **Dil ve çatı:** Java 21, Spring Boot 3
- **Veri:** Spring Data JPA / Hibernate, PostgreSQL, Flyway, Redis
- **Güvenlik:** Spring Security, JWT
- **Test:** JUnit 5, Mockito, AssertJ, Testcontainers
- **Dayanıklılık ve gözlemlenebilirlik:** Resilience4j, Actuator, Micrometer, OpenTelemetry
- **Mesajlaşma:** RabbitMQ, Apache Kafka
- **Altyapı:** Docker, Docker Compose, GitHub Actions, Spring Cloud
- **Derleme:** Maven

## Not

Bu notlar öğrenme sürecimin bir parçası olduğu için hata veya eksik içerebilir. Bir hata fark edersen ya da bir öneri varsa issue açabilir veya pull request gönderebilirsin. Her geri bildirim öğrenmeme katkı sağlar. 🙂

## İlerleme

Repo, öğrendikçe büyümeye devam ediyor. 🚀
