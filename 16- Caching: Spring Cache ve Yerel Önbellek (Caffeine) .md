# Caching - Bölüm 1: Spring Cache ve Yerel Önbellek

## 1. Sorun ve Takas

Aynı pahalı işin tekrar tekrar yapılması:
- Ürün detayı günde yüz binlerce kez okunuyor, haftada bir değişiyor.
- Dış servisten döviz kuru her çağrıda 300 ms sürüyor, dakikalarca değişmiyor.

Sonuç: Bağlantı havuzu dolar (`hikaricp.connections.pending`), cevap süreleri uzar (`http.server.requests`).

**Cache:** Pahalı işlemin sonucunu hızlı erişilen yerde saklayıp tekrar istendiğinde işlemi yapmadan döndürmek.

> Persistence context de bir önbellekti, ama transaction içinde. Burada istekler **arası** önbellek kuruluyor.

### Bedeli: Cache Invalidation

Veri önbelleğe alınınca **iki kopyası** olur. Veritabanındaki değişince önbellektekinin de güncellenmesi gerekir; unutulursa kullanıcı **eski (stale) veri** görür.

> **Takas:** Hız karşılığında tazelik riske atılır.

| ✅ Önbelleğe Uygun | ❌ Uygun Değil |
|---|---|
| Çok okunan, az değişen (ürün detayı, kategoriler) | Sık değişen (anlık stok) |
| Üretmesi pahalı (yavaş sorgu, dış servis) | Kesin doğru olmalı (bakiye, ödeme durumu) |
| Kısa süre eski kalabilir (döviz kuru) | Kullanıcıya özel ve hassas |

> ⚠️ **Stok örneği:** Önbellekten "stokta var" okunur ama son ürün az önce satıldıysa, olmayan ürün satılır.

> **Önce ölç, sonra önbelleğe al.** Actuator metrikleri ve Hibernate SQL logları nereye bakılacağını söyler.

## 2. Spring Cache Soyutlaması

> SLF4J loglar için, Micrometer metrikler için, **Spring Cache önbellek için.** Kod anotasyonlara bağımlı, sağlayıcı (bellek, Redis) kod değişmeden değiştirilir.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableCaching
public class ShopApplication { ... }
```

> ⚠️ **`@EnableCaching` unutulursa** anotasyonlar derlenir, uygulama çalışır ama **hiçbir şey önbelleğe alınmaz.** Hata alınmaz.

- Ek yapılandırma yoksa varsayılan: Basit `ConcurrentHashMap` (sadece öğrenmek için).

## 3. @Cacheable

```java
@Service
@Transactional(readOnly = true)
public class ProductService {

    @Cacheable("products")
    public ProductResponse findById(Long id) {
        log.debug("Ürün veritabanından yükleniyor: id={}", id);
        Product product = repository.findById(id).orElseThrow(...);
        return toResponse(product);
    }
}
```

1. Önce `products` önbelleğine bak.
2. Bu parametreyle varsa → metodu **hiç çalıştırmadan** saklanan sonucu dön.
3. Yoksa → metodu çalıştır, sonucu önbelleğe koy, dön.

> İkinci çağrıda ne SQL ne de metot içindeki log satırı görünür.

### Önbellek Adı ve Anahtar

```
Map<önbellek adı, Map<anahtar, değer>>
```

- **Önbellek adı** (`"products"`): İsim alanı, farklı veri türlerini ayırır.
- **Anahtar:** Varsayılan olarak **metot parametrelerinden** üretilir. `findById(42L)` → `42`

```java
@Cacheable(value = "products", key = "#id")
public ProductResponse findById(Long id) { ... }

@Cacheable(value = "productSearch", key = "#category + ':' + #page")
public List<ProductResponse> search(String category, int page, String requestedBy) { ... }
```

- SpEL ile `#parametreAdı` (`@PreAuthorize`'daki gibi).
- Sonucu **etkilemeyen** parametre (`requestedBy`) anahtara katılmaz.

