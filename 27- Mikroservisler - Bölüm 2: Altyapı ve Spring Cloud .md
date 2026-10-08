# Mikroservisler - Bölüm 2: Altyapı ve Spring Cloud

## 0. Spring Cloud Nedir?

**Spring Cloud**, tek bir kütüphane değil, **dağıtık sistemlerin ortak sorunlarına** hazır çözümler sunan **alt projelerin şemsiye adıdır.**

| Proje | Amacı |
|---|---|
| **Spring Boot** | **Tek bir** uygulamayı kolayca ayağa kaldırmak |
| **Spring Cloud** | **Birden fazla** uygulamanın birlikte çalışmasını kolaylaştırmak |

> Spring Cloud, Spring Boot'un **üzerine** kurulur; onun yerine geçmez. Her mikroservis yine bir Spring Boot uygulamasıdır.

### Alt Projeler

| Alt Proje | Çözdüğü Sorun |
|---|---|
| **Spring Cloud Gateway** | Tüm dış isteklerin tek giriş kapısı, yönlendirme |
| **Spring Cloud Netflix Eureka** | Servis keşfi (kayıt defteri) |
| **Spring Cloud LoadBalancer** | İstemci tarafında kopyalar arasında yük dağıtma |
| **Spring Cloud Config** | Merkezi yapılandırma (Git tabanlı) |
| **Spring Cloud Vault** | Gizli bilgileri HashiCorp Vault'tan okuma |
| **Spring Cloud OpenFeign** | Bildirimsel HTTP istemcileri (HTTP Interface'in öncülü) |
| **Spring Cloud CircuitBreaker** | Circuit breaker için soyutlama (arkada Resilience4j) |
| **Spring Cloud Stream** | Mesajlaşma için soyutlama (arkada Kafka veya RabbitMQ) |
| **Spring Cloud Contract** | Tüketici odaklı sözleşme testleri |
| **Spring Cloud Kubernetes** | Kubernetes'in ConfigMap ve servis keşfini Spring ile entegre etme |

> Spring Cloud CircuitBreaker ve Stream: Yine **"arayüz + değiştirilebilir implementasyon"** (SLF4J, Micrometer, Spring Cache gibi).

### Tarihçe: Netflix Bileşenleri

Spring Cloud'un ilk sürümleri büyük ölçüde **Netflix'in açık kaynak** araçlarına dayanıyordu. Çoğu zamanla bakım moduna girdi ve yerlerini başka araçlar aldı:

| Eski (Netflix) | Yerini Alan |
|---|---|
| **Hystrix** (circuit breaker) | **Resilience4j** |
| **Zuul** (gateway) | **Spring Cloud Gateway** |
| **Ribbon** (yük dağıtma) | **Spring Cloud LoadBalancer** |
| **Eureka** | Hâlâ kullanılıyor |
| Spring Cloud **Sleuth** (izleme) | **Micrometer Tracing** (Spring Boot 3 ile) |

> Eski eğitimlerde Hystrix, Zuul, Ribbon, Sleuth görürsen: Bunlar artık yeni projelerde kullanılmıyor.

### Sürüm Mantığı: Release Train

- Spring Cloud alt projeleri **ayrı ayrı** değil, uyumlu bir paket halinde yayınlanır: **Release train.**
- Eskiden Londra metro istasyonlarının adlarıyla (Hoxton, Greenwich...), şimdi **yıl tabanlı** sürümlerle (`2024.0`, `2025.0`...) adlandırılır.
- ⚠️ **Her release train belirli Spring Boot sürümleriyle uyumludur.** Uyumsuz sürümler başlangıçta hata verir.

| Spring Cloud | Spring Boot |
|---|---|
| 2024.0.x | 3.4.x |
| 2025.0.x | 3.5.x |
| 2025.1.x | 4.0.x |

> Spring Boot sürümü yükseltilince Spring Cloud sürümü de yükseltilmeli. **Resmi uyumluluk tablosuna** bakılmalı.

### Bağımlılıkları Eklemek: BOM

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2025.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Sürüm YAZILMAZ, BOM yönetir -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
</dependencies>
```

- **BOM (Bill of Materials):** Birbiriyle uyumlu tüm Spring Cloud kütüphanelerinin sürümlerini tek yerde tanımlar.
- Spring Boot'un kendi bağımlılıklarının sürümlerini yönetmesinin aynı mantığı.
- Tek tek starter'larda sürüm yazılmaz → Uyumsuz sürüm karışması önlenir.

### Spring Cloud'a Her Zaman İhtiyaç Var mı?

- Kubernetes; servis keşfi, yapılandırma, sağlık kontrolü, yeniden başlatma ve ölçeklemeyi **kendisi** sağlar.
- Kubernetes üzerindeki birçok sistemde **Eureka ve Config Server gereksiz** kalır.
- **Gateway, Resilience4j, Micrometer Tracing** ise Kubernetes'te de yaygın kullanılır.
- Bölüm 7'de karşılaştırma tablosu.

---

## 1. API Gateway

### Sorun

- Beş servis, beş ayrı adres. Ön yüz hepsini mi bilecek?
- Her servis JWT, CORS, istek sınırlamayı **ayrı ayrı** mı yapacak?
- Bir servisi ikiye bölünce ön yüz de mi güncellenecek?

### Çözüm: Tek Giriş Kapısı

```
                                      ┌──► /api/products/**  ──► Katalog servisi
Ön yüz ──► https://api.shop.com ──►  Gateway ──► /api/orders/**    ──► Sipariş servisi
                                      └──► /api/payments/**  ──► Ödeme servisi
```

> Spring Security **filter chain**'inin **sistem seviyesindeki** karşılığı. Strangler fig'deki "monolitin önüne konan yönlendirme katmanı" da bu.

### Spring Cloud Gateway

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: catalog
          uri: http://catalog-service:8080
          predicates:
            - Path=/api/products/**, /api/categories/**

        - id: orders
          uri: http://order-service:8080
          predicates:
            - Path=/api/orders/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
```

| Parça | Anlamı |
|---|---|
| `id` | Rotanın kimliği |
| `uri` | İsteğin gönderileceği adres |
| `predicates` | Rotaya uyma koşulu (`Path=` ≈ `requestMatchers`) |
| `filters` | Yönlendirme öncesi / cevap sonrası işler |

- `RequestRateLimiter` sayaçları **Redis**'te tutar → Gateway'in birden fazla kopyasında limit **toplamda** uygulanır.

> **Sürüm notu:** Geleneksel olarak reaktif (WebFlux); yeni sürümlerde Spring MVC versiyonu da var. Yeni sürümlerde bağımlılık adları ve yapılandırma anahtarları değişti; belgelere bakılmalı.

### Görevleri (Cross-Cutting Concerns)

| Görev | Açıklama |
|---|---|
| Yönlendirme | İsteği doğru servise iletmek |
| Kimlik doğrulama | JWT'yi kapıda doğrulamak (bölüm 4 uyarısıyla) |
| İstek sınırlama | Kötüye kullanım ve aşırı yüke karşı |
| CORS | Tarayıcı istekleri için izinler |
| İstek kimliği | `X-Request-Id`'yi en başta üretmek |
| HTTPS sonlandırma | Dış dünyayla şifreli iletişim |

### ⚠️ Tuzaklar

| Tuzak | Açıklama |
|---|---|
| **İş mantığı koymak** | Her servis değişikliği gateway'i de değiştirir → Dağıtık monolit. **İnce tut** (controller gibi). |
| **Tek hata noktası** | Gateway çökerse her şey çöker. **Birden fazla kopya** + **durumsuz** (sayaçlar Redis'te). |

## 2. Servis Keşfi

### Sorun

- Sipariş servisinin 3 kopyası, farklı IP'ler.
- Kampanyada 10'a çıkar, sonra 3'e iner. Çöken kopyanın yerine yenisi farklı adreste başlar.
- **Adresleri elle yazmak mümkün değil.**

### Çözüm 1: Eureka (Kayıt Defteri)

- Servisler başlarken kendini **kaydeder:** "Ben `order-service`, `10.0.0.15:8080`'deyim."
- Düzenli aralıklarla **"hâlâ buradayım"** sinyali. Gelmeyen kopya çıkarılır.
- Diğer servisler güncel adresleri Eureka'dan alır.

```yaml
uri: lb://order-service      # http:// yerine lb://
```

- `lb://`: "Servis adı; adresi kayıt defterinden bul, kopyalar arasında yükü dağıt."
- Servisler arası: `RestClient.Builder`'a `@LoadBalanced` → `http://order-service/...` otomatik çözülür.
- Health check'ler sayesinde trafik sadece sağlıklı kopyalara.

### Çözüm 2: Platformun Kendisi

- **Docker Compose:** Servise adıyla (`postgres`) ulaşmak = Basit servis keşfi.
- **Kubernetes Service:** Sabit DNS adı + sanal IP, sağlıklı kopyaları takip eder, yükü dağıtır. Uygulama sadece `http://order-service:8080` kullanır.

> Bugün yeni mikroservis sistemlerinin çoğu Kubernetes'te ve **Eureka genelde kullanılmıyor.** Eureka'yı bil (mevcut sistemlerde yaygın), ama yeni sistemde önce platformun sunduğuna bak.

## 3. Merkezi Yapılandırma

### Sorun

10 servis × 3 ortam = 30 yapılandırma. Bir ayarı değiştirmek = 10 servisi güncelleyip deploy etmek.

### Çözüm 1: Spring Cloud Config

Yapılandırmaları genelde bir **Git deposunda** tutan ve HTTP ile sunan ayrı servis.

```
config-repo/
├── application.yml               → tüm servislerde ortak
├── order-service.yml             → sadece sipariş servisi
├── order-service-prod.yml        → sipariş servisi, prod profili
└── catalog-service.yml
```

```yaml
spring:
  application:
    name: order-service
  config:
    import: configserver:http://config-server:8888
```

- Dosya adlandırması = `application-{profil}.yml` kuralı + başına servis adı.
- `spring.config.import`: `.env`'yi içe aktarmak için kullandığımız mekanizma.
- Her ayar değişikliği bir **commit** → Kim, neyi, ne zaman değiştirdi kayıt altında (Flyway'deki Git mantığı).

> ⚠️ Config deposuna **şifre yazılmaz.** Gizli bilgiler **Vault** (Spring Cloud Vault) veya ortam değişkenleri.

### Çözüm 2: Kubernetes

- **ConfigMap** (ayarlar) ve **Secret** (gizli bilgiler), genelde **ortam değişkeni** olarak verilir.
- Relaxed binding: `SPRING_DATASOURCE_URL` → `spring.datasource.url`
- Uygulama için ayarın ConfigMap'ten mi Compose'dan mı geldiği **fark etmez.**

> "Ayarlar dışarıdan gelir" yapısı uygulamayı bu platformlara hazırlamıştı. Kubernetes'te Config Server çoğu zaman gereksiz.

## 4. Servisler Arası Güvenlik

### Soru 1: JWT'yi Kim Doğrulayacak?

| Yaklaşım | Açıklama | Risk |
|---|---|---|
| **A: Sadece gateway** | Servisler gateway'den gelen her isteğe güvenir | İç ağa erişen saldırgan servislere **doğrudan** istek atar, kimlik sorulmaz |
| **B: Gateway + her servis** ✅ | Gateway erkenden reddeder, her servis de kendisi doğrular | Güvenli varsayılan |

> **Sıfır güven (zero trust):** İç ağ da güvenilir sayılmaz; her servis gelen isteği kendisi doğrular. (Bir parçanın ele geçirilmesi her şeyin ele geçirilmesi olmamalı.)

### Soru 2: ⚠️ Gizli Anahtar Sorunu

**HS256 (simetrik):** Aynı gizli anahtar hem imzalar hem doğrular.
- 10 servis doğrulama yapacak → 10 servisin hepsinde **gizli anahtar.**
- **Herhangi biri** ele geçirilirse saldırgan **admin dahil herkes** adına token üretir.
- Gizli bilginin kopyalandığı her yer yeni bir sızıntı noktası.

**RS256 (asimetrik):** İki anahtar.

| Anahtar | Nerede? | Ne Yapar? |
|---|---|---|
| **Özel anahtar** (private key) | Sadece kimlik doğrulama servisinde | Token **imzalar** |
| **Açık anahtar** (public key) | Herkeste olabilir | Sadece **doğrular**, token üretemez |

- Bir servis ele geçirilse bile **sahte token üretilemez.**
- Açık anahtar standart bir adresten yayınlanır: **JWKS** (JSON Web Key Set) endpoint'i.

### Kimlik Doğrulama Sunucusu ve Resource Server

| Rol | Görevi | Örnekler |
|---|---|---|
| **Authorization server** | Token üretir; giriş, şifre sıfırlama, 2FA tek yerde | Keycloak, Auth0, Okta, Spring Authorization Server |
| **Resource server** | Token **üretmez, sadece doğrular** | Diğer tüm servisler |

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.shop.com/realms/shop
```

```java
http.oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()));
```

- Elle yazılan `JwtAuthenticationFilter`, `JwtService` ve imza doğrulamanın **tamamının** yerini alır.
- `issuer-uri`'den açık anahtarlar otomatik bulunur; imza, süre ve üretici kontrol edilir.

> JWT konusunda filtreyi elle yazmamızın sebebi: Bu satırların arkasında ne olduğunu anlamak.

### ⚠️ SCOPE_ Öneki Tuzağı

- Spring varsayılan olarak yetkileri **`scope`** alanından okur ve **`SCOPE_`** öneki ekler.
- Roller `roles` gibi başka alanda geliyorsa `hasRole("ADMIN")` **eşleşmez** → 403.
- **Çözüm:** `JwtAuthenticationConverter` ile roller doğru alandan doğru önekle okunur.

> `ROLE_` öneki tartışmasının farklı kılıktaki versiyonu.

### Servis Servise Konuşurken

| Durum | Yöntem |
|---|---|
| **Kullanıcı adına** ("kullanıcının ödeme geçmişi") | Gelen **kullanıcı token'ı iletilir**; karşı servis kimin adına yapıldığını bilir |
| **Kendi adına** (zamanlanmış görev, kullanıcı yok) | **Client credentials** akışı: Servis kendi kimlik bilgileriyle kendisi için token alır |

## 5. Dağıtık İzleme

### Sorun

"Sipariş verirken 8 saniye bekledim." İstek gateway → sipariş → ödeme + envanter → Kafka → bildirim. **8 saniye nerede harcandı?**

### Kavramlar

| Kavram | Açıklama |
|---|---|
| **Trace** | Bir isteğin sistemdeki yolculuğunun **tamamı**. Tek `traceId`. |
| **Span** | Yolculuktaki **tek iş birimi** (HTTP çağrısı, DB sorgusu). Kendi `spanId`'si, başlangıcı, süresi, üst span'i. |

```
traceId: 4bf92f35...                                          toplam: 8.1 sn
├── gateway: POST /api/orders                     [████████████████████] 8.1 sn
│   └── order-service: placeOrder                 [███████████████████ ] 8.0 sn
│       ├── db: INSERT orders                     [▏                   ] 0.02 sn
│       ├── payment-service: POST /charges        [█████████████████   ] 7.4 sn  ← burada!
│       │   └── external: bank-api                [████████████████    ] 7.3 sn
│       └── inventory-service: POST /reserve      [▎                   ] 0.3 sn
```

> 7,3 sn banka API'sinde. Timeout ve circuit breaker eşiklerini ayarlamanın en iyi verisi.

### Kurulum

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0
  otlp:
    tracing:
      endpoint: http://tempo:4318/v1/traces
```

- **OpenTelemetry:** İzleme verisi için endüstri standardı. Veri Tempo, Jaeger, Zipkin gibi araçlardan herhangi birine gönderilebilir. ("Arayüz + değiştirilebilir arka uç.")
- **Kimlik taşınması otomatik:** `RestClient.Builder` `traceId`'yi istek başlıklarına ekler, karşı servis okuyup span'lerini aynı trace'e bağlar. Kafka / RabbitMQ'da mesaj başlıklarına eklenir (bazı sürümlerde ayrıca açılmalı).

### Örnekleme

| Ortam | `sampling.probability` |
|---|---|
| Spring Boot varsayılanı | **0.1** (%10) |
| Geliştirme | 1.0 (%100) |
| Production | Genelde düşük oran |

> Her span'i kaydetmek performans ve depolama maliyeti. (Metrik kardinalitesine benzer denge.)

### Üç Ayak Bir Arada

```
Metrikte ani yükseliş (cevap süreleri arttı)
   → O andaki yavaş trace'i aç (sorun ödeme servisinde)
   → traceId ile ödeme servisinin loglarına atla (bankadan gelen hata)
```

- `traceId` MDC'de → **Her log satırında** görünür.
- Loglar + metrikler + izler **birbirine bağlanınca**: Üç ayrı araç değil, **tek bir hikâye.**

## 6. Sözleşme Testleri

### Sorun

- Katalog ekibi `price` → `unitPrice` yaptı, kendi testleri geçti, deploy etti → **Sipariş servisi bozuldu.**
- Katalog ekibi sipariş servisinin bu alana ihtiyaç duyduğunu bilmiyordu.
- Tüm servisleri birlikte test etmek: Yavaş, kırılgan, pahalı.

### Tüketici Odaklı Sözleşme Testleri (Consumer-Driven Contracts)

```
1. Tüketici (sipariş) sağlayıcıdan NE BEKLEDİĞİNİ yazar:
   "GET /api/products/42 → price alanı olan bir cevap"
2. Sözleşme sağlayıcıya (katalog) iletilir
3. SAĞLAYICININ CI'ında otomatik test olarak çalışır
4. price değişirse katalog CI'ı KIRMIZI → deploy'dan önce yakalanır
```

- Araçlar: **Pact**, **Spring Cloud Contract**
- Branch protection ile birleşince sözleşmeyi bozan değişiklik `main`'e **giremez.**

## 7. Spring Cloud mu, Kubernetes mi?

| İhtiyaç | Spring Cloud | Kubernetes |
|---|---|---|
| **Servis keşfi** | Eureka + LoadBalancer | Service + yerleşik DNS |
| **Yapılandırma** | Config Server | ConfigMap + Secret |
| **Sağlık kontrolü** | Kayıt defteri heartbeat'leri | Liveness ve readiness probe'ları |
| **Yeniden başlatma, ölçekleme** | ❌ | ✅ |
| **API gateway** | Spring Cloud Gateway | Ingress / Gateway API |
| **Dayanıklılık** | Resilience4j | Kısmen (service mesh ile) |
| **Dağıtık izleme** | Micrometer Tracing | Kısmen (service mesh ile) |

- Actuator'daki **liveness / readiness** endpoint'leri tam Kubernetes'in kararları için var.
- Docker'daki **graceful shutdown:** Kubernetes de kapatırken SIGTERM gönderir.

> **Pratik bakış:**
> - **Altyapı sorunları** (keşif, yapılandırma, yeniden başlatma, ölçekleme) → Mümkünse **platforma** bırak.
> - **Uygulama davranışı** (iş akışı, retry stratejisi, fallback kararları) → **Uygulamada** çöz.
> - Gateway, Resilience4j, Micrometer Tracing Kubernetes'te de yaygın. Eureka ve Config Server genelde gereksiz.

> Kursta öğrenilenler (durumsuz uygulama, dışarıdan ayar, health endpoint'leri, graceful shutdown, Docker imajı, konteyner belleği) zaten **Kubernetes'e hazırlık.**

---

## Sorular

**1. Spring Cloud nedir? Spring Boot'tan farkı nedir?**
Dağıtık sistemlerin ortak sorunlarına çözüm sunan alt projelerin şemsiye adıdır. Spring Boot tek bir uygulamayı ayağa kaldırır; Spring Cloud birden fazla uygulamanın birlikte çalışmasını kolaylaştırır ve Spring Boot'un üzerine kurulur.

**2. Release train nedir? Neden önemlidir?**
Birbiriyle uyumlu Spring Cloud alt projelerinin birlikte yayınlanan paketidir. Her release train belirli Spring Boot sürümleriyle uyumludur; uyumsuz sürümler hata verir.

**3. Spring Cloud BOM ne işe yarar?**
Uyumlu tüm Spring Cloud kütüphanelerinin sürümlerini tek yerde tanımlar; starter'larda sürüm yazılmaz ve uyumsuz sürüm karışması önlenir.

**4. Hystrix, Zuul ve Ribbon'ın yerini ne aldı?**
Hystrix → Resilience4j, Zuul → Spring Cloud Gateway, Ribbon → Spring Cloud LoadBalancer. Sleuth'un yerini de Micrometer Tracing aldı.

**5. API gateway hangi sorunları çözer?**
Tek giriş noktası sağlar; yönlendirme, kimlik doğrulama, istek sınırlama, CORS ve istek kimliği gibi kesişen işleri tek yerde toplar.

**6. Gateway'e iş mantığı koymak neden yanlıştır?**
Her servis değişikliği gateway'i de değiştirmeyi gerektirir ve dağıtık monolit oluşur.

**7. Gateway'in istek sınırlama sayaçları neden Redis'te tutulur?**
Gateway'in birden fazla kopyası çalışır; ortak sayaç limitin toplamda uygulanmasını ve gateway'in durumsuz kalmasını sağlar.

**8. Servis keşfi neden gereklidir? Eureka nasıl çalışır?**
Kopya adresleri sürekli değişir. Servisler Eureka'ya kaydolup sinyal gönderir; diğerleri güncel adresleri oradan alır.

**9. Kubernetes'te Eureka neden genelde gereksizdir?**
Kubernetes Service nesneleri sabit DNS adı verir, sağlıklı kopyaları takip eder ve yükü dağıtır.

**10. Spring Cloud Config nedir?**
Servislerin yapılandırmalarını genelde Git'te tutup HTTP ile sunan merkezi servis. Gizli bilgiler buraya değil Vault'a konur.

**11. Sadece gateway'in JWT doğrulaması neden risklidir?**
İç ağa erişen saldırgan servislere doğrudan istek atabilir. Sıfır güven yaklaşımında her servis isteği kendisi doğrular.

**12. Mikroservislerde HS256 neden risklidir? RS256 nasıl çözer?**
HS256'da her servis gizli anahtara sahip olur ve biri ele geçirilirse herkes adına token üretilebilir. RS256'da servisler sadece doğrulamaya yarayan açık anahtara sahiptir.

**13. Resource server nedir?**
Token üretmeyen, sadece doğrulayan servistir. `issuer-uri` ile kimlik sunucusunun açık anahtarlarını otomatik bulur.

**14. Resource server'da `hasRole` neden eşleşmeyebilir?**
Yetkiler varsayılan olarak `scope` alanından `SCOPE_` önekiyle okunur; roller başka alandaysa `JwtAuthenticationConverter` gerekir.

**15. Client credentials akışı ne zaman kullanılır?**
Bir servis, ortada kullanıcı olmadan kendi adına başka servisi çağırırken kendisi için token almak için.

**16. Trace ve span farkı nedir?**
Trace bir isteğin tüm yolculuğudur; span bu yolculuktaki tek iş birimidir.

**17. Spring Boot neden varsayılan olarak isteklerin %10'unu izler?**
Tüm trace'leri kaydetmek performans ve depolama maliyeti yaratır; örnekleme bunu dengeler.

**18. Tüketici odaklı sözleşme testleri nedir?**
Tüketicinin sağlayıcıdan beklentilerini sözleşme olarak yazdığı ve bu sözleşmenin sağlayıcının CI'ında test olarak çalıştığı yaklaşım; kırıcı değişiklikleri deploy'dan önce yakalar.

**19. Hangi işler platforma, hangileri uygulamaya bırakılmalı?**
Keşif, yapılandırma, yeniden başlatma ve ölçekleme platforma; iş akışı, retry ve fallback kararları uygulamaya.
