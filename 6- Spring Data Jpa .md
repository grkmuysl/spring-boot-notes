# Spring Data JPA: Temeller, Sorgular ve Transaction

## 1. JPA, Hibernate ve Spring Data JPA

**ORM (Object-Relational Mapping):** Java nesnelerini tablo satırlarına, satırları tekrar nesnelere çevirme işi.

| İsim | Nedir? |
|---|---|
| **JPA** (Jakarta Persistence API) | ORM **standardı**. Arayüzler ve anotasyonlar (`@Entity`, `@Id`, `@Column`). Tek başına iş yapmaz. |
| **Hibernate** | JPA'nın en yaygın **implementasyonu**. SQL'leri üreten asıl iş gücü. Spring Boot'un varsayılanı. |
| **Spring Data JPA** | Üstteki **kolaylık katmanı**. Tekrarlı repository kodunu senin yerine üretir. |

**Akış:** Spring Data JPA → JPA → Hibernate → SQL → Veritabanı

> `jakarta.persistence` paketindeki anotasyonlar JPA'ya aittir, implementasyon değişse de kalır. `org.hibernate.annotations` paketindekiler (`@CreationTimestamp` gibi) Hibernate'e özeldir.

## 2. Kurulum

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

```properties
spring.datasource.url=jdbc:h2:mem:shopdb
spring.h2.console.enabled=true
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.hibernate.ddl-auto=create-drop
```

- `show-sql`: Üretilen SQL'leri konsola yazar. **Öğrenirken hep açık tut.**
- `h2-console`: `http://localhost:8080/h2-console` adresinden tabloları gösterir.

### ddl-auto

| Değer | Davranış |
|---|---|
| `create-drop` | Başlarken oluşturur, kapanırken siler |
| `create` | Başlarken silip yeniden oluşturur |
| `update` | Eksikleri ekler, hiçbir şey silmez |
| `validate` | Değiştirmez, uyuşmazlık varsa hata verir |
| `none` | Hiçbir şey yapmaz |

> **Production'da** `create`, `create-drop`, `update` kullanılmaz. Değişiklikler Flyway / Liquibase ile yönetilir, `ddl-auto` `validate` veya `none` olur.

## 3. Entity

```java
@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100, unique = true)
    private String name;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    private int stock;

    protected Product() {} // JPA için zorunlu

    public Product(String name, BigDecimal price, int stock) { ... }
}
```

| Anotasyon | Görevi |
|---|---|
| `@Entity` | Sınıf bir tabloya karşılık gelir |
| `@Table` | Tablo adı (yazılmazsa sınıf adı) |
| `@Id` | Birincil anahtar, **zorunlu** |
| `@GeneratedValue(IDENTITY)` | Id'yi veritabanının otomatik artan kolonu üretir |
| `@Column` | Kolon ayarları. Yazılmasa da her alan kolona dönüşür. |

- **Parametresiz constructor zorunludur.** Hibernate önce boş nesne oluşturur, sonra alanları doldurur. `protected` yapılır ki uygulama kodu eksik nesne oluşturamasın.
- **Entity'ler `record` olamaz:** Parametresiz constructor yok, alanlar `final`, sınıf `final` (Hibernate proxy üretemez).
- `@Column(nullable = false)` veritabanı kısıtıdır (son savunma hattı), DTO'daki `@NotNull` isteği kapıda reddeder. İkisi birlikte kullanılır.

