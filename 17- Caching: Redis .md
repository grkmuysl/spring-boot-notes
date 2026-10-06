# Caching - Bölüm 2: Redis ile Dağıtık Önbellek

## 1. Redis Nedir?

Verileri **bellekte** tutan, **ayrı bir sunucu** olarak çalışan **anahtar-değer** veri deposu. Uygulama ona ağ üzerinden bağlanır.

```
           ┌── Sunucu A ──┐
Kullanıcı ─┼── Sunucu B ──┼──► Redis (tek, ortak önbellek)
           └── Sunucu C ──┘
```

- A'daki evict, herkesin baktığı **tek** önbelleği temizler → tutarsızlık sorunu çözülür.
- **Hız:** Genelde milisaniyenin altında. Veritabanından çok hızlı, ama **yerel önbellekten yüzlerce kat yavaş** (ağ isteği).

**Diğer kullanım alanları:** Oturum saklama, istek sınırlama (rate limiting), sıralama tabloları, **refresh token** saklama (süre verilebilir, süresi dolan kendiliğinden silinir).

> **Lisans notu:** Redis 2024'te lisans değiştirdi, Linux Foundation altında uyumlu açık kaynak fork **Valkey** doğdu. Uygulama tarafında aynı şekilde kullanılır.

> ⚠️ **Tasarım varsayımı:** Redis'teki veri **her an kaybolabilir.** Asıl kaynak veritabanıdır, önbellek sadece hızlandırıcıdır.

## 2. Docker ile Redis

```yaml
services:
  postgres:
    # ... önceki tanım

  redis:
    image: redis:7-alpine
    container_name: shop-redis
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

- `redis-cli ping` → `PONG` (PostgreSQL'deki `pg_isready` karşılığı)
- **Volume yok:** Önbellek verisinin kaybolması sorun değil.
- `spring-boot-docker-compose` kullanılıyorsa bağlantı otomatik yapılandırılır.

```bash
docker exec -it shop-redis redis-cli   # psql karşılığı
```

## 3. Spring Boot Bağlantısı

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 500ms
  cache:
    type: redis
```

- Eski eğitimlerdeki `spring.redis.*` → Spring Boot 3'te **`spring.data.redis.*`**
- `spring.cache.type: redis`: Classpath'te birden fazla sağlayıcı (Caffeine) varsa açıkça belirtilir.

> **`ProductService`'te tek satır değişmez.** Spring Cache soyutlamasının getirisi.

### redis-cli ile İnceleme

```
127.0.0.1:6379> KEYS *
1) "products::42"           ← önbellekAdı::anahtar

127.0.0.1:6379> TTL products::42
(integer) -1                ← süresi YOK (düzeltilecek)

127.0.0.1:6379> GET products::42
```

> ⚠️ **`KEYS *` production'da kullanılmaz.** Redis komutları sırayla işler; milyonlarca anahtarı tararken Redis'i **kilitler.** Production'da parça parça çalışan **`SCAN`** kullanılır.

## 4. Serileştirme

Caffeine nesneyi **uygulamanın belleğinde** doğrudan tutar. Redis ayrı bir programdır, sadece **bayt dizisi** saklar.

- **Serileştirme:** Nesne → bayt
- **Ters serileştirme:** Bayt → nesne

### Varsayılan: Java Serileştirmesi

| Sorun | Açıklama |
|---|---|
| `Serializable` zorunlu | Unutulursa çalışma anında hata |
| Okunamaz | `redis-cli` ile hata ayıklanamaz |
| Kırılgan | Sınıf yapısı değişince eski veriler okunamaz |

### JSON Serileştirme

```java
@Configuration
public class CacheConfig {

    @Bean
    public RedisCacheConfiguration redisCacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .disableCachingNullValues()
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }
}
```

| Ayar | Anlamı |
|---|---|
| `entryTtl(10 dk)` | **TTL.** Süre dolunca Redis kaydı kendiliğinden siler. |
| `disableCachingNullValues()` | `null` sonuçlar önbelleğe alınmaz |
| `GenericJackson2JsonRedisSerializer` | Nesneler JSON olarak saklanır |

> Spring Boot context'te `RedisCacheConfiguration` bean'i görürse varsayılanı onunla değiştirir: **"Senin bean'in varsa benimkini kullanmam."**

```json
{"@class":"com.example.shop.dto.ProductResponse","id":42,"name":"Laptop","price":25000,"inStock":true}
```

- **`@class`:** Okunurken hangi sınıfa dönüştürüleceğini bilmek için sınıfın tam adı saklanır.
- DTO'da `LocalDateTime` gibi tipler varsa `ObjectMapper`'a **`JavaTimeModule`** eklemek gerekebilir.

