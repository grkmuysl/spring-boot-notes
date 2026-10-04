# Spring Data JPA: Entity İlişkileri ve Proxy

## 0. Proxy Nedir?

Bu notta (ve `@Transactional` konusunda) sürekli "proxy" kelimesi geçiyor. Önce bu kavramı netleştirelim.

### Fikir

**Proxy, gerçek nesnenin önünde duran ve onun yerine geçen bir aracı nesnedir.** Gerçek nesneyle aynı tipe sahiptir, bu yüzden çağıran taraf farkı anlamaz. Gelen çağrıyı önce proxy karşılar, araya kendi işini katar, sonra gerçek nesneye iletir (veya iletmez).

> **Benzetme:** Bir yöneticinin asistanı. Yöneticiyle konuşmak istediğinde önce asistana gidersin. Asistan randevuyu kontrol eder, notu alır, sonra seni yöneticiye yönlendirir. Sen yöneticiyle konuştuğunu düşünürsün ama araya biri girmiştir.

### Elle Yazılmış Bir Proxy

```java
public interface ProductService {
    void placeOrder(Long id);
}

// Gerçek nesne: sadece iş mantığı
public class ProductServiceImpl implements ProductService {
    public void placeOrder(Long id) {
        System.out.println("Sipariş oluşturuldu: " + id);
    }
}

// Proxy: aynı arayüz, araya transaction mantığı katıyor
public class TransactionalProxy implements ProductService {
    private final ProductService target;

    public TransactionalProxy(ProductService target) {
        this.target = target;
    }

    public void placeOrder(Long id) {
        System.out.println("Transaction açıldı");
        try {
            target.placeOrder(id);               // gerçek nesneye ilet
            System.out.println("Commit");
        } catch (RuntimeException e) {
            System.out.println("Rollback");
            throw e;
        }
    }
}
```

```java
ProductService service = new TransactionalProxy(new ProductServiceImpl());
service.placeOrder(5);
// Transaction açıldı
// Sipariş oluşturuldu: 5
// Commit
```

Çağıran taraf `ProductService` tipinde bir nesne kullanıyor ve araya bir proxy girdiğinden habersiz. **`@Transactional` tam olarak bunu yapar**, sadece Spring bu sınıfı elle yazmamızı beklemeden çalışma zamanında kendisi üretir.

### Spring Proxy'yi Nasıl Üretir?

