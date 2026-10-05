# Actuator: Health, Endpoint'ler ve Metrikler

## 1. Observability (Gözlemlenebilirlik)

Loglar **"ne oldu?"** sorusunu cevaplar. Production'da **"şu an durum ne?"** sorusu da sorulur:
- Uygulama ayakta mı? Veritabanına bağlanabiliyor mu?
- Saniyede kaç istek geliyor, cevap süresi ne?
- Bellek veya bağlantı havuzu doluyor mu?
- Production'da hangi sürüm çalışıyor?

| Ayak | Soru | Araç |
|---|---|---|
| **Loglar** | Ne oldu? | SLF4J + Logback |
| **Metrikler** | Ne kadar, ne sıklıkta, ne hızla? | Micrometer |
| **İzler (traces)** | Bir istek hangi yollardan geçti? | Micrometer Tracing |

## 2. Kurulum

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

- Endpoint'ler `/actuator` altında açılır.
- HTTP üzerinden varsayılan olarak **sadece `health`** açıktır.

> "Varsayılan olarak kapalı" ilkesi: Diğer endpoint'ler iç detayları ve gizli bilgileri açığa çıkarabilir. Neyin açılacağına açıkça karar verilir.

## 3. Health

```
GET /actuator/health  →  { "status": "UP" }
```

- Actuator, uygulamadaki bileşenler için otomatik **health indicator**'lar oluşturur (`db`, `diskSpace`, `ping`).
- `db` göstergesi veritabanına **gerçekten** bağlanıp kontrol eder.
- **Biri bile `DOWN` ise** genel durum `DOWN` olur ve endpoint **503 Service Unavailable** döner.

```yaml
management:
  endpoint:
    health:
      show-details: always   # geliştirme
```

| Değer | Detaylar Kime Görünür? |
|---|---|
| `never` (varsayılan) | Kimseye |
| `when-authorized` | Kimliği doğrulanmış kullanıcılara (**production için**) |
| `always` | Herkese (sadece geliştirme) |

> Detaylar veritabanı türü, sürümü gibi saldırgan için değerli bilgiler içerir.

**Kim kullanır?** Çoğunlukla makineler: Yük dengeleyiciler, Docker healthcheck, Kubernetes.

### Liveness ve Readiness

| Probe | Soru | Başarısızsa |
|---|---|---|
| **Liveness** | Uygulama takılıp kaldı mı? | Platform uygulamayı **yeniden başlatır** |
| **Readiness** | Trafik almaya hazır mı? | Platform **trafiği keser**, yeniden başlatmaz |

```
GET /actuator/health/liveness
GET /actuator/health/readiness
```

- Kubernetes'te otomatik açılır. Diğer ortamlarda: `management.endpoint.health.probes.enabled: true`

> ⚠️ **Veritabanı kontrolü readiness'a girer, liveness'a değil.**
> Veritabanı çöktüğünde uygulamaları yeniden başlatmak sorunu çözmez. Üstelik veritabanı geri geldiğinde tüm uygulamalar aynı anda bağlanıp onu tekrar çökertebilir.
> Doğrusu: "Trafik alamam" (readiness DOWN), ama "ölüyüm" değil (liveness UP). Spring Boot varsayılan olarak bunu doğru yapar.

### Özel Health Indicator

```java
@Component
public class PaymentServiceHealthIndicator implements HealthIndicator {

    private final PaymentClient paymentClient;

    public PaymentServiceHealthIndicator(PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }

    @Override
    public Health health() {
        try {
            paymentClient.ping();
            return Health.up().build();
        } catch (Exception ex) {
            return Health.down()
                    .withDetail("error", "Ödeme servisine ulaşılamıyor")
                    .build();
        }
    }
}
```

- `HealthIndicator` uygulayan bir bean yeterli; Actuator bulup `paymentService` adıyla ekler (`HealthIndicator` soneki atılır).
- `withDetail`'e exception mesajı değil **genel açıklama** yazılır (iç detay sızdırma).

## 4. Diğer Endpoint'ler

| Endpoint | Gösterdiği |
|---|---|
| `info` | Sürüm, build zamanı, git commit |
| `metrics` | Metrikler |
| `loggers` | Log seviyeleri (**çalışma anında değiştirilebilir**) |
| `mappings` | Tüm URL → controller eşlemeleri |
| `beans` | Context'teki tüm bean'ler |
| `configprops` | Tüm `@ConfigurationProperties` değerleri |
| `env` | Tüm yapılandırma kaynakları ve değerleri |
| `threaddump` | Tüm thread'lerin anlık durumu |
| `heapdump` | **Belleğin tam kopyası** |
| `prometheus` | Metrikler, Prometheus formatında |

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,loggers
```

> ⚠️ **`include: "*"` production'da asla.**
> - **`env`, `configprops`:** Şifreler ve secret'lar dahil tüm ayarlar. Spring Boot 3 varsayılan olarak `******` ile maskeler, ama maskeleme kapatılabilir.
> - **`heapdump`:** Bellekteki her şey: JWT secret, veritabanı şifresi, işlenen isteklerdeki kullanıcı şifreleri ve token'lar. Gerçek dünyada sistemlerin ele geçirilmesine yol açmış bilinen bir açık.

## 5. Actuator'ı Korumak

### Ayrı Port

```yaml
management:
  server:
    port: 9090
