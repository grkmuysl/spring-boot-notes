# Test Yazmak - Bölüm 1: Temeller, JUnit 5 ve Mockito

## 1. Neden Test Yazıyoruz?

Elle test (Postman) küçük projede işe yarar, ama her değişiklikte tüm endpoint'leri tekrar denemek mümkün değildir. Bir yeri düzeltirken başka yeri bozmaya **regression** (gerileme) denir.

| Fayda | Açıklama |
|---|---|
| **Regression'ı önler** | Her değişiklikte saniyeler içinde her şeyin hâlâ çalıştığını kontrol eder |
| **Değiştirme cesareti** | Refactoring'den korkulmaz, bozulan şey hemen görülür |
| **Yaşayan dokümantasyon** | Test isimleri kodun nasıl davranması gerektiğini anlatır; eskirse kırılır |
| **Tasarım geri bildirimi** | Bir sınıfı test etmek zorsa, tasarımında sorun vardır |

## 2. Test Piramidi

```
            /\
           /  \        Uçtan uca (E2E): Az, yavaş, tüm sistem
          /----\
         /      \      Entegrasyon: Orta, birkaç bileşen birlikte
        /--------\
       /          \    Birim (Unit): Çok, hızlı, tek sınıf
      /------------\
```

Tabanda çok sayıda hızlı test, tepede az sayıda yavaş test.

| Seviye | Araç | Spring Context | Hız |
|---|---|---|---|
| Birim testi | JUnit + Mockito | Yok | Çok hızlı |
| Dilim (slice) testi | `@WebMvcTest`, `@DataJpaTest` | Sadece ilgili katman | Orta |
| Tam entegrasyon | `@SpringBootTest` | Tamamı | Yavaş |

> Katmanlar birbirinin **alternatifi değil, tamamlayıcısıdır.**

## 3. Kurulum

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

İçinde: **JUnit 5**, **Mockito**, **AssertJ**, Spring test araçları. `scope test`: Sadece testlerde kullanılır, uygulama paketine girmez.

```
src/main/java/com/example/shop/service/ProductService.java
src/test/java/com/example/shop/service/ProductServiceTest.java
```

- Test sınıfları **aynı paket yapısında** durur.
- Çalıştırma: IDE'deki yeşil ok veya `mvn test`
- Maven adı `Test` ile biten sınıfları çalıştırır.

## 4. JUnit 5 Temelleri

```java
class OrderTest {

    @Test
    void getTotal_sumsAllItems() {
        // Arrange
        Order order = new Order();
        order.addItem(new OrderItem(null, 2, new BigDecimal("100.00")));
        order.addItem(new OrderItem(null, 1, new BigDecimal("50.00")));

        // Act
        BigDecimal total = order.getTotal();

        // Assert
        assertThat(total).isEqualByComparingTo("250.00");
    }
}
```

- `@Test`: Metodun test olduğunu belirtir. Sınıf ve metotların `public` olması gerekmez.
- **Arrange-Act-Assert (AAA):** Hazırla → Çalıştır → Doğrula. (Given-When-Then aynı anlamda.)

> ⚠️ **`BigDecimal` tuzağı:** `equals` ölçeği de karşılaştırır. `250.00` ≠ `250`.
> `isEqualTo` yerine **`isEqualByComparingTo`** kullanılır (`compareTo` ile sadece değere bakar).

### İsimlendirme

```
metotAdı_durum_beklenenSonuç

create_whenNameAlreadyExists_throwsBusinessException
findById_whenProductDoesNotExist_throwsResourceNotFoundException
```

Alternatif: `@DisplayName("Aynı isimde ürün varsa hata fırlatır")`

### AssertJ Doğrulamaları

```java
assertThat(product.getName()).isEqualTo("Laptop");
assertThat(products).hasSize(3);
assertThat(products).extracting(Product::getName).containsExactly("A", "B", "C");
assertThat(optional).isEmpty();
assertThat(response.inStock()).isTrue();

assertThatThrownBy(() -> service.findById(99L))
        .isInstanceOf(ResourceNotFoundException.class)
        .hasMessageContaining("99");
```

- AssertJ, JUnit'in `assertEquals`'ından daha okunaklıdır ve parametre sırası karıştırılamaz.
- Exception doğrulanırken kod **lambda içinde** yazılır. Aksi halde exception doğrulamadan önce fırlatılır.

