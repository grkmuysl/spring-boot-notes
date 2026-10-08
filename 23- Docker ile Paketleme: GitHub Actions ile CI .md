# Docker ile Paketleme - Bölüm 2: GitHub Actions ile CI

## 1. Sorun: İnsanlar Unutur

- Testleri çalıştırmayı unutmak
- Sadece değiştirilen sınıfın testlerini çalıştırmak
- Ayrı ayrı doğru çalışan değişikliklerin birleşince bozulması

> Testlerin değeri **gerçekten çalıştırıldıklarında** ortaya çıkar. Bunu hafızaya bırakmak `println` gibidir: Bir gün unutulur.

| Kavram | Soru | Kapsam |
|---|---|---|
| **CI** (Continuous Integration) | "Bu değişiklik sağlam mı?" | Her push'ta otomatik derleme ve test |
| **CD** (Continuous Delivery / Deployment) | "Sağlam değişiklik sunucuya gitsin" | Otomatik dağıtım (platforma göre değişir, kapsam dışı) |

Sonuç birkaç dakikada: **Yeşil** (çalışıyor) veya **kırmızı** (bozuldu, hangi değişiklikle bozulduğu belli).

## 2. GitHub Actions Kavramları

| Kavram | Açıklama |
|---|---|
| **Workflow** | Otomasyonun tamamı. `.github/workflows/` içinde YAML dosyası. |
| **Event** | Tetikleyici: push, pull request, saat, elle tetikleme |
| **Job** | İş birimi. Her job **ayrı, temiz bir sanal makinede** çalışır. |
| **Runner** | Job'ın çalıştığı makine (`ubuntu-latest`). **Her çalışmada sıfırdan.** |
| **Step** | Job içinde tek adım: komut veya hazır action |
| **Action** | Yeniden kullanılabilir adım ("kodu indir", "Java kur") |

> Runner'ın sıfırdan oluşması: Senin bilgisayarında önceden kalmış dosya / ayar yüzünden geçen test CI'da kırılır. Docker "bende çalışıyor"u **çalışma ortamı** için, CI **test ortamı** için çözer.

## 3. İlk Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Kodu indir
        uses: actions/checkout@v4

      - name: Java 21 kur
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Derle ve test et
        run: ./mvnw -B verify
```

| Satır | Anlamı |
|---|---|
| `on` | `main`'e her push'ta ve `main`'e açılan her PR'da |
| `runs-on: ubuntu-latest` | Temiz Ubuntu makinesi. Public repoda ücretsiz, private'ta aylık ücretsiz kota. |
| `actions/checkout@v4` | Makine boş; repoyu indirir |
| `actions/setup-java@v4` | `temurin` + `21` = Dockerfile'daki `eclipse-temurin:21` ile **aynı Java** |
| `cache: maven` | Bağımlılıkları önbelleğe alır. Anahtar `pom.xml` içeriği → `pom.xml` değişmedikçe kullanılır. (Docker katman önbelleğiyle aynı fikir.) |
| `./mvnw -B verify` | Maven Wrapper ile derle + **tüm testler.** `-B`: okunaklı log. `verify`: derleme, test, paketleme ve entegrasyon testleri. |

- Sonuç reponun **Actions** sekmesinde, her adımın logu ayrı.

> Testlerin çalıştığı Java = Production'daki Java → "Testte geçti, production'da patladı" sürprizleri azalır.

### Testcontainers CI'da Çalışır mı?

**Evet.** GitHub Ubuntu runner'larında Docker kurulu ve çalışır durumda gelir. Ek ayar gerekmez.

### Gizli Bilgiler Testlerde

`jwt.secret` için CI'a gizli bilgi eklemek **gerekmez:** `application-test.yml`'da sabit, gizli olmayan test anahtarı var.

> Testleri production gizli bilgilerinden bağımsız tasarlamanın faydası.

### ⚠️ İlk Takılınan Yer: mvnw İzni

```
./mvnw: Permission denied
```

- Linux'ta dosya çalıştırılabilmesi için **çalıştırma izni** gerekir.
- Windows bu izni tanımaz; `mvnw` Git'e izinsiz kaydedilmiş olabilir.

```bash
git update-index --chmod=+x mvnw
git commit -m "mvnw dosyasını çalıştırılabilir yap"
```

> Dockerfile'daki `RUN ./mvnw ...` için de aynı sorun ve çözüm.

## 4. Testler Kırıldığında

```yaml
      - name: Test raporlarını kaydet
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: test-raporlari
          path: target/surefire-reports/
