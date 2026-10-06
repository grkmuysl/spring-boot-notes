# API Dokümantasyonu: OpenAPI ve Swagger

## 1. Sorun: Dokümantasyon Eskir

API'yi kullanacak kişinin bilmesi gerekenler:
- Hangi endpoint'ler var, hangi HTTP metoduyla çağrılıyor?
- İstek gövdesi hangi alanları içeriyor, hangileri zorunlu?
- Cevap neye benziyor, hangi hata kodları dönebilir?
- Hangi endpoint'ler giriş gerektiriyor, token nasıl gönderiliyor?

**Elle yazılan dokümantasyon** (Word, Confluence, README, Postman koleksiyonu) kod değiştiğinde güncellenmeyi unutulur ve **eskir.**

> **Çözüm:** Dokümantasyonu **koddan üretmek.** Kod değişince doküman da değişir. Tek doğruluk kaynağı: kodun kendisi.

## 2. OpenAPI ve Swagger

| İsim | Nedir? |
|---|---|
| **OpenAPI Specification** | REST API'leri tarif eden **standart format** (JSON / YAML). Eski adı "Swagger Specification". |
| **Swagger UI** | OpenAPI dosyasını tarayıcıda gezilebilir, istek atılabilir arayüze dönüştüren **araç** |
| **springdoc-openapi** | Spring kodundan OpenAPI dosyasını **üreten** kütüphane |

> OpenAPI standart, Swagger UI onu görselleştiren araç (JPA / Hibernate ilişkisine benzer).

> ⚠️ **Springfox** kullanma: Geliştirilmiyor, Spring Boot 3 ile çalışmıyor.

## 3. Kurulum

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.9</version>
</dependency>
```

- Spring Boot sürümünü **yönetmez**, sürüm elle yazılır.
- Spring Boot 3 → springdoc **2.x**. Boot yükseltilince uyumluluk tablosuna bakılır.

```
/v3/api-docs        → OpenAPI dosyası (JSON)
/swagger-ui.html    → Swagger UI
```

### Security

```java
.requestMatchers("/v3/api-docs/**", "/swagger-ui/**", "/swagger-ui.html").permitAll()
```

> Yazılmazsa **401** alınır ("varsayılan olarak her şey kapalı").

### Hiç Anotasyon Yazmadan Gelenler

- Tüm controller'lar, endpoint'ler, HTTP metotları ve yollar
- `@PathVariable`, `@RequestParam` parametreleri
- İstek ve cevap DTO'larının alanları ve tipleri
- **Validation anotasyonları:**

| Validation | Dokümantasyonda |
|---|---|
| `@NotBlank`, `@NotNull` | required (zorunlu) |
| `@Size(max = 100)` | maxLength: 100 |
| `@Positive` | minimum |

> Validation kuralları = API sözleşmesi, artık okunabilir belge.
> **DTO'ların faydası:** Dokümantasyon iç modeli değil, bilinçli tasarlanmış sözleşmeyi gösterir. Entity dönülseydi `costPrice` gibi alanlar da görünürdü.

- **"Try it out":** Tarayıcıdan doğrudan istek atılır, çoğu zaman Postman'e gerek kalmaz.

## 4. Dokümantasyonu Zenginleştirmek

Otomatik dokümantasyon **yapıyı** gösterir, **anlamı** göstermez.

### Controller Seviyesi

```java
@RestController
@RequestMapping("/api/products")
@Tag(name = "Ürünler", description = "Ürün kataloğu işlemleri")
public class ProductController {

