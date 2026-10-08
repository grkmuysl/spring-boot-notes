# Mikroservisler - Bölüm 1: Ne Zaman, Nasıl Bölünür ve Tutarlılık

## 1. Monolit

Şimdiye kadar yazılan uygulama bir **monolit:** Tek uygulama, tek veritabanı, tek jar, tek imaj.

| Monolitin Avantajları |
|---|
| Tek yerde geliştirilir, test edilir, deploy edilir |
| Metot çağrısı nanosaniyeler sürer, ağ hatası vermez |
| `@Transactional` birden fazla tabloyu bölünemez şekilde kaydeder |
| Hata ayıklarken tek log, tek uygulama |

### Büyüdükçe Ortaya Çıkan Sorunlar

| Sorun | Açıklama | Türü |
|---|---|---|
| **Takım koordinasyonu** | 80 kişi aynı kod tabanında: çakışmalar, birbirini engelleyen değişiklikler | **Organizasyonel** |
| **Deploy bağımlılığı** | Ödeme düzeltmesi için tüm uygulama deploy edilir, katalogun test edilmemiş kodu da gider | **Organizasyonel** |
| **Ölçekleme taneciği** | Sadece arama 50 kat yük alıyor ama tüm uygulama ölçeklenir | Teknik |
| **Hata yalıtımı** | Rapor modülündeki bellek sızıntısı `OOMKilled` → ödeme de durur | Teknik |


## 2. Mikroservis Nedir?

Uygulamayı her biri:
- **Tek bir iş yeteneğinden** (business capability) sorumlu,
- **Kendi verisinin sahibi,**
- **Bağımsız deploy edilebilen**

servislere bölmek. Servisler ağ üzerinden (HTTP veya mesajlaşma) konuşur.

> **En önemli kelime: Bağımsız deploy.**
> **Test:** "Bu servisi diğerlerine dokunmadan deploy edebiliyor muyum?"

### ⚠️ Dağıtık Monolit

Servislere bölünmüş ama **bağımsız deploy edilemeyen** sistem. Her deploy'da üç servis birlikte güncellenir.

> İki dünyanın **en kötü** yanları: Monolitin bağımlılığı + dağıtık sistemin tüm zorlukları. **En sık düşülen tuzak.**

### Conway Yasası

> Bir organizasyonun tasarladığı sistemler, o organizasyonun **iletişim yapısını** yansıtır.

- Mikroservisler bunu bilinçli kullanır: **Sistem sınırları = Ekip sınırları.**
- Ödeme ekibi ödeme servisinin sahibi: Geliştirir, test eder, deploy eder, gece uyarılarına bakar.
- Mikroservisler **teknik olduğu kadar organizasyonel bir araç.**

> Tek geliştirici / küçük ekipte en büyük fayda (ekip bağımsızlığı) **hiç ortaya çıkmaz**, bedeli ise **tamamen ödenir.**

## 3. Bedeli

Kursta öğrenilen zorlukların **hepsi her servis sınırında** ortaya çıkar.

| Bedel | Açıklama |
|---|---|
| **Ağ güvenilmez** | Metot çağrısı → HTTP çağrısı. Timeout, retry, circuit breaker, bulkhead, fallback, idempotency **kendi servislerin arasında** geçerli. |
| **Transaction kaybolur** | Her servisin kendi veritabanı; "ya hepsi ya hiçbiri" servis sınırında biter |
| **Tutarlılık gecikir** | Değişiklikler diğer servislere mesajlarla, gecikmeli ulaşır |
| **Hata ayıklama zorlaşır** | Beş servisin logları. `requestId` / `traceId` **zorunlu** hale gelir. |
| **İşletme yükü katlanır** | Her servis için ayrı CI, Dockerfile, veritabanı, Flyway, Actuator, alarmlar |
| **Test zorlaşır** | "Sipariş verilince stok düşüyor mu?" için iki servis birlikte çalışmalı |

> **"Dağıtık sistemlerin yanılgıları":** Ağın güvenilir, gecikmenin sıfır, bant genişliğinin sınırsız olduğunu varsaymak.

> **Önce monolitle başla.** Projenin başında iş alanı yeterince anlaşılmadığı için sınırlar büyük ihtimalle yanlış çizilir.
> - Monolitte yanlış sınırı düzeltmek = **Refactoring**
> - Mikroserviste = İki servisin verisini, API'lerini ve ekiplerini **yeniden düzenlemek**