```

- `if: failure()`: Sadece önceki adım başarısızsa çalışır.
- Runner silinmeden raporlar kaydedilir, çalışma sayfasından indirilir.

### ⚠️ Kararsız Testler CI'ı Öldürür

```
Test her 10 çalışmada bir rastgele kırılıyor
→ "Yine o test, bir daha çalıştırayım"
→ Gerçek hata da aynı refleksle geçiştirilir
→ Kırmızı CI'ın anlamı kalmaz
```

> **Kural:** Kararsız test fark edildiğinde **hemen düzeltilir.**

| Kararsızlık Kaynağı | Çözüm |
|---|---|
| `Thread.sleep` | Awaitility |
| `Instant.now()` | Enjekte edilen `Clock` |
| Testler arası sızan önbellek | `spring.cache.type: none` |
| Zamanlayıcı testin ortasında çalışıyor | `@ConditionalOnProperty` ile kapat |

> Önceki konulardaki bu teknikler güvenilir bir CI'ın **ön koşullarıydı.**

## 5. Main Dalını Korumak: Branch Protection

CI kırık kodu **söyler** ama `main`'e girmesini **engellemez.**

**Repo ayarlarında `main` için kural:** Doğrudan push yok; değişiklik PR ile gelir; birleştirmek için **CI başarılı olmalı.**

```
1. Yeni dal aç:              git checkout -b siparis-iptali
2. Değişiklik yap, push'la
3. Pull request aç           → CI otomatik çalışır
4. CI kırmızı                → "Merge" butonu kilitli
5. Düzelt, tekrar push'la    → CI tekrar çalışır
6. CI yeşil                  → Birleştir
```

> **Sonuç:** `main` **her zaman** testlerden geçmiş kod içerir, her an production'a gönderilebilir.
> Doğru davranışı insanın dikkatine bırakmak yerine **sistemin zorunlu kılması** ("varsayılan olarak kapalı", güvenli varsayılanlar).

- Tek başına çalışırken bile iyi alışkanlık; GitHub profili incelendiğinde profesyonel izlenim.

## 6. İmajı Oluşturup Göndermek

**GitHub Container Registry (`ghcr.io`):** Kodla aynı yerde, ek hesap gerektirmez.

```yaml
  docker:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - name: Kodu indir
        uses: actions/checkout@v4

      - name: Registry'ye giriş yap
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Etiketleri belirle
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha
            type=raw,value=main

      - name: İmajı oluştur ve gönder
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

| Parça | Anlamı |
|---|---|
| `needs: test` | Testler **başarıyla** bitmeden başlamaz. Testler kırılırsa imaj oluşmaz. |
| `if: ...` | Sadece `main`'e push'ta. PR'larda imaj yayınlanmaz. |
| `permissions` | Sadece kodu oku + imaj yaz. **En az yetki ilkesi** CI'da. |
| `secrets.GITHUB_TOKEN` | GitHub'ın her çalışma için **otomatik** oluşturduğu, sadece o çalışmada geçerli anahtar. Ayrı şifre gerekmez. |
| `metadata-action` | `type=sha` → `sha-a1b2c3d`, `type=raw,value=main` → `main` etiketi |
| `build-push-action` | Bölüm 1'deki Dockerfile ile oluştur ve gönder |
| `cache-from/to: type=gha` | Runner sıfırdan başlar, Docker **katman önbelleği** kaybolur. Katmanlar GitHub Actions önbelleğinde saklanır; Dockerfile'daki katman sıralamasının faydası CI'da korunur. |

> ⚠️ Docker imaj adları **küçük harf** olmalı. GitHub kullanıcı adında büyük harf varsa imaj adı küçük harfle yazılmalı.

### Neden Commit Kimliğiyle Etiket?

- Production'da `shop:sha-a1b2c3d` çalışıyorsa **hangi koddan** derlendiği kesin.
- Hata çıkınca Git'te `a1b2c3d`'ye gidip kodu birebir görürsün.
- Geri dönmek gerekirse önceki commit'in imajı registry'de **değişmeden** duruyor.

> `info` endpoint'i + "migration'lar sadece ileri gider" + commit etiketli imajlar = Neyin nerede çalıştığı üzerinde **tam izlenebilirlik.**

```bash
docker run -p 8080:8080 -e SPRING_PROFILES_ACTIVE=prod ... ghcr.io/kullanici/shop:sha-a1b2c3d
```

- İmaj reponun ana sayfasında sağdaki **Packages** bölümünde görünür.

## 7. CI'da Gizli Bilgiler

- **Settings → Secrets and variables → Actions** → `${{ secrets.ADI }}`
- GitHub değerleri loglarda `***` ile maskeler.