## 5. ⚠️ Yeni Deploy, Eski Önbellek

- Caffeine: Uygulama yeniden başlayınca önbellek **boşalır.**
- Redis: Uygulamadan **bağımsız yaşar.** Eski sürümün yazdığı kayıtlar kalır.

**Sorun:** Yeni sürümde DTO'ya alan eklendi veya paket değişti → Eski formattaki kayıtlar okunamaz → **Ters serileştirme hataları.**
Kademeli deploy'da eski ve yeni sürüm aynı önbelleğe farklı formatlarda yazar.

> Önbellekteki veri de kodla değişen bir "şemaya" sahip, ama Flyway gibi migration'ı yok.

| Çözüm | Nasıl? |
|---|---|
| **TTL** | Eski kayıtlar en fazla TTL kadar yaşar |
| **Sürüm öneki** | DTO değişince önek değişir, eski anahtarlara bakılmaz |
| **CacheErrorHandler** | Okuma hatası miss gibi değerlendirilir (bölüm 7) |

```java
RedisCacheConfiguration.defaultCacheConfig()
        .prefixCacheNameWith("v2::")   // anahtar: v2::products::42
```

## 6. Önbelleğe Özel TTL

```java
@Bean
public RedisCacheManagerBuilderCustomizer redisCacheCustomizer(RedisCacheConfiguration defaults) {
    return builder -> builder
            .withCacheConfiguration("categories", defaults.entryTtl(Duration.ofHours(6)))
            .withCacheConfiguration("productSearch", defaults.entryTtl(Duration.ofMinutes(2)));
}
```

- Varsayılan alınır, sadece TTL değiştirilir (serileştirme, önek korunur).
- `RedisCacheConfiguration` **değiştirilemez:** `entryTtl` yeni kopya döner (record mantığı).
- **`...Customizer` kalıbı:** Otomatik yapılandırmayı tamamen değiştirmeden bir kısmını özelleştirmek.

## 7. Redis Çökerse Ne Olur?

### Varsayılan Davranış

```
Redis çöktü → önbellek exception fırlatır → GlobalExceptionHandler → 500
```

> **Veritabanı sağlıklı, ürün orada, ama uygulama cevap veremiyor.** Bir hızlandırıcının bozulması sistemi durdurdu.

**Daha kötüsü: Redis yavaşladı.** Her istek cevap beklerken asılı kalır, thread'ler dolar, uygulama kilitlenir.

> ⚠️ **`spring.data.redis.timeout: 500ms` kritik.** Timeout olmadan yavaş bir bağımlılık tüm uygulamayı yavaşlatır.

### Graceful Degradation (Zarif Bozulma)

Bir bileşen bozulduğunda sistem **çökmek yerine daha düşük performansla çalışmaya devam eder.**

```java
@Configuration
@EnableCaching
public class CacheConfig implements CachingConfigurer {

    @Override
    public CacheErrorHandler errorHandler() {
        return new LoggingCacheErrorHandler();
    }
}
```

| Handler | Davranış |
|---|---|
| Varsayılan | Hatayı fırlatır → 500 |
| **`LoggingCacheErrorHandler`** | Hatayı **loglar**, okuma hatasını **miss** gibi değerlendirir → metot çalışır, veritabanından okunur |

- Hata yutulmaz, **loglanır**; sadece kullanıcıya yansıtılmaz.
- Deploy sonrası ters serileştirme hataları da kendiliğinden çözülür.

> ⚠️ `@CacheEvict` başarısız olursa da yutulur. Redis geri gelince eski kayıt orada olabilir. **TTL yine güvenlik ağı.**

### Actuator ve Readiness

- Redis bağımlılığıyla otomatik **`redis` health indicator** eklenir.
- Redis çökünce health **503** → yük dengeleyici trafiği keser.
- Tüm sunucular aynı Redis'e bağlı → **hepsi birden** trafikten çıkar.

```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          include: readinessState,db   # redis YOK
```

> **Kural:** Bir bağımlılığın sağlık kararındaki ağırlığı, uygulamanın ona ne kadar **muhtaç** olduğuna göre belirlenir.
> Veritabanı olmadan çalışılamaz → readiness'ta. Önbellek olmadan çalışılabilir → readiness'ta değil.

## 8. Cache Stampede (Önbellek İzdihamı)

```
Popüler kayıt TTL doldu → 300 istek aynı anda miss → 300 sorgu birden veritabanına
```

```java
@Cacheable(value = "products", sync = true)
public ProductResponse findById(Long id) { ... }
```