## 4. Repository

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
}
```

- Arayüz yeterli, implementasyon yazılmaz. Spring çalışma zamanında implementasyonu (proxy) **kendisi üretip bean yapar.**
- `@Repository` gerekmez.
- Hazır metotlar: `save`, `saveAll`, `findById`, `findAll`, `existsById`, `count`, `deleteById`, `findAll(Pageable)`, `findAll(Sort)`
- Bellekteki repository'den JPA'ya geçince **service ve controller değişmedi.** Katmanlı mimarinin faydası.

## 5. Türetilmiş Sorgu Metotları

Metot adından sorgu üretilir:

```java
List<Product> findByCategory(String category);
Optional<Product> findByName(String name);
List<Product> findByCategoryAndPriceBetween(String category, BigDecimal min, BigDecimal max);
List<Product> findByNameContainingIgnoreCase(String keyword);
List<Product> findByNameStartingWithIgnoreCase(String prefix);
List<Product> findTop5ByOrderByPriceDesc();
boolean existsByName(String name);
long countByStockLessThan(int stock);
```

| Anahtar Kelime | Anlamı |
|---|---|
| `And`, `Or` | Bağlaçlar |
| `Not`, `IsNull` | Olumsuzluk, null kontrolü |
| `LessThan`, `GreaterThan`, `Between` | Karşılaştırma |
| `Containing`, `StartingWith`, `Like` | Metin arama |
| `IgnoreCase` | Büyük/küçük harf duyarsız |
| `In` | Liste içinde |
| `OrderBy...Asc/Desc` | Sıralama |
| `Top`, `First` | İlk N kayıt |

**Dönüş tipleri:** Tek sonuç `Optional<T>`, çok sonuç `List<T>`, varlık kontrolü `boolean`, sayım `long`.

- Alan adı yanlış yazılırsa uygulama **ayağa kalkmaz** (fail-fast).
- Metot adı çok uzarsa `@Query` kullanılır.

### Benzersizlik Kontrolü

```java
if (repository.existsByName(request.name())) {
    throw new BusinessException("Bu isimde bir ürün zaten var");
}
```

İki istek aynı anda gelirse ikisi de kontrolden geçebilir. Bu yüzden veritabanında da `@Column(unique = true)` olmalı. Service kontrolü düzgün mesaj için, veritabanı kısıtı son savunma hattı.

## 6. @Query

### JPQL

```java
@Query("SELECT p FROM Product p WHERE p.category = :category AND p.price < :maxPrice")
List<Product> findAvailable(@Param("category") String category,
                            @Param("maxPrice") BigDecimal maxPrice);
```

- **Entity ve alan adlarıyla** çalışır: `FROM Product` (tablo adı değil), `p.costPrice` (kolon adı değil).
- Veritabanından bağımsızdır, Hibernate uygun SQL'e çevirir.
- Başlangıçta kontrol edilir.
- Değerler **asla string birleştirerek** eklenmez (SQL injection). Parametre kullanılır.

### Native Query

```java
@Query(value = "SELECT * FROM products WHERE price > :minPrice", nativeQuery = true)
List<Product> findExpensive(@Param("minPrice") BigDecimal minPrice);
```

Tablo ve kolon adları kullanılır. Veritabanına bağımlıdır, başlangıçta kontrol edilmez. Gerekmedikçe JPQL tercih edilir.

### Güncelleme / Silme

```java
@Modifying
@Transactional
@Query("UPDATE Product p SET p.price = p.price * :ratio WHERE p.category = :category")
int increasePrices(@Param("category") String category, @Param("ratio") BigDecimal ratio);
```

`@Modifying` zorunludur. Dönüş değeri etkilenen satır sayısıdır.

## 7. Sayfalama ve Sıralama

Liste döndüren endpoint'ler **her zaman sayfalanır.** `findAll()` milyonlarca kaydı belleğe çeker.

```java
// Repository
Page<Product> findByCategory(String category, Pageable pageable);

// Service
public Page<ProductResponse> findAll(Pageable pageable) {
    return repository.findAll(pageable).map(this::toResponse);
}

// Controller
@GetMapping
public Page<ProductResponse> getAll(@PageableDefault(size = 20, sort = "name") Pageable pageable) {
    return service.findAll(pageable);
}
```

```
GET /api/products?page=0&size=10
GET /api/products?page=2&size=10&sort=price,desc
```

- Sayfa numaraları **0'dan başlar.**
- `Page` iki sorgu çalıştırır: veri (`LIMIT/OFFSET`) + toplam sayı (`COUNT`).
- Toplam sayı gerekmiyorsa `Slice` kullanılır (`COUNT` çalışmaz, sadece "sonraki sayfa var mı" bilgisi).
- Üst sınır: `spring.data.web.pageable.max-page-size` (varsayılan 2000).
- `Page`'i doğrudan JSON yapmak Spring Data 3.3+'ta uyarı verir. Kendi `PageResponse` record'unu yaz veya `@EnableSpringDataWebSupport(pageSerializationMode = VIA_DTO)` kullan.

## 8. Transaction

Birden fazla veritabanı işlemini **bölünemez tek birim** olarak çalıştırır: ya hepsi başarılı olur (**commit**), ya da hepsi geri alınır (**rollback**).

```java
@Transactional
public void placeOrder(Long productId, int quantity) {
    Product product = productRepository.findById(productId).orElseThrow(...);
    product.setStock(product.getStock() - quantity);
    orderRepository.save(new Order(product, quantity)); // hata olursa stok da geri alınır
}
```

- **Service katmanına** yazılır. "Bu işlemler birlikte yapılmalı" bir iş kuralıdır.
- Repository metotları zaten kendi transaction'ında çalışır. Sorun, birden fazlasını birleştirmektir.

### readOnly

```java
@Service
@Transactional(readOnly = true)
public class ProductService {