## 5. Bağımlılıkları Olan Sınıflar

Service testinde gerçek repository kullanılırsa:
- Test yavaşlar.
- Veritabanının durumuna bağlı hale gelir.
- Kırıldığında sorunun service'te mi veritabanında mı olduğu anlaşılamaz.

**Çözüm:** Gerçek bağımlılık yerine, davranışını bizim kontrol ettiğimiz **sahte bir nesne** vermek.

### Constructor Injection'ın Getirisi

```java
ProductService service = new ProductService(fakeRepository);
```

Spring yok, container yok, veritabanı yok. **Constructor injection** sayesinde bağımlılık doğrudan verilebilir. Field injection'da reflection veya Spring context gerekirdi.

> "Bir sınıfı test etmek zorsa, tasarımında sorun vardır."

## 6. Test Double'lar: Stub, Mock ve Diğerleri

Test sırasında gerçek bağımlılığın yerine geçen her sahte nesneye genel olarak **test double** denir (filmlerdeki dublör gibi). Beş türü vardır:

| Tür | Ne Yapar? | Örnek |
|---|---|---|
| **Dummy** | Sadece parametre doldurmak için verilir, hiç kullanılmaz | Kullanılmayan bir bağımlılığa `null` yerine verilen nesne |
| **Stub** | Çağrılara **önceden belirlenmiş sabit cevaplar** döner | "`findById(1)` çağrılırsa bu ürünü dön" |
| **Spy** | Gerçek nesneyi sarar, gerçek metotları çalıştırır ve çağrıları kaydeder | Gerçek bir listeyi sarıp `add` çağrılarını izlemek |
| **Mock** | Çağrıları kaydeder; test sonunda **beklenen çağrıların yapıldığı doğrulanır** | "`save` hiç çağrılmamış olmalı" |
| **Fake** | Basitleştirilmiş ama **gerçekten çalışan** bir implementasyon | `HashMap` ile çalışan bellek repository'si |

> REST konusundaki `ConcurrentHashMap`'li `ProductRepository` tam olarak bir **fake**'tir.

### Stub ve Stubbing

**Stub**, bir çağrıya önceden belirlenmiş cevap veren sahte nesnedir. Cevabı tanımlama işine **stubbing** denir.

```java
when(repository.findById(1L)).thenReturn(Optional.of(laptop));   // stubbing
```

- Stub, testin **girdisini hazırlar**: "Bu durumda dünya böyle görünüyor."
- Gerçek veritabanında kurması zor durumlar (bağlantı koptu, kayıt yok) tek satırda kurulur.

### Stub mu, Mock mu?

| | Stub | Mock |
|---|---|---|
| **Amacı** | Test edilen koda **veri sağlamak** | Test edilen kodun **ne yaptığını doğrulamak** |
| **Odak** | Durum (sonuç ne oldu?) | Etkileşim (ne çağrıldı?) |
| **Mockito karşılığı** | `when(...).thenReturn(...)` | `verify(...)` |

> **Mockito'da ayrım:** `@Mock` ile oluşturulan nesne, nasıl kullanıldığına göre ikisi de olabilir. `when` ile cevap tanımlanınca **stub** gibi, `verify` ile çağrı doğrulanınca **mock** gibi davranır. Aynı nesne bir testte ikisini birden yapabilir.

### Mockito Spy

```java
List<String> list = spy(new ArrayList<>());
list.add("a");                     // gerçek metot çalışır
verify(list).add("a");             // çağrı doğrulanabilir
when(list.size()).thenReturn(100); // istenirse tek tek metotlar stub'lanabilir
```

Mock'tan farkı: **Gerçek metotlar çalışır**, sadece stub'lananlar değişir. Nadiren gerekir; çoğu durumda mock yeterlidir.

### Strict Stubs

`MockitoExtension` varsayılan olarak **strict stubs** modunda çalışır: Stub'lanan bir metot testte hiç çağrılmazsa test **`UnnecessaryStubbingException`** ile kırılır.

Bu iki şeye işaret eder:
- Test gereksiz kurulum içeriyor, ya da
- Test edilen kod beklenen yoldan geçmiyor (daha ciddi).

## 7. Mockito