- `sync = true`: Bulunamayan bir anahtar için **sadece bir thread** metodu çalıştırır, diğerleri bekleyip sonucu önbellekten alır.
- ⚠️ **Sınırı:** Kilit her sunucunun **kendi içinde** geçerli. 3 sunucu = en fazla 3 sorgu (300'e göre büyük iyileşme).
- **Ek önlem:** TTL'lere küçük **rastgele** farklar eklemek (aynı anda giren kayıtlar aynı anda sona ermesin).

## 9. Yerel mi, Dağıtık mı?

| | Yerel (Caffeine) | Dağıtık (Redis) |
|---|---|---|
| **Hız** | Nanosaniyeler | Ağ gecikmesi (< 1 ms) |
| **Sunucular arası tutarlılık** | ❌ | ✅ |
| **Ek altyapı** | Yok | Redis sunucusu |
| **Serileştirme** | Gerekmez | Gerekir |
| **Yeniden başlatınca** | Boşalır | Korunur |
| **Bellek** | Her sunucuda ayrı kopya | Tek kopya |

- **Caffeine:** Tek sunucu veya kısa süreli tutarsızlık kabul edilebilir (30 sn TTL ile döviz kuru).
- **Redis:** Birden fazla sunucu ve tutarlılık önemli.
- **İki seviyeli önbellek:** Yerel → Redis → veritabanı. Büyük sistemlerde, ek karmaşıklık getirir.

## 10. Testcontainers ile Redis

```java
@SpringBootTest
@Testcontainers
class ProductCacheRedisTest {

    @Container
    @ServiceConnection(name = "redis")
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
}
```

- Redis için özel sınıf yerine **`GenericContainer`**, `name = "redis"` ile Spring Boot'a türü bildirilir.
- **Değeri:** DTO'nun gerçekten JSON'a çevrilip geri okunabildiğini doğrular. Serileştirme sorunları production'dan önce yakalanır.

---

## Sorular

**1. Redis yerel önbelleğin hangi sorununu çözer? Bedeli nedir?**
Tüm sunucular aynı önbelleği paylaşır, tutarsızlık ortadan kalkar. Bedeli ağ gecikmesi, ek altyapı ve serileştirme ihtiyacıdır.

**2. Redis önbellek olarak kullanılırken hangi varsayımla tasarım yapılır?**
Verinin her an kaybolabileceği varsayımıyla. Asıl kaynak veritabanıdır; önbellek olmadan sistem çalışabilmelidir.

**3. Production'da `KEYS *` neden kullanılmaz?**
Redis komutları sırayla işler; tüm anahtarları tararken Redis'i kilitler. `SCAN` kullanılır.

**4. Redis'te serileştirme neden gerekir? Neden JSON tercih edilir?**
Redis ayrı bir programdır ve sadece bayt saklar. Java serileştirmesi `Serializable` gerektirir, okunamaz ve kırılgandır; JSON okunabilir ve esnektir.

**5. Deploy sonrası önbellek okumalarında hata alınıyorsa sebep nedir?**
Redis uygulamadan bağımsız yaşar; eski sürümün eski formatta yazdığı kayıtlar DTO değiştiği için okunamaz. TTL, sürüm öneki ve `CacheErrorHandler` ile önlenir.

**6. Redis çökünce veritabanı sağlıklı olduğu halde neden 500 alınır?**
Önbellek hatası exception olarak metottan çıkar. `LoggingCacheErrorHandler` ile hatalar loglanıp miss gibi değerlendirilir.

**7. Graceful degradation nedir?**
Bir bileşen bozulduğunda sistemin çökmek yerine daha düşük performansla çalışmaya devam etmesidir.

**8. Redis bağlantısında timeout neden kritiktir?**
Yavaşlayan Redis, timeout olmadan istekleri asılı bırakır; thread'ler dolar ve uygulama kilitlenir.

**9. Redis health indicator'ı neden readiness grubundan çıkarılmalıdır?**
Önbellek isteğe bağlıdır; Redis çökünce tüm sunucuların trafikten çıkarılmasına yol açmamalıdır.

**10. Cache stampede nedir? `sync = true` nasıl yardımcı olur?**
Popüler kaydın süresi dolunca aynı anda gelen isteklerin hepsinin veritabanına gitmesidir. `sync = true` ile aynı anahtar için sadece bir thread metodu çalıştırır; kilit sunucu bazındadır.

**11. `RedisCacheManagerBuilderCustomizer` ne işe yarar?**
Spring Boot'un otomatik oluşturduğu cache manager'ı tamamen değiştirmeden, örneğin önbelleğe özel TTL'ler tanımlamak için özelleştirir.

**12. Caffeine ne zaman Redis yerine tercih edilir?**
Tek sunucuda veya kısa süreli tutarsızlığın kabul edilebildiği durumlarda; daha hızlıdır ve ek altyapı gerektirmez.