## 4. Orta Yol: Modüler Monolit

**Tek uygulama** olarak deploy edilir, içinde **sınırları güçlü korunan modüller** vardır.

### Katmana Göre → İş Yeteneğine Göre

```
# KATMANA GÖRE (package by layer)
com.example.shop
├── controller   → ProductController, OrderController, UserController
├── service      → ProductService, OrderService, UserService
└── repository   → ProductRepository, OrderRepository, UserRepository
```

- Sipariş kodu üç pakete dağılmış.
- `OrderService` hiçbir şey engellemediği için `ProductRepository`'ye doğrudan erişebiliyor. **Sınır yok.**

```
# İŞ YETENEĞİNE GÖRE (package by feature)
com.example.shop
├── catalog
│   ├── CatalogController
│   ├── CatalogService        (public: diğer modüller kullanır)
│   └── internal/
│       └── ProductRepository (diğer modüller erişemez)
├── ordering
│   ├── OrderController
│   ├── OrderService
│   └── internal/ ...
└── payment
    └── ...
```

- Her modülün **içinde** katmanlı mimari aynen duruyor.

**Kurallar:**
- Bir modül başka modülün sadece **açıkça yayınladığı** sınıflarını kullanır.
- Başka modülün iç sınıflarına, repository'lerine ve **tablolarına** erişemez.
- Modüller arası bildirimler tercihen **Spring Events** ile.

> **Değeri:** Modüller zaten mikroservis sınırlarına sahip. Servise çıkarmak gerekirse: Metot çağrıları → HTTP, Spring Events → Kafka.

### Spring Modulith

```java
@Test
void verifyModuleBoundaries() {
    ApplicationModules.of(ShopApplication.class).verify();
}
```