```

- `/api/...` → 8080 (internete açık), `/actuator/...` → 9090 (sadece iç ağ).
- Docker'da sadece 8080 dışarıya eşlenir.

### Security Kuralları

```java
.authorizeHttpRequests(auth -> auth
        .requestMatchers("/actuator/health/**").permitAll()   // önce spesifik
        .requestMatchers("/actuator/**").hasRole("ADMIN")     // sonra genel
        .anyRequest().authenticated())
```

- Health herkese açık: Yük dengeleyiciler token göndermez.
- ⚠️ **Sıra ters olursa** health de admin ister, yük dengeleyici 401 alır, uygulamayı sağlıksız sanıp **trafiği keser.**

## 6. loggers: Log Seviyesini Çalışırken Değiştirmek

```
GET /actuator/loggers/com.example.shop
→ { "configuredLevel": null, "effectiveLevel": "INFO" }

POST /actuator/loggers/com.example.shop
{ "configuredLevel": "DEBUG" }
```

- **Yeniden başlatmadan** hemen etkili olur. Sorun araştırırken kullanılır.
- İş bitince `{ "configuredLevel": null }` ile **eski haline döndür.** DEBUG production'da performansı düşürür, log maliyetini artırır.
- Sadece adminlere açık olmalı: Log seviyesini değiştirebilen biri sistemi gereksiz loglarla boğabilir.

## 7. info: Production'da Ne Çalışıyor?

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>build-info</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

```json
{
  "build": { "artifact": "shop", "version": "1.4.2", "time": "2026-10-05T11:20:45.000Z" }
}
```

- "Hata dünkü sürümde mi vardı, bugünkü deploy ile mi geldi?" sorusunu cevaplar.
- `git-commit-id-maven-plugin` ile Git commit bilgisi de eklenir.

## 8. Metrikler ve Micrometer

> **Micrometer = "Metrikler için SLF4J."** Kod Micrometer arayüzlerine bağımlı; metriklerin gideceği sistem (Prometheus, Datadog, CloudWatch) ayrı modülle belirlenir, kod değişmeden değiştirilir.

```
GET /actuator/metrics                                             → tüm metrik isimleri
GET /actuator/metrics/http.server.requests?tag=uri:/api/products  → istek sayısı, süre
GET /actuator/metrics/http.server.requests?tag=status:500         → sadece hatalar
```

Hiç kod yazmadan her endpoint için `COUNT`, `TOTAL_TIME`, `MAX` gelir.

| Metrik | Ne Söyler? |
|---|---|
| `http.server.requests` | Endpoint bazında istek sayısı ve süreleri |
| `jvm.memory.used` | Bellek kullanımı. Sürekli artıyorsa sızıntı olabilir. |
| `jvm.threads.live` | Canlı thread sayısı |
| `hikaricp.connections.active` | Kullanımdaki veritabanı bağlantısı |
| `hikaricp.connections.pending` | **Bağlantı bekleyen** istek sayısı |

### Bağlantı Havuzu (HikariCP)

- Veritabanı bağlantıları pahalıdır; sınırlı sayıda (varsayılan **10**) bağlantı havuzda tutulup yeniden kullanılır.
- Bağlantılar uzun tutulursa (örn. **Open Session in View** açık) havuz dolar, `pending` artar, uygulama yavaşlar.

> Metrikler, soyut olarak konuşulan sorunları somut sayılara dönüştürür.

### Metrik Türleri

| Tür | Ne Ölçer? | Örnek |
|---|---|---|
| **Counter** | Sadece artan sayı | Oluşturulan sipariş sayısı |
| **Gauge** | Anlık, artıp azalabilen değer | Bekleyen sipariş sayısı, kuyruk uzunluğu |
| **Timer** | İşlem süresi ve sayısı | Ödeme servisi cevap süresi |

### Kendi Metriklerin

```java
@Service
public class OrderService {

    private final MeterRegistry meterRegistry;

    @Transactional
    public OrderResponse placeOrder(OrderRequest request, String paymentMethod) {
        // ... sipariş oluşturma
        meterRegistry.counter("orders.created", "payment_method", paymentMethod).increment();
        return toResponse(order);
    }
}
```

- `MeterRegistry` Actuator'ın sağladığı bir bean, constructor'dan alınır.
- **Tag:** Metriği boyutlara ayırır. `/actuator/metrics/orders.created?tag=payment_method:credit_card`

### ⚠️ Kardinalite Tuzağı

```java
meterRegistry.counter("orders.created", "user_id", userId).increment();  // YAPMA
```

- Micrometer **her farklı tag değeri için ayrı sayaç** tutar.
- 100.000 kullanıcı = 100.000 sayaç → bellek tükenir (**yüksek kardinalite**).
- **Tag değerleri sınırlı sayıda olmalı:** Ödeme yöntemi ✅, HTTP durum kodu ✅, kullanıcı / sipariş / e-posta ❌

> **Metrikler** toplamları ve eğilimleri gösterir. **Loglar** tek tek olayları gösterir.

## 9. Prometheus ve Grafana

`/actuator/metrics` sadece **anlık** değeri gösterir, geçmişi tutmaz.

| Araç | Görevi |
|---|---|
| **Prometheus** | Uygulamalardan düzenli aralıklarla metrikleri **çeker**, zaman serisi olarak saklar |
| **Grafana** | Grafikler, panolar ve uyarılar ("hata oranı %5'i geçerse bildir") |

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <scope>runtime</scope>
</dependency>
```