    public ProductResponse findById(Long id) { ... }   // readOnly

    @Transactional                                      // yazma
    public ProductResponse create(ProductRequest request) { ... }
}
```

- Dirty checking atlanır, performans kazanılır.
- Yanlışlıkla yapılan değişikliklerin kaydedilmesini önler.
- Metot seviyesindeki anotasyon sınıf seviyesindekini geçersiz kılar.

### Tuzak 1: Checked Exception Rollback Yapmaz

Varsayılan olarak sadece **unchecked** exception'larda (`RuntimeException`, `Error`) rollback yapılır. Checked exception'da (`IOException`) **commit edilir.**

```java
@Transactional(rollbackFor = Exception.class)
```

> Kendi exception'larını `RuntimeException`'dan türetmenin bir sebebi de bu.

### Tuzak 2: Self-Invocation

`@Transactional` **proxy** üzerinden çalışır. Dışarıdan gelen çağrı proxy'den geçer, transaction açılır. Ama sınıf içinden `this.method()` ile yapılan çağrı proxy'yi atlar ve **transaction açılmaz.** Hata da alınmaz.

```java
public void processCart(List<Long> ids) {
    ids.forEach(id -> placeOrder(id, 1)); // proxy atlanır!
}

@Transactional
public void placeOrder(Long productId, int quantity) { ... }
```

**Çözüm:** Transaction'ı dıştaki metoda koymak veya metodu ayrı bir bean'e taşımak.

- `private` metotlarda `@Transactional` çalışmaz.
- `@Cacheable`, `@Async` gibi anotasyonlar da aynı proxy mekanizmasıyla çalışır ve aynı tuzağa sahiptir.

## 9. Persistence Context

Hibernate'in transaction boyunca yüklediği entity'leri tuttuğu alan (birinci seviye önbellek).

```java
@Transactional
public void example(Long id) {
    Product p1 = repository.findById(id).get(); // SQL çalışır
    Product p2 = repository.findById(id).get(); // SQL çalışmaz
    System.out.println(p1 == p2);                // true
}
```

### Entity Durumları

| Durum | Açıklama |
|---|---|
| **Transient** | `new` ile oluşturuldu, Hibernate'in haberi yok |
| **Managed** | Persistence context'te, Hibernate takip ediyor |
| **Detached** | Transaction bitti, artık takip edilmiyor |
| **Removed** | Silinmek üzere işaretlendi |

## 10. Dirty Checking

Managed bir entity'de yapılan değişiklik, `save()` çağrılmadan **commit sırasında otomatik kaydedilir.**

```java
@Transactional
public void updateStock(Long id, int newStock) {
    Product product = repository.findById(id).orElseThrow(...);
    product.setStock(newStock); // save() yok ama UPDATE çalışır
}
```

Hibernate yüklediği entity'nin kopyasını saklar, commit'te karşılaştırır, değişen ("kirli") alanlar için `UPDATE` üretir.

### Tehlikesi

```java
@Transactional
public ProductResponse findWithDiscount(Long id) {
    Product product = repository.findById(id).orElseThrow(...);
    product.setPrice(product.getPrice().multiply(new BigDecimal("0.9"))); // sadece göstermek için
    return toResponse(product); // ama fiyat veritabanına kaydedilir!
}
```

- Sadece göstermek için hesaplama yapılacaksa **entity değil DTO değiştirilir.**
- `readOnly = true` olsaydı değişiklik kaydedilmezdi.

---
