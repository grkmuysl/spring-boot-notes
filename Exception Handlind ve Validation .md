# Exception Handling ve Validation
---

## Exception Handling

### 1. Kendi Exception Sınıfları

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) { super(message); }
}

public class BusinessException extends RuntimeException {
    public BusinessException(String message) { super(message); }
}
```

- Java'nın genel exception'ları (`IllegalArgumentException`) her yerden gelebilir; merkezi işleyici ayırt edemez. Anlamı belli olan kendi sınıfların yazılır.
- **Unchecked (`RuntimeException`) tercih edilir.** Checked olsaydı hatanın geçtiği her metoda `throws` yazmak gerekirdi; hata zaten en üstte yakalanacağı için bunun faydası yoktur.

### 2. Service Fırlatır, Controller Sadeleşir

```java
// Service
public ProductResponse findById(Long id) {
    Product product = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Ürün bulunamadı: " + id));
    return toResponse(product);
}

// Controller: sadece mutlu yol
@GetMapping("/{id}")
public ProductResponse getById(@PathVariable Long id) {
    return service.findById(id);
}
```

### 3. @RestControllerAdvice ile Merkezi Yönetim

```java
public record ErrorResponse(int status, String error, String message, LocalDateTime timestamp) {}

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return build(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException ex) {
        return build(HttpStatus.BAD_REQUEST, ex.getMessage());
    }

    private ResponseEntity<ErrorResponse> build(HttpStatus status, String message) {
        return ResponseEntity.status(status)
                .body(new ErrorResponse(status.value(), status.getReasonPhrase(),
                                        message, LocalDateTime.now()));
    }
}
```

| Anotasyon | Görevi |
|---|---|
| `@ExceptionHandler(X.class)` | X tipinde hata fırlatılırsa bu metot çalışır |
| `@RestControllerAdvice` | `@ControllerAdvice` + `@ResponseBody`. Handler'lar tüm controller'lar için geçerli olur. |

- `@ExceptionHandler` bir controller'ın içine de yazılabilir; o zaman sadece o controller için geçerlidir ve merkezi olandan önce gelir.
- **Tüm hatalar tek formatta dönmelidir.** İstemci tek yerde işleyebilsin.

**Akış:** Service hata fırlatır → hata controller'dan yukarı çıkar → Spring tipine uyan handler'ı bulur → handler HTTP cevabını üretir.

### 4. Beklenmeyen Hatalar İçin Güvenlik Ağı

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleUnexpected(Exception ex) {
    log.error("Beklenmeyen hata", ex);
    return build(HttpStatus.INTERNAL_SERVER_ERROR, "Beklenmeyen bir hata oluştu");
}
```

- **Durum kodu `500` olmalı.** Beklenmeyen hata istemcinin suçu değildir.
- **`ex.getMessage()` istemciye dönülmez.** Tablo adı, dosya yolu, SQL parçası gibi iç detaylar sızabilir (güvenlik açığı).
- **Mutlaka loglanır.** Aksi halde hata sessizce yutulur.

### 5. Hangi Handler Çalışır?

Spring **en spesifik eşleşmeyi** seçer. `BusinessException` fırlatılırsa ve hem `BusinessException` hem `RuntimeException` handler'ı varsa, `BusinessException` handler'ı çalışır. Genel olanlar sadece başka hiçbiri uymazsa devreye girer.

### 6. Alternatifler

