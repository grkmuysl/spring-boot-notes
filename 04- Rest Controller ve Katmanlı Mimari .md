# REST Controller ve Katmanlı Mimari

## Katmanlı Mimari

Kod sorumluluklarına göre üç katmana ayrılır:

| Katman | Görevi | Bilmediği |
|---|---|---|
| **Controller** | HTTP isteğini alır, service'i çağırır, cevabı döner | İş kuralları |
| **Service** | İş kurallarını uygular | HTTP, veritabanı detayları |
| **Repository** | Veriye erişir (kaydet, bul, sil) | İş kuralları |

- Bağımlılıklar **tek yönde** akar: Controller → Service → Repository
- Controller repository'yi doğrudan çağırmaz.
- Her katman alttakinin **ne** yaptığını bilir, **nasıl** yaptığını bilmez. Veri bellekten veritabanına taşınsa controller ve service etkilenmez.

```
com.example.shop
├── controller   → ProductController
├── service      → ProductService
├── repository   → ProductRepository
├── model        → Product
└── dto          → ProductRequest, ProductResponse
```

## DTO (Data Transfer Object)

Model sınıfı doğrudan API'ye açılmaz. DTO, API'nin sözleşmesidir: dışarıdan ne kabul edilir, dışarıya ne gösterilir.

```java
public record ProductRequest(String name, BigDecimal price, BigDecimal costPrice, int stock) {}
public record ProductResponse(Long id, String name, BigDecimal price, boolean inStock) {}
```

- **Girişte:** Request'te `id` yok. Aksi halde istemci mevcut bir ürünün üzerine yazabilir.
- **Çıkışta:** Response'ta `costPrice` yok. Hassas alanlar dışarı sızmaz.
- Model değişse bile API sözleşmesi bozulmak zorunda kalmaz.
- `record`: Sadece veri taşıyan değişmez sınıflar için kısa yazım (Java 16+).

## Repository

```java
@Repository
public class ProductRepository {
    private final Map<Long, Product> products = new ConcurrentHashMap<>();
    private final AtomicLong idGenerator = new AtomicLong();
    // save, findById, findAll, deleteById
}
```

Repository singleton olduğu için paylaşılan veri thread-safe yapılarla tutulur (`ConcurrentHashMap`, `AtomicLong`).

## Service

```java
@Service
public class ProductService {
    private final ProductRepository repository;

    public ProductResponse create(ProductRequest request) {
        if (request.price().compareTo(request.costPrice()) < 0) {
            throw new IllegalArgumentException("Satış fiyatı alış fiyatından düşük olamaz");
        }
        Product product = new Product(null, request.name(), request.price(),
                                      request.costPrice(), request.stock());
        return toResponse(repository.save(product));
    }
}
```

- İş kuralları burada yaşar. Böylece istek HTTP'den, zamanlanmış görevden veya başka bir yerden gelse de aynı kural çalışır.
- DTO ↔ model dönüşümü genelde service'te veya ayrı bir mapper sınıfında yapılır.

## Controller

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    private final ProductService service;

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getById(@PathVariable Long id) {
        return service.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<ProductResponse> create(@RequestBody ProductRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(service.create(request));
    }
}
```

| Anotasyon | Görevi |
|---|---|
| `@RestController` | `@Controller` + `@ResponseBody`. Dönüş değeri JSON'a çevrilir. |
| `@RequestMapping` | Sınıf seviyesinde ortak yol |
| `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping` | HTTP metodunu Java metoduna bağlar |
| `@PathVariable` | URL içindeki değer: `/products/5` |
| `@RequestParam` | Sorgu parametresi: `/products?category=x` |
| `@RequestBody` | İstek gövdesindeki JSON'u nesneye çevirir |
| `ResponseEntity` | Durum kodu, başlık ve gövdeyi kontrol eder |

> Sadece `@Controller` kullanılırsa Spring dönüş değerini HTML şablonu adı sanar.

### @PathVariable mı, @RequestParam mı?

- Belirli **bir kaynağı** tanımlıyorsa → `@PathVariable`
- Listeyi **filtreliyor veya sıralıyorsa** → `@RequestParam`

```
GET /api/products/12/comments                      → ürüne ait alt kaynak
GET /api/products?category=elektronik&sort=price   → filtreleme ve sıralama
```

## REST Kuralları

URL kaynağı (isim, çoğul) gösterir, eylemi HTTP metodu belirtir.
`/getProduct?id=5` değil, `GET /api/products/5`.

| İşlem | Metot ve URL | Başarılı Cevap |
|---|---|---|
| Listele | `GET /api/products` | 200 OK |
| Tek ürün | `GET /api/products/5` | 200 OK (yoksa 404) |
| Oluştur | `POST /api/products` | 201 Created |
| Güncelle | `PUT /api/products/5` | 200 OK |
| Sil | `DELETE /api/products/5` | 204 No Content |

## Sık Yapılan Hatalar

- Controller'ın repository'ye doğrudan erişmesi
- İş kurallarının controller'a yazılması
- Model sınıfının giriş veya çıkışta doğrudan kullanılması (DTO yerine)
- Her cevabın `200 OK` dönmesi (oluşturmada 201, silmede 204 olmalı)