```java
@ExtendWith(MockitoExtension.class)
class ProductServiceTest {

    @Mock
    private ProductRepository repository;

    @InjectMocks
    private ProductService service;
}
```

| Anotasyon | Görevi |
|---|---|
| `@ExtendWith(MockitoExtension.class)` | JUnit'e Mockito anotasyonlarını işlemesini söyler |
| `@Mock` | Sahte nesne oluşturur. Varsayılan dönüşler: `null`, `false`, `0`, boş liste, `Optional.empty()` |
| `@InjectMocks` | Test edilen sınıfı oluşturur, mock'ları constructor'dan verir (`new ProductService(repository)`) |

> Mock bir **proxy**'dir: Gerçek nesnenin yerine geçip çağrıları karşılar.

### Stubbing ile Test

```java
@Test
void findById_whenProductExists_returnsResponse() {
    Product laptop = new Product("Laptop", new BigDecimal("25000"), new BigDecimal("20000"), 5);
    when(repository.findById(1L)).thenReturn(Optional.of(laptop));

    ProductResponse response = service.findById(1L);

    assertThat(response.name()).isEqualTo("Laptop");
    assertThat(response.inStock()).isTrue();
}

@Test
void findById_whenProductDoesNotExist_throwsResourceNotFoundException() {
    when(repository.findById(99L)).thenReturn(Optional.empty());

    assertThatThrownBy(() -> service.findById(99L))
            .isInstanceOf(ResourceNotFoundException.class)
            .hasMessageContaining("99");
}
```

Hata durumları da stub'lanabilir:

```java
when(repository.findById(any())).thenThrow(new DataAccessResourceFailureException("Bağlantı koptu"));
```

### Verify

Dönüş değerinden görünmeyen **yan etkileri** doğrular.

```java
@Test
void create_whenNameAlreadyExists_throwsAndDoesNotSave() {
    when(repository.existsByName("Laptop")).thenReturn(true);

    assertThatThrownBy(() -> service.create(request))
            .isInstanceOf(BusinessException.class);

    verify(repository, never()).save(any());
}
```

| Kullanım | Anlamı |
|---|---|
| `verify(mock).save(any())` | Tam olarak **1 kez** çağrıldı |
| `verify(mock, never()).save(any())` | **Hiç** çağrılmadı |
| `verify(mock, times(2)).save(any())` | Tam olarak 2 kez |
| `verify(mock, atLeastOnce()).save(any())` | En az 1 kez |
| `any()` | Hangi parametreyle çağrılırsa çağrılsın |

> Exception fırlatıldığını doğrulamak yetmez; exception'dan önce `save` çağrılmış olabilir.

### ArgumentCaptor

Metot bir şey dönmüyor, nesneyi oluşturup bağımlılığa veriyorsa, **gönderilen parametre yakalanır:**

```java
@ExtendWith(MockitoExtension.class)
class AuthServiceTest {

    @Mock private UserRepository userRepository;
    @Mock private PasswordEncoder passwordEncoder;
    @InjectMocks private AuthService authService;

    @Test
    void register_hashesPasswordAndAssignsUserRole() {
        RegisterRequest request = new RegisterRequest("ali", "gizli123");
        when(userRepository.existsByUsername("ali")).thenReturn(false);
        when(passwordEncoder.encode("gizli123")).thenReturn("HASHLENMIS_SIFRE");

        authService.register(request);

        ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
        verify(userRepository).save(captor.capture());

        User savedUser = captor.getValue();
        assertThat(savedUser.getPassword()).isEqualTo("HASHLENMIS_SIFRE");
        assertThat(savedUser.getPassword()).isNotEqualTo("gizli123");
        assertThat(savedUser.getRole()).isEqualTo(Role.USER);
    }
}
```

- Güvenlik kuralları (şifre hashleniyor mu, rol sunucuda mı veriliyor) bozulursa **hata alınmaz.** Tam olarak test edilmesi gereken türden davranışlar.
- `PasswordEncoder` mock'landı: Test edilen şey BCrypt'in çalışması değil, **service'in şifreyi encoder'dan geçirmesi.** Kütüphaneyi test etmek kütüphanenin işidir.

## 8. Neyi Test Etmeli, Neyi Etmemeli?