**`@ResponseStatus`:** Exception sınıfına yazılır, handler gerektirmez. Ama cevap gövdesi kontrol edilemez ve HTTP kavramı iş katmanına sızar. Küçük denemeler için uygundur.

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException { ... }
```

**`ProblemDetail` (Spring Boot 3):** Hata cevapları için RFC 9457 standardı. `type`, `title`, `status`, `detail`, `instance` alanlarını içerir.

```java
@ExceptionHandler(ResourceNotFoundException.class)
public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
    return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
}
```

---

## Validation

### 1. Kurulum

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

> **Dikkat:** Bağımlılık eksikse anotasyonlar derlenir, uygulama çalışır ama **hiçbir doğrulama yapılmaz** ve hata da alınmaz.

### 2. DTO'ya Kurallar

```java
public record ProductRequest(
        @NotBlank(message = "Ürün adı boş olamaz")
        @Size(max = 100, message = "Ürün adı en fazla 100 karakter olabilir")
        String name,

        @NotNull(message = "Fiyat zorunludur")
        @Positive(message = "Fiyat sıfırdan büyük olmalıdır")
        BigDecimal price,

        @PositiveOrZero(message = "Stok negatif olamaz")
        int stock
) {}
```

| Anotasyon | Kullanım |
|---|---|
| `@NotNull`, `@NotEmpty`, `@NotBlank` | Boşluk kontrolleri |
| `@Size(min, max)` | String uzunluğu, liste eleman sayısı |
| `@Min`, `@Max` | Tam sayı aralığı |
| `@Positive`, `@PositiveOrZero`, `@DecimalMin` | Sayısal kontroller |
| `@Email`, `@Pattern(regexp)` | Format |
| `@Past`, `@Future` | Tarihler |

### 3. @NotNull, @NotEmpty, @NotBlank Farkı

| Değer | `@NotNull` | `@NotEmpty` | `@NotBlank` |
|---|---|---|---|
| `null` | ❌ | ❌ | ❌ |
| `""` | ✅ | ❌ | ❌ |
| `"   "` | ✅ | ✅ | ❌ |
| `"Laptop"` | ✅ | ✅ | ✅ |

- Metin alanları → `@NotBlank`
- Sayı ve nesneler → `@NotNull`
- Listeler → `@NotEmpty`

> **Dikkat:** `@Positive`, `@Email` gibi anotasyonlar **null değeri geçerli sayar** ("değer varsa uygun olsun" demektir). Zorunlu alanlarda `@NotNull` / `@NotBlank` ile birlikte kullanılmalıdır.

### 4. Doğrulamayı Tetiklemek: @Valid

```java
@PostMapping
public ResponseEntity<ProductResponse> create(@Valid @RequestBody ProductRequest request) { ... }
```

- `@Valid` olmadan anotasyonlar çalışmaz.
- Doğrulama başarısızsa metoda hiç girilmez, `MethodArgumentNotValidException` fırlatılır.
- İç içe nesnelerde doğrulama otomatik inmez, iç alana da `@Valid` eklenir:

```java
public record OrderRequest(
        @NotNull @Valid AddressRequest address,
        @NotEmpty List<@Valid OrderItemRequest> items
) {}
```

### 5. Validation Hatalarını Döndürmek

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<Map<String, Object>> handleValidation(MethodArgumentNotValidException ex) {
    Map<String, String> fieldErrors = new HashMap<>();
    ex.getBindingResult().getFieldErrors()
            .forEach(err -> fieldErrors.put(err.getField(), err.getDefaultMessage()));

    return ResponseEntity.badRequest().body(Map.of(
            "status", 400,
            "message", "Geçersiz istek",
            "errors", fieldErrors));
}
```

```json
{
  "status": 400,
  "message": "Geçersiz istek",
  "errors": {
    "name": "Ürün adı boş olamaz",
    "price": "Fiyat sıfırdan büyük olmalıdır"
  }
}
```

**Tüm hatalar tek seferde döner.** Elle yazılan `if` zincirinde kullanıcı hataları birer birer görürdü.

### 6. Path Variable ve Request Param Doğrulama

```java
@GetMapping("/{id}")
public ProductResponse getById(@PathVariable @Positive Long id) { ... }
```

- **Spring Boot 3.2+:** Doğrudan çalışır, hata durumunda `HandlerMethodValidationException` fırlatılır.
- **Eski sürümler:** Controller'a `@Validated` eklenir, `ConstraintViolationException` fırlatılır.

## Sık Yapılan Hatalar

- Validation bağımlılığını eklememek (sessizce çalışmaz)
- `@Valid` yazmayı unutmak
- Metin alanlarında `@NotBlank` yerine `@NotEmpty` kullanmak
- `@Email`, `@Positive` gibi anotasyonların null'ı geçirdiğini unutmak
- Beklenmeyen hatalarda `ex.getMessage()`'ı istemciye dönmek veya loglamamak
- Beklenmeyen hatalara `500` yerine `400` dönmek