    @Operation(
            summary = "Ürün oluştur",
            description = "Yeni bir ürün oluşturur. Satış fiyatı alış fiyatından düşük olamaz.")
    @ApiResponses({
            @ApiResponse(responseCode = "201", description = "Ürün oluşturuldu"),
            @ApiResponse(responseCode = "400", description = "Geçersiz istek veya iş kuralı ihlali",
                    content = @Content(schema = @Schema(implementation = ErrorResponse.class))),
            @ApiResponse(responseCode = "401", description = "Giriş yapılmamış",
                    content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    })
    @PostMapping
    public ResponseEntity<ProductResponse> create(@Valid @RequestBody ProductRequest request) { ... }

    @Operation(summary = "Ürün getir")
    @ApiResponse(responseCode = "404", description = "Ürün bulunamadı",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    @GetMapping("/{id}")
    public ProductResponse getById(
            @Parameter(description = "Ürün kimliği", example = "42") @PathVariable Long id) { ... }
}
```

| Anotasyon | Görevi |
|---|---|
| `@Tag` | Endpoint'leri gruplar (varsayılan: `product-controller`) |
| `@Operation` | `summary`: kısa başlık, `description`: detay ve iş kuralları |
| `@ApiResponse` | Dönebilecek cevaplar ve şemaları |
| `@Parameter` | Parametre açıklaması, `example` "Try it out"ta hazır gelir |


### DTO Seviyesi

```java
public record ProductRequest(
        @Schema(description = "Ürün adı", example = "Kablosuz Kulaklık")
        @NotBlank @Size(max = 100)
        String name,

        @Schema(description = "Satış fiyatı (TL)", example = "1499.90")
        @NotNull @Positive
        BigDecimal price,

        @Schema(description = "Alış fiyatı (TL). Müşteriye gösterilmez.", example = "950.00")
        @NotNull @PositiveOrZero
        BigDecimal costPrice
) {}
```

- Örnek değerler "Try it out" gövdesini gerçekçi verilerle doldurur.

### Ne Kadar Anotasyon?

Her şeye anotasyon → Controller iş kodundan çok anotasyondan oluşur, okunmaz.

**En yüksek değer:**
1. Her endpoint için anlamlı **`summary`**
2. Her endpoint'in **hata cevapları** (`@ApiResponse`)
3. Kendiliğinden anlaşılmayan alanlar için **`@Schema`** açıklaması ve örnek

> `name` = "ürün adı" açıklaması gereksiz. `costPrice` = "müşteriye gösterilmez" açıklaması değerli.

### Sayfalama

```java
@GetMapping
public Page<ProductResponse> getAll(@ParameterObject Pageable pageable) { ... }
```

`@ParameterObject`: `page`, `size`, `sort` ayrı sorgu parametreleri olarak görünür.

## 5. Genel Bilgiler ve JWT

```java
@Configuration
@OpenAPIDefinition(
        info = @Info(
                title = "Shop API",
                version = "v1",
                description = "E-ticaret uygulaması REST API'si"),
        security = @SecurityRequirement(name = "bearerAuth"))
@SecurityScheme(
        name = "bearerAuth",
        type = SecuritySchemeType.HTTP,
        scheme = "bearer",
        bearerFormat = "JWT")
public class OpenApiConfig {
}
```

- `@SecurityScheme`: Kimlik doğrulama yöntemi (`Authorization: Bearer ...`)
- `security = @SecurityRequirement(...)`: Tüm endpoint'lere uygulanır.

### Swagger UI'da Token Kullanmak

1. `/api/auth/login` → "Try it out" → token al
2. Sağ üstteki **"Authorize"** butonuna token'ı yapıştır
3. Tüm isteklere `Authorization` başlığı otomatik eklenir

### Token Gerektirmeyen Endpoint'ler

```java
@SecurityRequirements   // boş: güvenlik gereksinimi yok
@PostMapping("/login")
public LoginResponse login(@Valid @RequestBody LoginRequest request) { ... }
```

## 6. Production'da Açık mı, Kapalı mı?

Dokümantasyon API'nin **tam haritasıdır** (admin endpoint'leri dahil).

| API Türü | Dokümantasyon |
|---|---|
| Dışa açık (dış geliştiriciler için) | Bilinçli olarak paylaşılır |
| Sadece kendi ön yüzün için | Saldırgana keşifte kolaylık sağlar, kapalı tutulmalı |

### Profillerle Yönetim

```yaml
# application.yml (güvenli varsayılan: KAPALI)
springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false
```

```yaml
# application-dev.yml
springdoc:
  api-docs:
    enabled: true
  swagger-ui:
    enabled: true
```

> Güvenli varsayılan ilkesi: Profil unutulursa dokümantasyon sızmaz.

### @Hidden

```java
@Hidden
@RestController
@RequestMapping("/api/admin")
public class AdminController { ... }
```

> ⚠️ **`@Hidden` güvenlik önlemi değildir.** Endpoint çalışmaya devam eder, URL'yi bilen istek atabilir. Sadece haritada gösterilmez. **Asıl koruma Spring Security'dir.**


---