> ⚠️ Sonucu **etkileyen** parametre anahtardan çıkarılırsa farklı girdiler aynı anahtarı paylaşır, **bir kullanıcı başkasının sonucunu görür.** Kullanıcıya özel sonuçta kullanıcı kimliği anahtarda olmalı.

### Koşullar

```java
@Cacheable(value = "products", unless = "#result == null")
```

| Özellik | Ne Zaman Değerlendirilir? | Anlamı |
|---|---|---|
| `condition` | Metot çağrılmadan **önce** | Sağlanmazsa önbellek hiç devreye girmez |
| `unless` | Metot çalıştıktan **sonra** | Sağlanırsa sonuç önbelleğe **alınmaz** |

- **Exception'lar önbelleğe alınmaz.**

## 4. Proxy ile Çalışır

```java
public List<ProductResponse> findByIds(List<Long> ids) {
    return ids.stream()
            .map(this::findById)   // PROXY ATLANIR → önbellek çalışmaz
            .toList();
}

@Cacheable("products")
public ProductResponse findById(Long id) { ... }
```

> ⚠️ **Self-invocation:** Sınıf içi çağrıda önbellek devreye girmez, hata da alınmaz. `private` metotlarda da çalışmaz.
> **Çözüm:** Önbelleklenen metodu ayrı bean'e taşı veya önbelleği dışarıdan çağrılan metoda koy.

### @Transactional ile Sıralama

- Cache proxy'si, transaction proxy'sinin **dışında** çalışır.
- Önbellekte bulunursa metoda ve **transaction'a hiç girilmez**, veritabanı bağlantısı tüketilmez.
- Caching sonrası `hikaricp` metriklerinin düşmesinin sebebi bu.

## 5. Önbelleği Geçersiz Kılmak

### @CacheEvict: Sil

```java
@Transactional
@CacheEvict(value = "products", key = "#id")
public void delete(Long id) {
    repository.deleteById(id);
}
```

```java
@CacheEvict(value = "productSearch", allEntries = true)   // tüm önbelleği temizle
```

> Arama / liste sonuçlarında hangi sonuçların etkilendiği bilinemez; önbelleğin **tamamı** temizlenir.

### @CachePut: Güncelle

```java
@Transactional
@CachePut(value = "products", key = "#id")
public ProductResponse update(Long id, ProductRequest request) {
    Product product = repository.findById(id).orElseThrow(...);
    product.update(request);
    return toResponse(product);
}
```

| | `@Cacheable` | `@CachePut` | `@CacheEvict` |
|---|---|---|---|
| **Metot çalışır mı?** | Önbellekte varsa **hayır** | **Her zaman** | Her zaman |
| **Önbelleğe etkisi** | Yoksa sonucu yazar | Sonucu yazar | Siler |
| **Kullanım** | Okuma | Güncelleme | Güncelleme / silme |

> ⚠️ Güncelleme metoduna yanlışlıkla **`@Cacheable`** yazılırsa, önbellekte değer olduğu sürece güncelleme **hiç çalışmaz.**

> ⚠️ `@CachePut` metodu önbellekteki değerle **aynı tipi** dönmeli. `void` dönerse önbelleğe `null` yazılır.
> Çoğu zaman **`@CacheEvict` daha basit ve güvenli:** Sil, sonraki okuma taze veriyi yüklesin.

### @Caching: Birden Fazla İşlem

```java
@Caching(evict = {
        @CacheEvict(value = "products", key = "#id"),
        @CacheEvict(value = "productSearch", allEntries = true)
})
public ProductResponse update(Long id, ProductRequest request) { ... }
```

### ⚠️ Unutulan Yazma Yolu

```java
@Modifying
@Query("UPDATE Product p SET p.price = p.price * :ratio WHERE p.category = :category")
int increasePrices(String category, BigDecimal ratio);   // hiçbir @CacheEvict'ten geçmez!
```

Önbelleği atlayan yollar:
- `@Modifying` toplu güncelleme sorguları
- Doğrudan veritabanında çalıştırılan SQL
- Flyway migration'ları
- Aynı veritabanını kullanan başka uygulamalar