> ⚠️ Gizli bilgi **workflow dosyasına yazılmaz** (o da kod, Git'e girer).
> ⚠️ Loglara yazdırılmaz; değer dönüştürülürse (Base64) maskeleme işe yaramayabilir.

### Fork'tan Gelen Pull Request'ler

Başkasının fork'undan gelen PR'larda workflow'lar gizli bilgilere **erişemez.**
> Aksi halde herkes workflow'u değiştiren bir PR açarak gizli bilgileri kendine gönderebilirdi.

### Action Sürümlerini Sabitlemek

- `actions/checkout@v4` gibi sürüm kullanılır (`latest` kullanmama ilkesi).
- Action'lar başkalarının kodu ve CI'da çalışıyor; sabitlenmezse yeni sürüm habersizce CI'ı bozabilir.
- Daha güvenlik odaklı: Sürüm etiketi yerine action'ın **commit kimliği.** Depo ele geçirilip etiket değiştirilse bile etkilenmez.

## 8. CI'ın Yapabileceği Diğer Şeyler

| Araç | Ne Yapar? |
|---|---|
| **JaCoCo** (code coverage) | Testlerin kodun ne kadarını çalıştırdığını ölçer. %100 hedefi anlamsız; ama kapsamın **birden düşmesi** test yazılmadan eklenmiş kod işaretidir. |
| **Dependabot** | Bağımlılıklardaki güvenlik açıklarını tarar, güncelleme PR'ı açar. PR'da CI çalışır; testler geçerse güncelleme güvenli. |

### Hız

- İyi CI **hızlıdır:** İdeal olarak **10 dakikanın altında.**
- 40 dakikalık CI'ı kimse beklemez, geri bildirim gecikir.
- En etkili araçlar: **Önbellekler** (Maven + Docker katmanları) ve **test context önbelleği** (farklı yapılandırılmış her test sınıfı uygulamayı yeniden başlatır).

## 9. Tam Workflow

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Derle ve test et
        run: ./mvnw -B verify

      - name: Test raporlarını kaydet
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: test-raporlari
          path: target/surefire-reports/

  docker:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha
            type=raw,value=main

      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## Sorular

**1. CI nedir, hangi sorunu çözer?**
Her değişikliğin otomatik olarak temiz bir ortamda derlenip test edilmesidir. Unutulan testler ve birleşince bozulan değişiklikler gibi insan kaynaklı hataları yakalar.

**2. CI ile CD farkı nedir?**
CI değişikliğin sağlam olup olmadığını kontrol eder. CD sağlam değişikliği otomatik olarak sunuculara götürür.

**3. Runner'ların her çalışmada sıfırdan oluşturulmasının faydası nedir?**
Testler geliştiricinin makinesindeki kalıntılardan etkilenmez; "bende çalışıyor" sorunu test ortamı için çözülür.

**4. `cache: maven` nasıl çalışır?**
Bağımlılıkları çalışmalar arasında saklar; anahtar `pom.xml` içeriğinden üretilir, `pom.xml` değişince yenilenir.

**5. Testcontainers testleri GitHub Actions'ta çalışır mı?**
Evet, Ubuntu runner'larında Docker kurulu ve çalışır durumda gelir.

**6. CI'da `./mvnw: Permission denied` neden alınır?**
Windows çalıştırma iznini tanımadığı için `mvnw` Git'e izinsiz kaydedilmiştir. `git update-index --chmod=+x mvnw` ile düzeltilir.

**7. Kararsız testler CI için neden tehlikelidir?**
Kırmızı CI görmezden gelinmeye başlanır ve gerçek hatalar da geçiştirilir. Kararsız testler hemen düzeltilmelidir.

**8. Branch protection neyi garanti eder?**
`main`'e doğrudan push'u engeller ve PR'ların birleştirilmesi için CI'ın başarılı olmasını zorunlu kılar; `main` her zaman testlerden geçmiş kod içerir.

**9. `needs: test` ne sağlar?**
İmaj job'ının sadece test job'ı başarılı olduktan sonra çalışmasını; testler kırılırsa imaj oluşmaz.

**10. Workflow'da `permissions` neden sınırlanır?**
En az yetki ilkesi; kötü niyetli bir action'ın yapabilecekleri sınırlanır.

**11. `GITHUB_TOKEN` nedir?**
GitHub'ın her workflow çalışması için otomatik oluşturduğu, sadece o çalışmada geçerli anahtar. Yetkileri `permissions` ile sınırlanır.

**12. İmajı commit kimliğiyle etiketlemenin avantajı nedir?**
Production'da hangi kodun çalıştığı kesin bilinir ve önceki sürüme değişmemiş imajla dönülebilir.

**13. `cache-from: type=gha` neden gereklidir?**
Runner sıfırdan başladığı için Docker katman önbelleği kaybolur; GitHub Actions önbelleği katmanları çalışmalar arasında saklar.

**14. Fork PR'larında workflow'lar neden gizli bilgilere erişemez?**
Aksi halde workflow'u değiştiren bir PR ile gizli bilgiler çalınabilirdi.

**15. Action sürümleri neden sabitlenir?**
Action'lar başkalarının kodudur; yeni sürüm CI'ı habersizce bozabilir. En güvenlisi commit kimliğiyle sabitlemektir.

**16. Dependabot ve CI birlikte nasıl çalışır?**
Dependabot açığı olan bağımlılık için güncelleme PR'ı açar, CI o PR'da otomatik çalışır; testler geçerse güncelleme güvenle birleştirilir.