| ✅ Test Et | ❌ Test Etme |
|---|---|
| İş kuralları (service'teki her `if`) | Getter / setter |
| Sınır durumları (stok tam 0, fiyat tam eşit, boş liste) | Framework'ün kendisi (`JpaRepository.save` kaydediyor mu?) |
| Yan etkiler (kaydedildi mi, silindi mi?) | Kütüphanelerin doğruluğu (BCrypt çalışıyor mu?) |
| Hata durumları | |

### Davranışı Test Et, İmplementasyonu Değil

- Her `when` için bir `verify` yazmak hatadır. Stub'lanan değer sonuçta görünüyorsa metot zaten çağrılmıştır.
- `verify` sadece **yan etkiler** için kullanılır.
- Aşırı `verify` içeren testler, davranış aynı kalsa bile kod her düzenlendiğinde kırılır.

### Mock'ların Sınırı

Mock, ona ne söylenirse onu döner. **Gerçek sorgu hiç çalışmaz.**

`existsByName` stub'landığında, gerçek sorgunun doğru çalıştığı **test edilmemiş** olur. Sorguların doğruluğu `@DataJpaTest` ile test edilir.

---

## Örnek Sorular

**1. Test piramidi nedir?**
Tabanda çok sayıda hızlı birim testi, ortada daha az entegrasyon testi, tepede az sayıda yavaş uçtan uca test olması gerektiğini anlatan modeldir.

**2. Constructor injection test yazmayı neden kolaylaştırır?**
Test, Spring'e ihtiyaç duymadan `new ProductService(mockRepository)` ile sahte bağımlılıkları verebilir. Field injection'da reflection veya Spring context gerekir.

**3. Test double nedir? Türleri nelerdir?**
Testte gerçek bağımlılığın yerine geçen sahte nesnedir. Dummy (kullanılmaz), stub (sabit cevap döner), spy (gerçek nesneyi sarar), mock (çağrıları doğrular), fake (basitleştirilmiş çalışan implementasyon).

**4. Stub ile mock arasındaki fark nedir?**
Stub test edilen koda veri sağlar (durum odaklı), mock test edilen kodun ne çağırdığını doğrular (etkileşim odaklı). Mockito'da aynı `@Mock` nesnesi `when` ile stub, `verify` ile mock gibi kullanılır.

**5. Fake'e bir örnek verin.**
`HashMap` ile çalışan bellek içi repository. Gerçekten çalışır ama veritabanı yerine bellekte tutar.

**6. Mockito'da spy ile mock farkı nedir?**
Mock'un metotları varsayılan olarak hiçbir şey yapmaz. Spy gerçek nesneyi sarar ve gerçek metotları çalıştırır, sadece stub'lanan metotlar değişir.

**7. `UnnecessaryStubbingException` neden alınır?**
Strict stubs modunda, stub'lanan bir metot testte hiç çağrılmazsa. Gereksiz kurulumu veya kodun beklenen yoldan geçmediğini gösterir.

**8. `when(...).thenReturn(...)` ile `verify(...)` farkı nedir?**
`when` mock'un ne döneceğini önceden belirler. `verify` test sonunda bir metodun çağrılıp çağrılmadığını doğrular.

**9. `ArgumentCaptor` ne zaman kullanılır?**
Metot bir şey dönmeyip nesneyi bağımlılığa verdiğinde, gönderilen parametrenin içeriğini doğrulamak için.

**10. `BigDecimal` karşılaştırmasında `isEqualTo` neden başarısız olabilir?**
`equals` ölçeği de karşılaştırır (`250.00` ≠ `250`). `isEqualByComparingTo` kullanılmalıdır.

**11. `AuthServiceTest`'te `PasswordEncoder` neden mock'landı?**
Test edilen şey BCrypt'in çalışması değil, service'in şifreyi encoder'dan geçirmesidir. Birim testi sadece test edilen sınıfın sorumluluğunu doğrular.

**12. Repository mock'lanarak yazılan testler sorguların doğruluğunu garanti eder mi?**
Hayır. Mock gerçek sorguyu çalıştırmaz. Sorgular `@DataJpaTest` ile test edilmelidir.

**13. Her stub için `verify` yazmak neden kötüdür?**
Test davranış yerine implementasyonu kontrol eder ve kod her düzenlendiğinde kırılır. `verify` yan etkiler için kullanılmalıdır.
