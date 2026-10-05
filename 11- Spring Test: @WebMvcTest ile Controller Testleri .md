# Test Yazmak - Bölüm 2: @WebMvcTest ile Controller Testleri

## 1. Controller Testi Neyi Test Eder?

```java
// YETERSİZ: Anotasyonların hiçbiri çalışmaz
ProductController controller = new ProductController(mockService);
controller.getById(1L);
```

Controller'ın asıl işi **anotasyonlarla** yapılır ve onları Spring işler. Controller testi şunları doğrular:

- İstek doğru metoda gidiyor mu? (`@GetMapping`, `@PathVariable`)
- JSON doğru çevriliyor mu? (`@RequestBody`, cevap formatı)
- Geçersiz veri 400 dönüyor mu? (`@Valid`)
- Doğru durum kodu dönüyor mu? (200, 201, 204)
- Hatalar `GlobalExceptionHandler` ile doğru cevaba çevriliyor mu?
- Security kuralları doğru uygulanıyor mu? (401, 403)

> Controller testi **iş mantığını değil, HTTP sözleşmesini** test eder. İş mantığı service testlerinde test edilir; burada service mock'lanır.

## 2. @WebMvcTest

```java
@WebMvcTest(ProductController.class)
class ProductControllerTest { ... }
```

Bir **dilim (slice) testidir**: Spring context'ini sadece web katmanıyla ayağa kaldırır.

| Yüklenenler | Yüklenmeyenler |
|---|---|
| Belirtilen controller | `@Service` sınıfları |
| `@RestControllerAdvice` | `@Repository` / JPA |
| `Filter` bean'leri (JWT filtresi dahil) | Diğer `@Component` sınıfları |
| Jackson (`ObjectMapper`), validation | Veritabanı |
| Spring Security altyapısı | |
| `MockMvc` | |

- `ProductController.class` parametresi sadece o controller'ı yükler. Parametresiz yazılırsa tüm controller'lar yüklenir.

### Eksik Bean'ler: @MockitoBean

```
NoSuchBeanDefinitionException: No qualifying bean of type 'ProductService' available
```

Slice testlerinde **en sık karşılaşılan hata.** Anlamı: Bu bean test context'inde yok, sağlanmalı.

```java
@MockitoBean
private ProductService productService;
```

| | `@Mock` | `@MockitoBean` |
|---|---|---|
| **Spring'den haberi var mı?** | Hayır | Evet, mock'u context'e bean olarak koyar |
| **Nasıl enjekte edilir?** | `@InjectMocks` ile elle | Spring normal DI ile |
| **Ne zaman?** | Spring context'i **olmayan** testlerde | Spring context'i **olan** testlerde |