> **Sorulacak soru:** "Bu veri değişebilecek **tüm yollar** hangileri ve hepsi önbelleği temizliyor mu?"
> Cevap her zaman tam olmayacağı için **TTL** güvenlik ağıdır.

## 6. Entity Değil, DTO Önbelleğe Alınır

```java
@Cacheable("products")
public Product findEntity(Long id) { ... }   // YANLIŞ
```

| Sorun | Açıklama |
|---|---|
| **Detached + lazy** | Transaction bitmiş, entity detached. Lazy alana erişim → `LazyInitializationException` |
| **Paylaşılan, değiştirilebilir nesne** | Önbellekteki **tek nesne** herkese verilir. Bir istek `setPrice(...)` yaparsa **sonraki tüm istekler** değişmiş fiyatı görür. (Singleton'da state tutma tehlikesinin aynısı) |
| **İç modelin sızması** | Önbellek uygulamanın dışına (Redis) taşınabilir |

> **DTO önbelleğe al.** `record` DTO'lar **değiştirilemez**, lazy alanı yoktur, transaction'a bağlı değildir.

## 7. Caffeine: Gerçek Yerel Önbellek

**`ConcurrentHashMap` önbelleğinin eksikleri:**
- **Boyut sınırı yok** → Her şey bellekte birikir, uygulama çöker (`jvm.memory.used` sürekli artar).
- **Süre sınırı yok** → Unutulan geçersiz kılmada eski veri **sonsuza kadar** kalır.

```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

```yaml
spring:
  cache:
    cache-names: products,categories,productSearch
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=10m,recordStats
```

- Classpath'te Caffeine varsa Spring Boot onu kullanır. **`@Cacheable` kodu değişmez.**

| Ayar | Anlamı |
|---|---|
| `maximumSize=1000` | En fazla 1000 kayıt. Dolunca en az kullanılması muhtemel olanlar çıkarılır. |
| `expireAfterWrite=10m` | **TTL:** Kayıt yazıldıktan 10 dk sonra geçersiz. Unutulan evict'e karşı güvenlik ağı. |
| `recordStats` | İstatistik toplar (Actuator metrikleri için) |

> **TTL belirlerken sor:** "Bu veri en fazla ne kadar eski olabilir?" Ürün açıklaması: saatler. Fiyat: dakikalar.

### İsabet Oranı

```
GET /actuator/metrics/cache.gets?tag=cache:products&tag=result:hit
GET /actuator/metrics/cache.gets?tag=cache:products&tag=result:miss
```

```
İsabet oranı = hit / (hit + miss)
```

Düşükse (örn. %10) önbellek işe yaramıyor:
- Anahtar tasarımı yanlış (her çağrı farklı anahtar üretiyor)
- Veri zaten tekrar istenmiyor

> Önbelleği ekledikten sonra da **işe yarayıp yaramadığını ölç.**

## 8. Testler ve Önbellek

### İstenmeyen Etki

> ⚠️ Test sınıfındaki `@Transactional` veritabanını geri alır, **önbelleği geri almaz.** Bir testin önbelleğe koyduğu değer sonraki teste sızar.

```yaml
# application-test.yml
spring:
  cache:
    type: none
```

### Önbelleğin Çalıştığını Test Etmek

```java
@SpringBootTest
class ProductServiceCacheTest {

    @Autowired private ProductService productService;    // Spring'den: proxy devrede
    @MockitoBean private ProductRepository productRepository;

    @Test
    void findById_secondCallIsServedFromCache() {
        when(productRepository.findById(1L)).thenReturn(Optional.of(laptop));

        productService.findById(1L);
        productService.findById(1L);

        verify(productRepository, times(1)).findById(1L);
    }
}
```

- İki çağrı, repository'ye **bir kez** gidildi.
- Proxy devrede olmasaydı (self-invocation) test kırılırdı.

## 9. Yerel Önbelleğin Sınırı

```
           ┌── Sunucu A (önbellek: yeni fiyat) ← güncelleme buraya düştü, evict çalıştı
Kullanıcı ─┼── Sunucu B (önbellek: ESKİ fiyat)
           └── Sunucu C (önbellek: ESKİ fiyat)
```

- Her sunucunun **kendi** önbelleği var.
- Güncelleme sadece düştüğü sunucunun önbelleğini temizler.
- Kullanıcı, isteğinin düştüğü sunucuya göre bir yeni bir eski fiyat görür.

> Session tabanlı kimlik doğrulamadaki "oturum başka sunucuda" sorununun önbellek versiyonu. **Çözüm: Ortak önbellek (Redis).**

---

## Örnek Sorular

**1. Cache invalidation neden zordur?**
Veri önbelleğe alınınca iki kopyası olur. Veritabanındaki değişince önbellektekinin de güncellenmesi gerekir; tüm değişiklik yollarını bilmek zordur ve unutulursa eski veri gösterilir.

**2. Hangi veriler önbelleğe alınmamalıdır?**
Sık değişen ve kesin doğru olması gereken veriler (anlık stok, bakiye, ödeme durumu).

**3. `@EnableCaching` unutulursa ne olur?**
Uygulama hatasız çalışır ama cache anotasyonları işlenmez, hiçbir şey önbelleğe alınmaz.

**4. `@Cacheable` önbellek anahtarını nasıl üretir?**
Varsayılan olarak metot parametrelerinden. `key` özelliğiyle SpEL ifadesi verilebilir.

**5. Sonucu etkileyen bir parametre anahtardan çıkarılırsa ne olur?**
Farklı girdiler aynı anahtarı paylaşır; bir kullanıcı başka bir kullanıcının sonucunu görebilir.

**6. `@Cacheable`, `@CachePut` ve `@CacheEvict` farkı nedir?**
`@Cacheable` önbellekte varsa metodu atlar. `@CachePut` metodu her zaman çalıştırıp sonucu yazar. `@CacheEvict` önbellekten siler.

**7. Güncelleme metoduna `@Cacheable` yazılırsa ne olur?**
Önbellekte değer olduğu sürece metot çalışmaz, güncelleme veritabanına yazılmaz.

**8. Sınıf içinden çağrılan `@Cacheable` metot neden önbelleğe almaz?**
Proxy ile çalışır; `this` üzerinden yapılan çağrı proxy'yi atlar.

**9. Önbellekte bulunan bir değer için `@Transactional` metodun transaction'ı açılır mı?**
Hayır. Cache proxy'si transaction proxy'sinin dışında çalışır; önbellekte bulunursa metoda hiç girilmez.

**10. `@CacheEvict` olduğu halde eski veri gösteriliyorsa sebep ne olabilir?**
Veri önbelleği temizlemeyen başka bir yoldan değişmiştir: `@Modifying` sorgusu, doğrudan SQL, migration veya başka uygulama.

**11. Neden entity değil DTO önbelleğe alınır?**
Önbellekteki entity detached'dır (lazy alanlar patlar) ve değiştirilebilirdir (bir istekteki değişiklik herkesi etkiler). Record DTO'lar değiştirilemez ve transaction'a bağlı değildir.

**12. Varsayılan `ConcurrentHashMap` önbelleği neden production için uygun değildir?**
Boyut ve süre sınırı yoktur; bellek sürekli artar ve eski veri kalıcı olur.

**13. TTL nedir, neden önemlidir?**
Kaydın önbellekte kalacağı süredir. Unutulan geçersiz kılmalara karşı güvenlik ağıdır; eski veri en fazla TTL kadar yaşar.

**14. İsabet oranı düşükse ne anlama gelir?**
Önbellek işe yaramıyordur; anahtar tasarımı yanlış olabilir veya veri tekrar istenmiyordur.

**15. Test sınıfındaki `@Transactional` önbelleği geri alır mı?**
Hayır, sadece veritabanını. Testlerde `spring.cache.type: none` ile önbellek kapatılabilir.

**16. Birden fazla sunucuda yerel önbellek hangi sorunu yaratır?**
Her sunucunun kendi önbelleği vardır; bir sunucudaki evict diğerlerini etkilemez ve kullanıcılar tutarsız veri görür.