- `ordering`'den `catalog.internal`'e erişilirse **test kırılır, CI kırmızı.**
- Mimari kuralları insan dikkatine değil **otomatik kontrole** bırakmak (branch protection'ın mimari karşılığı).
- Modüller arası olayları veritabanında kalıcı saklayan mekanizma = **Hazır transactional outbox.**

> **Pek çok proje için doğru cevap mikroservisler değil, iyi bir modüler monolittir.**

## 5. Sınırları Çizmek

### ❌ Yanlış Yollar

| Yanlış Yol | Sorun |
|---|---|
| **Teknik katmanlara göre** ("veritabanı servisi", "doğrulama servisi") | Her iş özelliği hepsine dokunur → Birlikte değişirler → Dağıtık monolit |
| **Çok küçük parçalar** (her entity için servis = nano-servis) | Ürün detay sayfası için 4 servise istek → **Geveze (chatty)** servisler, birikmiş gecikme ve hata |

### ✅ Doğru Yol: İş Yetenekleri ve Bounded Context

Sınırlar **iş yeteneklerini** takip eder: Katalog, sipariş, ödeme, envanter, kargo.

**Bounded context** (Domain-Driven Design): Aynı kelime işin farklı alanlarında **farklı anlamlara** gelir.

| Bağlam | "Ürün" Ne Demek? |
|---|---|
| **Katalog** | Ad, açıklama, resimler, kategori |
| **Envanter** | Stok miktarı, depo, raf |
| **Fiyatlandırma** | Liste fiyatı, kampanya, maliyet |
| **Sipariş** | Satın alındığı andaki ad ve **o anki fiyat** |

- Monolitte tek `Product` her şeyi taşımaya çalışır, devasa bir sınıfa dönüşür.
- Her bağlam **kendi ürün modeline** sahip olmalı ve birbirinin alanlarından habersiz olmalı.

> **`OrderItem.unitPrice`:** Fiyatı üründen okumak yerine kopyaladık. Sipariş bağlamında önemli olan **satın alındığı andaki** fiyat. Katalogda fiyat yarın değişse de dünkü siparişin tutarı değişmemeli. Farkında olmadan bir **bağlam sınırı** çizmişiz.

### Kötü Sınır Belirtileri

- İki servis neredeyse her zaman **birlikte değişiyor** / deploy ediliyor
- Bir işlem için servisler arasında **çok sayıda** çağrı
- İki servis **aynı tabloları** okuyor / yazıyor

## 6. Her Servisin Kendi Veritabanı

> **Kural:** Her servis kendi verisinin **tek sahibidir.** Başka servis tablolarına erişemez; veriye sadece **API** veya **olaylar** üzerinden ulaşılır.

### ⚠️ Ortak Veritabanı Neden Kötü?

```
Sipariş ve envanter servisleri aynı products tablosunu okuyor
→ Envanter ekibi migration ile stock → quantity_on_hand
→ Sipariş servisi BOZULUR
→ Envanter, siparişi güncellemeden deploy edilemez
→ Bağımsız deploy yok
```

> Ortak veritabanı servisleri kodla değil **şemayla** bağlar; bu bağ çok daha görünmezdir.

**Kaybedilenler:**
- **Servisler arası `JOIN` yok** ("siparişleri müşteri adlarıyla listele" tek SQL değil)
- **Servisler arası transaction yok**

### Veri Tekrarı ve Nihai Tutarlılık

**Sorun:** Her sipariş için müşteri servisine istek = **Dağıtık N+1.** 100 sipariş = 100 ağ isteği; müşteri servisi çökerse liste de çöker.

**Çözüm:** İhtiyaç duyulan verinin **yerel kopyası.**
- Müşteri servisi `CustomerUpdated` olayı yayınlar.
- Sipariş servisi dinler, **sadece gereken alanları** (ad, e-posta) kendi tablosunda saklar.
- Kafka **log compaction** bu senaryo için ideal.

**Bedeli: Nihai tutarlılık (eventual consistency).**
- Ad değişince sipariş servisindeki kopya **birkaç saniye** eski kalabilir.
- Sistem bir an tutarsız olabilir, yeterli zamanda tüm kopyalar aynı değere ulaşır.

> Caching'deki "hız karşılığında tazelik" takası, bu sefer **servis bağımsızlığı** karşılığında.

## 7. Servisler Arası Tutarlılık: Saga

**Sorun:** Sipariş kaydet + ödeme al + stok ayır. Monolitte tek `@Transactional`. Şimdi üç servis, üç veritabanı. Ödeme alındı, envanter "stok yok" dedi → Ödemeyi kim geri alacak?

**Saga:** Dağıtık işlemi, her biri **kendi servisinde yerel transaction** olan adımlara bölmek. Bir adım başarısız olursa öncekiler **telafi edici işlemlerle** (compensating transactions) geri çevrilir.

```
Mutlu yol:
1. Sipariş servisi:  Siparişi PENDING kaydet
2. Ödeme servisi:    Ödemeyi al
3. Envanter servisi: Stoğu ayır
4. Sipariş servisi:  CONFIRMED yap

Envanter başarısız:
3. Envanter servisi: "Stok yok"      ✗
   → Ödeme servisi:  Ödemeyi iade et         (telafi)
   → Sipariş servisi: CANCELLED yap          (telafi)
```

> Asenkron işlerdeki "PENDING kaydet, sonuca göre ilerlet" **durum makinesinin** birden fazla servise yayılmış hali.

### ⚠️ Telafi ≠ Rollback

| | Rollback | Telafi |
|---|---|---|
| Ne yapar? | Değişikliği **hiç olmamış gibi** yapar | Olanı **yeni bir işlemle dengeler** |
| Örnek | — | Ödeme alındı + iade edildi = Hesapta **iki ayrı kayıt** |

- Bazı adımlar **hiç telafi edilemez:** Gönderilmiş e-posta → Telafisi ancak "iptal edildi, özür dileriz" e-postası.

> **Telafisi zor / imkânsız adımlar** (e-posta, kargoya verme) mümkün olduğunca **sona** bırakılır.

### Koreografi vs Orkestrasyon

| | Koreografi | Orkestrasyon |
|---|---|---|
| **Benzetme** | Dans topluluğu: Herkes müziği dinler, adımını bilir | Şef: Herkese sırasını söyler |
| **Koordinasyon** | Dağıtık, her servis olaylara tepki verir | Merkezi orkestratör |
| **Bağımlılık** | Düşük, servisler birbirini tanımaz | Orkestratör tüm adımları bilir |
| **Akışı anlamak** | Zor, akış hiçbir yerde tek parça yazılı değil | Kolay, akış tek yerde |
| **Uygun** | Az adımlı, basit akışlar | Çok adımlı, karmaşık, dallanan akışlar |

```
Koreografi:
Sipariş ─OrderCreated─► Ödeme ─PaymentCompleted─► Envanter ─StockReserved─► Sipariş

Orkestrasyon:
Orkestratör ─"ödemeyi al"─► Ödeme
            ◄──cevap────────
            ─"stoğu ayır"──► Envanter
            ◄──cevap────────
```

> Koreografinin sinsi sorunu: Adımlar arttıkça "sipariş verilince tam olarak ne oluyor?" sorusunun cevabı beş servisin dinleyicilerine dağılır.

**Her iki yolda da gerekenler:**
- Olaylar güvenle yayınlanmalı → **Outbox**
- Her adım tekrar gelen mesajlara karşı → **Idempotent**
- İşlenemeyen adımlar → **DLQ**

### Kullanıcıya Nasıl Görünür?

- Monolitte cevap gelince sipariş **kesindi.**
- Saga'da cevap gelince sipariş hâlâ **`PENDING`**; ödeme ve stok birkaç saniye sonra.
- Arayüz: **"Siparişiniz alındı, onaylanıyor."** (Onay e-postasının dakikalar sonra gelmesinin bir sebebi.)

> Nihai tutarlılık teknik detay değil, **iş tarafının kabul etmesi gereken** bir davranış. Ürün sahibiyle birlikte karar verilir.

## 8. Servisler Nasıl Konuşmalı?

| | Senkron (HTTP) | Asenkron (Mesajlaşma) |
|---|---|---|
| **Ne zaman?** | **Cevaba hemen ihtiyaç** var | Bir şeyin **olduğu bildiriliyor** |
| **Örnek** | "Sepetteki ürünlerin güncel fiyatı ne?" | "Bir sipariş oluşturuldu" |

### ⚠️ Senkron Zincirler

```
Kullanıcı → A → B → C → D

Her servis %99,9 ayakta:
0,999 × 0,999 × 0,999 × 0,999 ≈ 0,996
```

| Sorun | Açıklama |
|---|---|
| **Erişilebilirlik çarpılır** | 4 servislik zincir tek servisten **~4 kat sık** başarısız olur |
| **Gecikmeler toplanır** | Her biri 50 ms → Kullanıcı 200 ms bekler |
| **Zincirleme çöküş** | D yavaşlar → C'nin thread'leri D'yi, B'ninkiler C'yi, A'nınkiler B'yi bekler |

> **Senkron zincirler kısa tutulur.** Başka servise ihtiyaç duyan servis ya **yerel kopya** tutar ya işi **asenkron olaylara** dönüştürür.

## 9. Monolitten Mikroservise Geçiş: Strangler Fig

> ⚠️ **En büyük hata:** Her şeyi baştan yeniden yazmak. Aylar sürer, iş tarafı yeni özellik bekler, eski sistemin biriktirdiği iş kuralları ve özel durumlar unutulur.

**Strangler fig (boğucu incir):** Bir ağacın etrafında büyüyüp zamanla yerini alan incir türü.

```
1. Monolitin önüne yönlendirme katmanı (API gateway) konur → Tüm istekler monolite
2. Tek yetenek (bildirimler) yeni servise taşınır → Gateway bildirim isteklerini yeni servise yönlendirir
3. Yeni servis sağlam çalışınca monolitteki eski kod silinir
4. Sonraki yetenek...
```

- Monolit adım adım küçülür, **her adımda sistem çalışır.**
- İlk taşınacak: **En bağımsız** yetenek (en az ilişkili, verisine en net sahip olan).
- Modüler monolitteki modüller bu seçimi kolaylaştırır.

## 10. Karar Listesi

| Soru | Mikroservis Lehine Cevap |
|---|---|
| Kaç ekip, kaç geliştirici? | Birden fazla bağımsız ekip aynı kod tabanında birbirini yavaşlatıyor |
| Deploy'lar birbirini bekliyor mu? | Evet, bir ekibin değişikliği diğerlerini sürekli bekletiyor |
| Ölçekleme ihtiyaçları çok mu farklı? | Bir parça diğerlerinden kat kat fazla yük alıyor |
| İş alanı ve sınırlar ne kadar net? | Yetenekler net, sınırlar zamanla oturmuş |
| Dağıtık sistemi işletecek altyapı ve bilgi var mı? | CI/CD, izleme, dağıtık izleme, mesajlaşma tecrübesi mevcut |

> Çoğuna "hayır" → Doğru seçim büyük ihtimalle **iyi yapılandırılmış modüler monolit.**
> Bu bir başarısızlık değil; **probleme uygun araç seçmek mühendisliğin ta kendisi.**

---

## Sorular

**1. Monolit kötü bir mimari midir?**
Hayır. Tek yerde geliştirme ve deploy, hızlı metot çağrıları, gerçek transaction'lar ve kolay hata ayıklama gibi büyük avantajları vardır. Sorunlar ölçek ve ekip büyüdükçe ortaya çıkar.

**2. Monolitin büyüdükçe yaşadığı sorunlar nelerdir?**
Takım koordinasyonu, deploy bağımlılığı (organizasyonel), ölçekleme taneciği ve hata yalıtımı (teknik).

**3. Mikroservisin en önemli özelliği nedir?**
Bağımsız deploy edilebilmesi; diğer servislere dokunmadan kendi zamanında yayınlanabilmesi.

**4. Dağıtık monolit nedir?**
Servislere bölünmüş ama birlikte deploy edilmek zorunda olan sistem; monolitin bağımlılığını dağıtık sistemin zorluklarıyla birleştirir.

**5. Conway yasası nedir?**
Sistemlerin onları tasarlayan organizasyonun iletişim yapısını yansıtmasıdır. Mikroservisler sınırları ekiplerle hizalar.

**6. Mikroservislerin bedelleri nelerdir?**
Güvenilmez ağ, kaybolan transaction'lar, gecikmeli tutarlılık, zorlaşan hata ayıklama ve test, katlanan işletme yükü.

**7. Neden "önce monolit" önerilir?**
Projenin başında iş alanı yeterince anlaşılmaz ve sınırlar yanlış çizilir; monolitte düzeltmek bir refactoring, mikroserviste ise büyük bir yeniden düzenlemedir.

**8. Modüler monolit nedir?**
Tek uygulama olarak deploy edilen, içinde iş yeteneklerine göre ayrılmış ve sınırları korunan modüller içeren yapı. İleride modülleri servise çıkarmayı kolaylaştırır.

**9. Package by layer ile package by feature farkı nedir?**
Katmana göre düzenlemede bir özelliğin kodu dağılır ve modül sınırı yoktur. Özelliğe göre düzenlemede her yetenek kendi paketindedir ve iç sınıfları gizlenebilir.

**10. Spring Modulith ne sağlar?**
Modül sınırlarını testle doğrular ve modüller arası olayları kalıcı saklayan hazır bir outbox mekanizması sunar.

**11. Servis sınırları neden teknik katmanlara göre çizilmemelidir?**
Her iş özelliği tüm katman servislerine dokunur; servisler birlikte değişir ve dağıtık monolit oluşur.

**12. Bounded context nedir?**
Aynı kavramın farklı iş alanlarında farklı anlamlara geldiği ve her alanın kendi modeline sahip olması gerektiği fikridir.

**13. Servisler neden ortak veritabanı paylaşmamalıdır?**
Şema üzerinden birbirine bağlanırlar; bir servisin migration'ı diğerini bozar ve bağımsız deploy kaybolur.

**14. Nihai tutarlılık nedir?**
Verinin kopyalarının bir süre farklı olabildiği ama zamanla aynı değere ulaştığı tutarlılık modeli.

**15. Saga nedir?**
Dağıtık işlemi yerel transaction'lardan oluşan adımlara bölüp başarısızlıkta önceki adımları telafi edici işlemlerle geri çeviren desen.

**16. Telafi edici işlem rollback'ten nasıl farklıdır?**
Rollback değişikliği yok eder; telafi olanı yeni bir işlemle dengeler. Bazı adımlar hiç telafi edilemez, bu yüzden sona bırakılır.

**17. Koreografi ve orkestrasyon farkı nedir?**
Koreografide servisler olaylara tepki verir, merkez yoktur ama akış görünmez. Orkestrasyonda merkezi orkestratör adımları yönetir, akış nettir.

**18. Senkron servis zincirlerinin riski nedir?**
Erişilebilirlik çarpılarak düşer, gecikmeler toplanır ve zincirleme çöküş riski artar.

**19. Strangler fig deseni nedir?**
Monoliti baştan yazmak yerine yetenekleri bir yönlendirme katmanı arkasında tek tek yeni servislere taşıyıp eski kodu adım adım silme yaklaşımı.