> **Sürüm notu:** `@MockitoBean` Spring Boot 3.4 ile geldi. Eski sürümlerde aynı işi **`@MockBean`** yapar (3.4'te deprecated).

## 3. MockMvc

Gerçek sunucu başlatmadan, port açmadan HTTP isteğini **simüle eder.** İstek filtreler, `DispatcherServlet`, controller, exception handler ve JSON çevirme hattının tamamından geçer.

```java
@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private ProductService productService;

    @Test
    void getById_whenProductExists_returns200WithJson() throws Exception {
        when(productService.findById(1L)).thenReturn(
                new ProductResponse(1L, "Laptop", new BigDecimal("25000"), true));

        mockMvc.perform(get("/api/products/1"))
                .andExpect(status().isOk())
                .andExpect(content().contentType(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.name").value("Laptop"));
    }
}
```

- `perform(...)`: İsteği gönderir. `get`, `post`, `put`, `delete` → `MockMvcRequestBuilders`
- `andExpect(...)`: Cevabı doğrular.
- Test sınıflarında field injection (`@Autowired`) kabul görmüş bir istisnadır; test sınıfını JUnit oluşturur.

### JsonPath

| İfade | Anlamı |
|---|---|
| `$.name` | Kök nesnenin `name` alanı |
| `$.category.name` | İç içe nesne |
| `$[0].name` | Dizinin ilk elemanı |
| `$.content.length()` | Dizinin eleman sayısı |
| `$.errors.price` | Map içindeki anahtar |

## 4. POST İsteği

```java
@Autowired
private ObjectMapper objectMapper;

@Test
@WithMockUser
void create_withValidRequest_returns201() throws Exception {
    ProductRequest request = new ProductRequest("Laptop", new BigDecimal("25000"),
                                                new BigDecimal("20000"), 5);
    when(productService.create(any(ProductRequest.class))).thenReturn(
            new ProductResponse(1L, "Laptop", new BigDecimal("25000"), true));

    mockMvc.perform(post("/api/products")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1));
}
```

- Gövde `ObjectMapper` ile oluşturulur; DTO değişince test güncel kalır.

> ⚠️ `contentType` unutulursa **415 Unsupported Media Type** alınır.

## 5. Validation Testi

```java
@Test
@WithMockUser
void create_withInvalidRequest_returns400WithFieldErrors() throws Exception {
    ProductRequest invalid = new ProductRequest("", new BigDecimal("-50"), null, -3);

    mockMvc.perform(post("/api/products")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(invalid)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors.name").value("Ürün adı boş olamaz"))
            .andExpect(jsonPath("$.errors.price").exists());

    verify(productService, never()).create(any());
}
```

**Tek testle kanıtlananlar:**
- DTO anotasyonları çalışıyor.
- Controller'da `@Valid` yazılı.
- Validation bağımlılığı ekli (eksikse sessizce çalışmaz).
- `MethodArgumentNotValidException` handler'ı doğru formatı dönüyor.
- Geçersiz istek **service'e hiç ulaşmıyor.**

## 6. Exception Handling Testi

```java
@Test
void getById_whenProductDoesNotExist_returns404() throws Exception {
    when(productService.findById(99L))
            .thenThrow(new ResourceNotFoundException("Ürün bulunamadı: 99"));

    mockMvc.perform(get("/api/products/99"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.message").value("Ürün bulunamadı: 99"));
}

@Test
void getById_whenUnexpectedError_returns500WithoutLeakingDetails() throws Exception {
    when(productService.findById(1L))
            .thenThrow(new RuntimeException("SQL hatası: products tablosu bulunamadı"));

    mockMvc.perform(get("/api/products/1"))
            .andExpect(status().isInternalServerError())
            .andExpect(jsonPath("$.message").value("Beklenmeyen bir hata oluştu"));
}
```

- Test edilen: `GlobalExceptionHandler`'ın hatayı doğru koda ve formata çevirmesi.
- `@RestControllerAdvice` otomatik yüklenir.
- İkinci test, iç detayların (SQL, tablo adı) istemciye **sızmadığını** garanti eder.

## 7. Security Testleri

### Kurulum

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

> `spring-boot-starter-test` içinde **gelmez**, ayrıca eklenir.

```java
@WebMvcTest(ProductController.class)
@Import({SecurityConfig.class, JsonAuthenticationEntryPoint.class, JsonAccessDeniedHandler.class})
class ProductControllerTest {

    @Autowired private MockMvc mockMvc;
    @Autowired private ObjectMapper objectMapper;

    @MockitoBean private ProductService productService;
    @MockitoBean private JwtService jwtService;               // JWT filtresinin bağımlılığı
    @MockitoBean private UserDetailsService userDetailsService; // JWT filtresinin bağımlılığı
}
```

- **Kendi `SecurityConfig`'in `@Import` edilmeli.** Aksi halde varsayılan ayarlar (her şey kilitli, CSRF açık) devreye girebilir ve gerçek kurallar test edilmez.
- JWT filtresi bir `Filter` olduğu için **yüklenir**, ama bağımlılıkları yüklenmez.

| Bean | Gerçeği mi, Mock mu? |
|---|---|
| Test edilen davranışın parçası (`SecurityConfig`, hata handler'ları) | `@Import` ile **gerçeği** |
| Davranışın parçası değil (`JwtService`, service'ler) | `@MockitoBean` ile **mock** |

### @WithMockUser

İsteği atmadan önce `SecurityContext`'e **kimliği doğrulanmış sahte bir kullanıcı** koyar.

```java
@WithMockUser                                     // kullanıcı "user", rol USER
@WithMockUser(username = "ali", roles = "ADMIN")  // özelleştirilmiş
```

- `roles = "ADMIN"` → `ROLE_ADMIN` yetkisi (`hasRole` ile uyumlu).
- Token, `UserDetailsService`, şifre kontrolü **atlanır.**

> ⚠️ `@WithMockUser` JWT filtresini **test etmez**, sadece **authorization kurallarını** test eder. JWT akışı uçtan uca test ile doğrulanır.

### Dört Temel Senaryo

```java
// Kural: .requestMatchers(HttpMethod.DELETE, "/api/products/**").hasRole("ADMIN")

@Test
void getById_withoutLogin_isPublic() throws Exception {
    when(productService.findById(1L)).thenReturn(...);
    mockMvc.perform(get("/api/products/1"))
            .andExpect(status().isOk());
}

@Test
void create_withoutLogin_returns401() throws Exception {
    mockMvc.perform(post("/api/products").contentType(MediaType.APPLICATION_JSON).content("{}"))
            .andExpect(status().isUnauthorized())
            .andExpect(jsonPath("$.status").value(401));
    verifyNoInteractions(productService);
}

@Test
@WithMockUser(roles = "USER")
void delete_asUser_returns403() throws Exception {
    mockMvc.perform(delete("/api/products/1"))
            .andExpect(status().isForbidden());
    verifyNoInteractions(productService);
}

@Test
@WithMockUser(roles = "ADMIN")
void delete_asAdmin_returns204() throws Exception {
    mockMvc.perform(delete("/api/products/1"))
            .andExpect(status().isNoContent());
    verify(productService).delete(1L);
}
```

| Test | Kanıtladığı |
|---|---|
| Giriş yok, açık endpoint → 200 | `permitAll()` çalışıyor, JWT filtresi token yokken hata vermiyor |
| Giriş yok, korumalı endpoint → **401** | Kimlik kontrolü + `JsonAuthenticationEntryPoint` devrede, istek controller'a ulaşmıyor |
| USER, admin işlemi → **403** | Yetki kontrolü çalışıyor |
| ADMIN, admin işlemi → 204 | Doğru kullanıcı **erişebiliyor** |

- `verifyNoInteractions(mock)`: Mock'un **hiçbir** metodu çağrılmadı.
- Sadece reddi test etmek yetmez: **Her şeyi reddeden bozuk bir kural da geçer.** Doğru kullanıcının erişimi de test edilmeli.
- Kural sırası hatası (`/api/**`'ın `/api/admin/**`'ı yutması) 403 testiyle yakalanır.

### İstek Bazında Kullanıcı

```java
mockMvc.perform(delete("/api/products/1").with(user("admin").roles("ADMIN")))
        .andExpect(status().isNoContent());
```

Aynı testte farklı kullanıcılar gerektiğinde kullanılır.

### CSRF

CSRF açık uygulamalarda POST / PUT / DELETE testlerine **`.with(csrf())`** eklenmezse **403** alınır.

> "Neden 403 alıyorum?" sorusunda CSRF her zaman kontrol listesinde olmalı.

### ⚠️ @PreAuthorize Burada Test Edilmez

- `@WebMvcTest`'te service **mock'lanır.** Mock gerçek kodu çalıştırmaz ve `@PreAuthorize`'ı işleyecek Spring proxy'si yoktur.
- USER rolüyle `@PreAuthorize("hasRole('ADMIN')")`'lı service metodunu çağıran endpoint testi **başarılı olur** ve yanlış güven verir.
- Burada sadece **URL bazlı kurallar** test edilir. Method security `@SpringBootTest` ile test edilir.

> Hangi testin neyi kapsadığını bilmek, testlerin kendisi kadar önemlidir.

---

## Sorular

**1. Controller metodunu doğrudan çağırarak test etmek neden yetersizdir?**
URL eşleme, JSON çevirme, `@Valid`, durum kodları, exception handler ve security filtreleri Spring tarafından işlenir. Metot doğrudan çağrıldığında hiçbiri çalışmaz.

**2. `@WebMvcTest` neyi yükler, neyi yüklemez?**
Controller, `@RestControllerAdvice`, filtreler, Jackson, validation, security altyapısı ve `MockMvc` yüklenir. Service, repository, diğer `@Component`'ler ve veritabanı yüklenmez.

**3. `@Mock` ile `@MockitoBean` farkı nedir?**
`@Mock` Spring'den habersiz düz mock'tur, birim testlerinde kullanılır. `@MockitoBean` mock'u Spring context'ine bean olarak koyar, context'li testlerde kullanılır. 3.4 öncesinde `@MockBean`.

**4. Slice testinde `NoSuchBeanDefinitionException` neden alınır?**
Gereken bean dilimin dışında kalmış ve yüklenmemiştir. `@MockitoBean` ile mock'lanmalı veya `@Import` ile eklenmelidir.

**5. MockMvc gerçek sunucu başlatır mı?**
Hayır. HTTP isteğini simüle eder, Spring MVC'nin tüm işlem hattından geçirir ama ağ katmanı yoktur.

**6. POST testinde 415 neden alınır?**
`contentType(MediaType.APPLICATION_JSON)` belirtilmemiştir.

**7. Validation testinde `verify(service, never())` neden önemlidir?**
Geçersiz isteğin service'e hiç ulaşmadığını, validation'ın kapıda durduğunu kanıtlar.

**8. `@WithMockUser` JWT filtresini test eder mi?**
Hayır. Sahte kullanıcıyı doğrudan `SecurityContext`'e koyar; token ve şifre kontrolü atlanır. Sadece authorization kurallarını test eder.

**9. `SecurityConfig` neden `@Import` edilmelidir?**
Edilmezse varsayılan security ayarları devreye girebilir ve uygulamanın gerçek kuralları test edilmez.

**10. Security testlerinde neden sadece reddin test edilmesi yetmez?**
Her isteği reddeden bozuk bir kural da bu testlerden geçer. Doğru rolün erişebildiği de test edilmelidir.

**11. `@WebMvcTest`'te `@PreAuthorize` neden çalışmaz?**
Service mock'lanır; mock'ta `@PreAuthorize`'ı işleyecek Spring proxy'si yoktur. Method security `@SpringBootTest` ile test edilir.

**12. CSRF açık uygulamada doğru rolle atılan POST testi neden 403 alır?**
CSRF token'ı eklenmemiştir. `.with(csrf())` gerekir.

**13. `verify(mock, never()).method()` ile `verifyNoInteractions(mock)` farkı nedir?**
İlki belirli bir metodun çağrılmadığını, ikincisi mock'un **hiçbir** metodunun çağrılmadığını doğrular.