- `prometheus` endpoint'i açılır → `/actuator/prometheus`
- Yerelde denemek için Prometheus ve Grafana `compose.yaml`'a servis olarak eklenebilir.
- Datadog'a geçmek = bu bağımlılığı değiştirmek. Sayaç kodu aynen kalır.

## 10. İzler (Kısaca)

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
```

- Her isteğe bir **`traceId`** verilir ve otomatik olarak **MDC**'ye konur, loglarda görünür.
- Başka servise atılan HTTP isteklerinin başlıklarında **taşınır**; istek beş servisten geçse bile tek kimlikle takip edilir.
- Her adımın süresi kaydedilir, Zipkin veya Grafana Tempo ile zaman çizelgesi olarak görülür.
- Mikroservislerde vazgeçilmez. Elle yazılan `RequestIdFilter` bu işin el yapımı versiyonudur.

---

## Örnek Sorular

**1. Observability'nin üç ayağı nedir?**
Loglar (ne oldu?), metrikler (ne kadar, ne hızla?) ve izler (istek hangi yollardan geçti?).

**2. Actuator neden varsayılan olarak sadece `health`'i açar?**
Diğer endpoint'ler iç detayları ve gizli bilgileri açığa çıkarabilir; neyin açılacağına açıkça karar verilmelidir.

**3. Health endpoint'i ne zaman 503 döner?**
Herhangi bir health indicator `DOWN` olduğunda genel durum `DOWN` olur ve 503 döner.

**4. Liveness ve readiness farkı nedir? Veritabanı kontrolü hangisine girer?**
Liveness başarısızsa uygulama yeniden başlatılır, readiness başarısızsa sadece trafik kesilir. Veritabanı kontrolü readiness'a girer; yeniden başlatmak veritabanı sorununu çözmez.

**5. Özel health indicator nasıl yazılır?**
`HealthIndicator` arayüzünü uygulayan bir bean yazılır; Actuator onu otomatik bulur ve sağlık kontrolüne ekler.

**6. `heapdump` endpoint'i neden tehlikelidir?**
Belleğin tam kopyasını verir; secret'lar, şifreler ve token'lar içinde bulunabilir.

**7. Actuator endpoint'leri nasıl korunur?**
Ayrı bir porttan (`management.server.port`) sunulup sadece iç ağa açılır ve/veya Spring Security kurallarıyla health herkese, diğerleri adminlere açılır.

**8. Security'de `/actuator/**` kuralı health kuralından önce yazılırsa ne olur?**
Health de admin ister; yük dengeleyici 401 alır, uygulamayı sağlıksız sanıp trafiği keser.

**9. Production'da log seviyesi yeniden başlatmadan nasıl değiştirilir?**
`POST /actuator/loggers/{paket}` ile `{"configuredLevel": "DEBUG"}` gönderilir; iş bitince `null` ile geri alınır.

**10. Micrometer nedir?**
Metrikler için bir arayüz (facade). Kod Micrometer'a bağımlıdır, metriklerin gideceği sistem kod değişmeden değiştirilebilir.

**11. `hikaricp.connections.pending` artıyorsa ne anlama gelir?**
İstekler havuzdan boş bağlantı beklemektedir; havuz doludur. Uzun tutulan bağlantılar (Open Session in View) veya küçük havuz buna yol açar.

**12. Counter, gauge ve timer farkı nedir?**
Counter sadece artar, gauge anlık ve artıp azalabilir, timer süre ve sayıyı ölçer.

**13. Metriklere `user_id` tag'i eklemek neden yanlıştır?**
Her farklı değer için ayrı sayaç tutulur; sınırsız değerler belleği tüketir (yüksek kardinalite). Tek tek kayıtlar loglara aittir.

**14. `/actuator/metrics` varken neden Prometheus gerekir?**
Actuator sadece anlık değeri gösterir. Prometheus metrikleri düzenli çekip zaman serisi olarak saklar; geçmişe dönük analiz ve uyarı mümkün olur.

**15. Micrometer Tracing'in MDC ile ilişkisi nedir?**
Her isteğe verdiği `traceId`'yi otomatik olarak MDC'ye koyar, böylece tüm log satırlarında görünür. Ayrıca bu kimliği servisler arası isteklerde taşır.