| Yöntem | Nasıl Çalışır | Ne Zaman |
|---|---|---|
| **JDK Dynamic Proxy** | Aynı **arayüzü** uygulayan bir sınıf üretir | Hedef bir arayüz olduğunda (örn. Spring Data repository'leri) |
| **CGLIB / ByteBuddy** | Hedef sınıftan **türeyen** (subclass) bir sınıf üretir | Arayüz yokken (Spring Boot'ta varsayılan) |

Türeyerek çalışmanın sonuçları:
- `final` sınıflar proxy'lenemez (türetilemez).
- `final` ve `private` metotlar ezilemez, proxy araya giremez. Bu yüzden `private` metotta `@Transactional` çalışmaz.

### Spring'de Proxy Nerelerde Karşımıza Çıkıyor?

| Yer | Proxy Ne Yapıyor? |
|---|---|
| `@Transactional` | Metottan önce transaction açar, sonra commit / rollback yapar |
| Spring Data repository | Arayüzün implementasyonunu çalışma zamanında üretir |
| Hibernate lazy loading | Gerçek entity yerine durur, ilk erişimde veritabanından yükler |
| `@Cacheable`, `@Async` | Önbelleğe bakar / metodu ayrı thread'de çalıştırır |

### Self-Invocation Tuzağı (Proxy Mantığıyla)

```
Controller ──► [Proxy] ──► ProductService.processCart()
                                   │
                                   └──► this.placeOrder()   ← proxy atlandı!
```

Dışarıdan gelen çağrı proxy'den geçer. Ama sınıf içinden `this.placeOrder()` çağrısı doğrudan gerçek nesneye gider; proxy araya giremez ve `@Transactional`, `@Cacheable` gibi anotasyonlar çalışmaz.

---

## 1. Veritabanında İlişki

İlişkisel veritabanında ilişki **foreign key** ile kurulur:

```
categories                    products
+----+-------------+          +----+----------+-------------+
| id | name        |          | id | name     | category_id |
+----+-------------+          +----+----------+-------------+
| 1  | Elektronik  |  <-----  | 10 | Laptop   | 1           |
| 2  | Kitap       |  <-----  | 11 | Telefon  | 1           |
+----+-------------+          +----+----------+-------------+
```

`categories` tablosunda ürünlere dair kolon yok. **İlişki bilgisi sadece foreign key'in olduğu tabloda durur.** "İlişkinin sahibi" kavramının temeli bu.

## 2. @ManyToOne

```java
@Entity
public class Product {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private Category category;
}
```

- Java'da sayı (`categoryId`) değil nesne (`Category`) tutulur, JPA kolona çevirir.
- `@JoinColumn`: Foreign key kolonunun adı.
- **`@ManyToOne`'a her zaman `LAZY` yazılır** (sebebi aşağıda).
- DTO'da nesne değil `categoryId` alınır, service kategoriyi bulup bağlar.

Tek yönlü ilişki çoğu zaman yeterlidir. Kategorinin ürünleri için:

```java
List<Product> findByCategoryId(Long categoryId);
```

## 3. @OneToMany ve İki Yönlü İlişki

```java
@Entity
public class Category {

    @OneToMany(mappedBy = "category")
    private List<Product> products = new ArrayList<>();

    public void addProduct(Product product) {
        products.add(product);
        product.setCategory(this);
    }

    public void removeProduct(Product product) {
        products.remove(product);
        product.setCategory(null);
    }
}
```

### İlişkinin Sahibi (Owning Side)

- **Sahip:** Foreign key'in olduğu taraf. Kodda `@JoinColumn` olan taraf (`Product.category`).
- **Sahip olmayan:** `mappedBy` olan taraf. Sadece bir yansımadır.
- `@OneToMany`–`@ManyToOne` ilişkisinde sahip her zaman `@ManyToOne` tarafıdır.

```java
category.getProducts().add(product); // YANLIŞ: category_id kaydedilmez
category.addProduct(product);        // DOĞRU: iki taraf birlikte güncellenir
```

> **Kural:** Veritabanı sahip tarafa bakar, ama Java nesnelerinin tutarlı kalması için iki tarafı da güncelle.

### Tek Yönlü mü, İki Yönlü mü?

- **Önce sadece `@ManyToOne` ile başla.** İki yönlü ilişki ek karmaşıklık getirir.
- `@OneToMany`'yi parent üzerinden child'ları yönetmen gerektiğinde ekle (sipariş–kalem gibi).
- `@OneToMany` `mappedBy` olmadan kullanılırsa Hibernate gereksiz bir **ara tablo** oluşturur.

## 4. Cascade ve orphanRemoval

```java
@Entity
@Table(name = "orders") // "order" SQL'de ayrılmış kelime
public class Order {

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }
}

@Entity
public class OrderItem {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    private int quantity;
    private BigDecimal unitPrice;
}
```

```java
order.addItem(new OrderItem(laptop, 1, laptop.getPrice()));
orderRepository.save(order); // kalemler de otomatik kaydedilir
```

| Ayar | Davranış |
|---|---|
| `cascade = ALL` | Parent'a yapılan işlem (kaydet, sil) child'lara yayılır |
| `orphanRemoval = true` | Listeden çıkarılan child veritabanından silinir |

### Cascade'in Tehlikesi

```java
@ManyToOne(cascade = CascadeType.ALL) // FELAKET: ürün silinince kategori de silinir
private Category category;
```

- Cascade sadece **gerçek parent-child** ilişkilerinde kullanılır: child, parent olmadan anlamsızsa.
- **`@ManyToOne` tarafına asla cascade konmaz.** Cascade parent'tan child'a, `@OneToMany` tarafında akar.

## 5. Sonsuz Döngü Tuzağı

İki yönlü ilişkisi olan entity JSON'a çevrilirse: kategori → ürünler → kategori → ürünler... `StackOverflowError`.

- `@JsonIgnore` / `@JsonManagedReference` sorunu yamalar.
- **Asıl çözüm: Entity değil DTO dönmek.**
- Lombok'un `@Data` / `@ToString`'i de aynı döngüyü oluşturur. Entity'lerde `@Getter` / `@Setter` kullanılır, `equals` / `hashCode`'da ilişki alanları kullanılmaz.

## 6. @ManyToMany

```java
@ManyToMany
@JoinTable(name = "student_courses",
           joinColumns = @JoinColumn(name = "student_id"),
           inverseJoinColumns = @JoinColumn(name = "course_id"))
private Set<Course> courses = new HashSet<>();
```

Pratikte kaçınılır, çünkü ara tabloya ek bilgi (tarih, not, durum) eklenemez. Bunun yerine ara tablo ayrı bir entity yapılır ve iki `@ManyToOne` ile bağlanır. `OrderItem` bunun örneğidir: `Order` ile `Product` arasındaki çoka çok ilişkinin ek bilgili hali.

---

## 7. Eager ve Lazy Loading

| | Ne Zaman Yüklenir? |
|---|---|
| **EAGER** | Ana entity ile **hemen** |
| **LAZY** | İlişkiye **ilk erişildiğinde** |

### Varsayılanlar

| İlişki | Varsayılan |
|---|---|
| `@ManyToOne` | **EAGER** ⚠️ |
| `@OneToOne` | **EAGER** ⚠️ |
| `@OneToMany` | LAZY |
| `@ManyToMany` | LAZY |

**EAGER neden tehlikeli?**
- Zincirleme yükleme: kalem → ürün → kategori → ...
- Sorgu bazında **kapatılamaz.**

> **Kural:** Tüm ilişkileri LAZY yap, ihtiyacın olan sorguda açıkça birlikte yükle.

### Lazy Nasıl Çalışır?

Hibernate gerçek entity yerine ondan **türeyen bir proxy** koyar. Proxy'de sadece id vardır. `getName()` gibi bir alana erişildiğinde proxy veritabanına gidip gerçek veriyi yükler.

> Entity'lerin `final` (dolayısıyla `record`) olamamasının bir sebebi de bu.

## 8. LazyInitializationException

```java
@Transactional(readOnly = true)
public Product findEntity(Long id) { ... } // transaction biter, entity detached olur

// Controller'da:
product.getCategory().getName(); // LazyInitializationException: no Session
```

Detached entity'nin lazy alanına erişilince proxy veritabanına gidemez, çünkü persistence context kapanmıştır.

### Open Session in View (OSIV)

`spring.jpa.open-in-view` varsayılan olarak **`true`**: Persistence context istek sonuna kadar açık kalır, bu yüzden hata alınmayabilir.

**Sorunları:**
- İstek boyunca veritabanı bağlantısı meşgul kalır.
- Controller'da ve JSON serileştirmede fark edilmeden sorgu çalışır, N+1 gizlenir.
- Veritabanı erişimi service dışına sızar.

```properties
spring.jpa.open-in-view=false
```

### Çözüm

Gereken veriyi **transaction içinde** hazırla. DTO dönüşümünü service'te yap:

```java
@Transactional(readOnly = true)
public ProductResponse findById(Long id) {
    Product product = repository.findById(id).orElseThrow(...);
    return new ProductResponse(product.getId(), product.getName(),
                               product.getCategory().getName());
}
```

## 9. N+1 Problemi

```java
repository.findAll().stream()
        .map(p -> new ProductResponse(p.getId(), p.getName(), p.getCategory().getName()))
        .toList();
```

```sql
select ... from products                       -- 1 sorgu
select ... from categories where id = ?        -- her ürün için 1 sorgu
select ... from categories where id = ?
...
```

**1 + N sorgu.** Geliştirmede 10 kayıtla fark edilmez, production'da sayfa saniyeler sürer.

> **EAGER çözüm değildir.** JPQL sorgularında Hibernate yine her eleman için ayrı sorgu atar, üstelik bunu ihtiyaç olmayan sorgularda da yapar.

## 10. N+1 Çözümleri

### JOIN FETCH

```java
@Query("SELECT p FROM Product p JOIN FETCH p.category")
List<Product> findAllWithCategory();
```

Tek sorgu. İlişki proxy değil, yüklü gerçek nesne olarak gelir.

### @EntityGraph

```java
@EntityGraph(attributePaths = {"category"})
List<Product> findByPriceLessThan(BigDecimal price);

@Override
@EntityGraph(attributePaths = {"category"})
List<Product> findAll();
```

JPQL yazmadan, türetilmiş ve hazır metotlarda da çalışır. İç içe: `{"items", "items.product"}`

### DTO Projection

```java
public record ProductSummary(Long id, String name, String categoryName) {}

@Query("""
       SELECT new com.example.shop.dto.ProductSummary(p.id, p.name, c.name)
       FROM Product p JOIN p.category c
       """)
List<ProductSummary> findSummaries();
```

- En verimlisi: sadece gereken kolonlar, persistence context'e entity girmez, dirty checking yok.
- Sınıfın **tam paket adı** yazılmalı.
- Dönen nesneler üzerinde değişiklik yapılıp kaydedilemez.

### Batch Fetching

```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=50
```

```sql
select ... from categories where id in (?, ?, ... 50 tane)
```

Kod değiştirmeden 1 + N sorguyu 1 + N/50'ye indirir. Genel güvenlik ağı.

### Hangisi Ne Zaman?

| Durum | Çözüm |
|---|---|
| Entity'yi ilişkisiyle yükleyip üzerinde iş yapılacak | `JOIN FETCH` / `@EntityGraph` |
| Okuma amaçlı liste, birkaç alan yeterli | DTO projection |
| Genel güvenlik ağı | `default_batch_fetch_size` |

## 11. JOIN FETCH Tuzakları

**Koleksiyon + Sayfalama:**

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items")
Page<Order> findAllWithItems(Pageable pageable); // tehlikeli
```

Join'de her sipariş kalem sayısı kadar satıra çoğalır, `LIMIT` doğru çalışamaz. Hibernate **tüm kayıtları belleğe çekip** sayfalamayı Java'da yapar (HHH90003004 uyarısı).
**Çözüm:** Sayfalamayı ilişkisiz yapıp batch fetching kullanmak veya önce id'leri sayfalayıp ikinci sorguda `JOIN FETCH` yapmak.

> `@ManyToOne` ilişkilerinde bu sorun yoktur, satırlar çoğalmaz.

**Birden Fazla List Koleksiyonu:**
Aynı sorguda iki `List` koleksiyonu `JOIN FETCH` edilirse `MultipleBagFetchException` alınır (kartezyen çarpım). Koleksiyonlar ayrı sorgularda çekilmeli veya batch fetching kullanılmalı.

## 12. N+1'i Yakalamak

- `show-sql` açıkken aynı yapıda tekrarlayan `select ... where id = ?` satırları N+1 işaretidir.
- Sorgu sayısı ve süreleri için:

```properties
spring.jpa.properties.hibernate.generate_statistics=true
```

> Liste döndüren bir endpoint'te sorgu sayısı kayıt sayısıyla birlikte artıyorsa bir sorun vardır.

---
