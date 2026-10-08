# Docker ile Paketleme - Bölüm 1: Dockerfile ve Konteynerde Spring Boot

## 1. Sorun: "Bende Çalışıyor"

Uygulamayı çalıştırmak için gerekenler:
- Doğru Java sürümü
- Doğru ortam değişkenleri
- Doğru çalıştırma komutu
- Doğru JVM ayarları

Her biri sunucuyu kuran kişi için bir hata fırsatı.

> **Docker'ın cevabı:** Uygulamayı **çalışması için gereken her şeyle** tek pakette (imajda) dağıtmak. İmaj her ortamda **birebir aynı**, değişen sadece dışarıdan verilen ayarlar.

- İmaj = sınıf, konteyner = nesne.
- Şimdiye kadar başkalarının "sınıflarını" (`postgres:16-alpine`) kullandık; şimdi kendi sınıfımızı yazıyoruz.
- **Dockerfile:** İmajın tarifi.

## 2. İlk Dockerfile ve Sorunları

```dockerfile
FROM eclipse-temurin:21-jdk
COPY target/shop-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

| Talimat | Görevi |
|---|---|
| `FROM` | Neyin üzerine kurulacak (Java 21 içeren hazır imaj) |
| `COPY` | Bilgisayardaki dosyayı imaja kopyala |
| `ENTRYPOINT` | Konteyner başlayınca çalışacak komut |

| Sorun | Açıklama |
|---|---|
| **Jar dışarıda derleniyor** | Senin bilgisayarındaki Java / Maven ile derlenmiş. "Bende çalışıyor" derleme adımında geri geldi. |
| **İmaj gereksiz büyük** | `jdk` derleyiciyi de içerir. Çalışan uygulama sadece **JRE** ister. |
| **Root olarak çalışıyor** | En yetkili kullanıcı |
| **Her değişiklikte her şey yeniden** | Tek satır kod değişince 60 MB'lık jar yeniden kopyalanır |

## 3. Katmanlar ve Önbellek

- Her talimat bir **katman** oluşturur. İmaj = üst üste katmanlar.
- Talimat ve girdileri değişmediyse katman **önbellekten** kullanılır.
- Bir katman değişince **ondan sonraki tüm katmanlar** yeniden oluşturulur.

> **Kural:** Talimatları **en az değişenden en çok değişene** sırala.

| Ne | Değişme Sıklığı |
|---|---|
| Java sürümü | Yılda bir |
| Bağımlılıklar (`pom.xml`) | Ayda bir |
| Kendi kodun | Her gün |

> Tek jar = Birkaç yüz KB senin kodun + 60 MB bağımlılık. Kod değişince 60 MB'ın tamamı yeniden gönderilir.

## 4. Multi-Stage Build

```dockerfile
# ---------- 1. aşama: derle ----------
FROM eclipse-temurin:21-jdk AS build
WORKDIR /workspace

COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
RUN ./mvnw dependency:go-offline -B

COPY src src
RUN ./mvnw package -DskipTests -B

# ---------- 2. aşama: çalıştır ----------
FROM eclipse-temurin:21-jre
WORKDIR /app

RUN useradd --system --uid 1001 spring
USER spring

COPY --from=build /workspace/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

| Aşama | Görevi |
|---|---|
| **1. aşama** (`AS build`) | JDK ile uygulamayı **imajın içinde** derler. Maven Wrapper (`mvnw`) → herkes aynı ortamda derler. |
| **2. aşama** | Sadece JRE. Birinci aşamadan **sadece jar'ı** alır (`COPY --from=build`). |

> **Benzetme:** Birinci aşama bir iskele. Binayı kurmak için gerekli, bina bitince sökülür. Derleyici, Maven, kaynak kod son imaja **girmez.**

### Bağımlılık Katmanı

```dockerfile
COPY pom.xml .
RUN ./mvnw dependency:go-offline -B   # ayrı katman
COPY src src                          # kod sonra
RUN ./mvnw package -DskipTests -B
```

- Kod değişince `COPY src` katmanı değişir, ama **öncesi** (bağımlılık indirme) önbellekten gelir.
- Bağımlılıklar **sadece `pom.xml` değişince** yeniden indirilir.
- Dakikalar → saniyeler.

### Testler Neden Atlandı? (`-DskipTests`)

- Testler **CI hattında**, imaj oluşturulmadan **önce** çalışır.
- Testcontainers testleri Docker ister; imaj derlenirken içeride Docker çalıştırmak ayrı bir karmaşıklık.

> **İş bölümü:** CI testleri çalıştırır → testler geçerse imaj oluşturulur.

## 5. Root Olmayan Kullanıcı

```dockerfile
RUN useradd --system --uid 1001 spring
USER spring
```

- Uygulamadaki bir açıkla konteynerde komut çalıştırılırsa, saldırganın yetkisi **uygulamanın yetkisiyle** sınırlı kalır.
- Root ise konteynerde her şey yapılabilir; izolasyon zaafiyetiyle ana makineye bile ulaşılabilir.

> **En az yetki ilkesi** (principle of least privilege): Her bileşen işini yapmak için gereken **en az** yetkiyle çalışır.
> Security'de en az rol, Actuator'da en az endpoint, şimdi işletim sisteminde en az yetki.

## 6. Katmanlı Jar

| Katman | İçerik | Değişme Sıklığı |
|---|---|---|
| `dependencies` | Spring, Hibernate, Jackson (sabit sürüm) | Nadiren |
| `spring-boot-loader` | Jar'ı başlatan kod | Çok nadiren |
| `snapshot-dependencies` | SNAPSHOT bağımlılıklar | Ara sıra |
| `application` | **Senin kodun** | Her derleme |

```dockerfile
# ---------- 1. aşama: derle ----------
FROM eclipse-temurin:21-jdk AS build
WORKDIR /workspace
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
RUN ./mvnw dependency:go-offline -B
COPY src src
RUN ./mvnw package -DskipTests -B

# ---------- 2. aşama: jar'ı katmanlara ayır ----------
FROM eclipse-temurin:21-jre AS extract
WORKDIR /builder
COPY --from=build /workspace/target/*.jar app.jar
RUN java -Djarmode=tools -jar app.jar extract --layers --launcher --destination extracted

# ---------- 3. aşama: çalıştır ----------
FROM eclipse-temurin:21-jre
WORKDIR /app
RUN useradd --system --uid 1001 spring
USER spring

COPY --from=extract /builder/extracted/dependencies/ ./
COPY --from=extract /builder/extracted/spring-boot-loader/ ./
COPY --from=extract /builder/extracted/snapshot-dependencies/ ./
COPY --from=extract /builder/extracted/application/ ./

EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

- Dört `COPY` = En az değişenden en çok değişene.
- Deploy'da sunucu büyük `dependencies` katmanını önbellekte bulur, **sadece birkaç yüz KB** indirir.

> `extract` komutu **Spring Boot 3.3+** içindir. Eski eğitimlerdeki `-Djarmode=layertools` eski sürümlerin karşılığı. Sürüm belgelerine bakılmalı.

### Alternatif: Buildpacks

```bash
./mvnw spring-boot:build-image
```

- **Dockerfile yazmadan** imaj (Cloud Native Buildpacks).
- Katmanlı jar, root olmayan kullanıcı, JRE seçimi, konteyner JVM ayarları **otomatik.**
- Dezavantaj: Daha az kontrol.

> Dockerfile'ı elle yazmayı öğrenmek, Buildpacks'in arka planda ne yaptığını anlamayı sağlar.

## 7. Gizli Bilgiler İmaja Girmemeli

```
# .dockerignore
target/
.git/
.idea/
*.iml
.env
.env.*
```

- `.dockerignore` = `.gitignore`'un Docker versiyonu. `COPY` sırasında **gönderilmeyecek** dosyalar.
- `target/`, `.git/` → derleme hızlanır.

> ⚠️ **`.gitignore` Docker'ı etkilemez.** `COPY . .` yapılırsa `.env` **imaja girer.** İmajı indirebilen herkes JWT secret'ı ve veritabanı şifresini okur.

```dockerfile
ENV JWT_SECRET=gizli-anahtar   # YAPMA
```

> ⚠️ **Katmanlar kalıcı ve incelenebilir** (`docker history`). Sonraki katmanda silinse bile önceki katmanda durur. (Git geçmişinde kalan gizli bilginin Docker versiyonu.)

> **Kural:** Gizli bilgiler imaja değil, **çalışma zamanında ortam değişkeni** olarak verilir.

## 8. İmajı Oluşturmak ve Çalıştırmak

```bash
docker build -t shop:1.0.0 .
```

- `-t`: Ad ve etiket. **`latest` yerine anlamlı sürüm.** (CI'da commit kimliği.)

```bash
docker run -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e DATABASE_URL=jdbc:postgresql://db.example.com:5432/shop \
  -e JWT_SECRET=... \
  shop:1.0.0
```

- `SPRING_PROFILES_ACTIVE` → **Relaxed binding** ile `spring.profiles.active`
- `DATABASE_URL` → Production profilindeki `${DATABASE_URL}` yer tutucusu

> Yapılandırma konusundaki yapı tam bu an için: **İmaj aynı, ortam değişkenleri farklı.**

## 9. JVM'i Konteynerde Doğru Çalıştırmak

### Bellek

- Konteynerler bellek **sınırıyla** çalışır (örn. 1 GB). Aşılırsa işletim sistemi konteyneri **öldürür.**
- Java 10+ JVM konteyner sınırını anlar.
- ⚠️ **Varsayılan:** Heap için kullanılabilir belleğin sadece **%25'i.** 1 GB konteynerde 256 MB heap, 750 MB boşta.

```dockerfile
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0"
```

- `JAVA_TOOL_OPTIONS`: JVM başlarken otomatik okunur. `ENTRYPOINT`'e dokunmadan, hatta `docker run -e` ile ayarlanabilir.

> **Neden %100 değil?** JVM belleği **sadece heap değil:** Metaspace (sınıf bilgileri), her thread'in yığını (Tomcat'in 200 thread'i), ağ tamponları, JVM iç yapıları. Heap = konteyner boyutu → toplam aşılır → konteyner öldürülür.

| Belirti | Anlamı | Çözüm |
|---|---|---|
| Logda `OutOfMemoryError: Java heap space` | **Heap** doldu. JVM fırlattı, uygulama çalışıyor olabilir. | Heap'i büyüt veya bellek sızıntısı ara |
| Konteyner aniden durdu, logda hata yok, durum **`OOMKilled`** | **Konteynerin toplam belleği** aşıldı, işletim sistemi öldürdü | `MaxRAMPercentage`'i düşür veya konteyner sınırını artır |

> `OOMKilled` çok daha kafa karıştırıcı: Uygulama hiçbir şey söylemeden ölür. `jvm.memory.used` metriği ayırt etmeyi sağlar.

### Graceful Shutdown

**Docker konteyneri durdururken:**
1. **SIGTERM** gönderir ("lütfen kapan")
2. Belirli süre bekler (varsayılan 10 sn)
3. Hâlâ kapanmadıysa **SIGKILL** ile zorla öldürür

**Graceful shutdown:** Yeni istekleri reddet → İşlenmekte olanların bitmesini bekle → Kapan.

- Spring Boot **3.4+ varsayılan.** Eski sürümlerde: `server.shutdown: graceful`

```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s
```

> Bu süre, platformun SIGKILL göndermeden önce beklediği süreden **kısa** olmalı.

### ⚠️ Exec Form vs Shell Form

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]    # ✅ exec form
ENTRYPOINT java -jar app.jar              # ❌ shell form
```

| Form | Ne Olur? |
|---|---|
| **Exec form** | Java doğrudan başlar, **SIGTERM'i kendisi alır** |
| **Shell form** | Java bir **kabuğun içinde** çalışır. SIGTERM kabuğa gider, **Java'ya iletilmez.** Spring kapanması gerektiğini öğrenmez, SIGKILL ile öldürülür, istekler yarıda kalır. |

> **Her zaman exec form.**

> Graceful shutdown `@Async` kuyruğu kaybını **hafifletir ama çözmez**; SIGKILL her zaman bir ihtimal. Kaybolmaması gereken işler için **outbox.**

## 10. Tüm Sistem: Docker Compose

```yaml
services:
  app:
    build: .
    container_name: shop-app
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: prod
      DATABASE_URL: jdbc:postgresql://postgres:5432/shop
      DATABASE_USERNAME: shop
      DATABASE_PASSWORD: shop
      SPRING_DATA_REDIS_HOST: redis
      JWT_SECRET: ${JWT_SECRET}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  postgres:
    # ... önceki tanım

  redis:
    # ... önceki tanım
```

| Ayar | Anlamı |
|---|---|
| `build: .` | Hazır imaj yerine bu klasördeki Dockerfile'dan oluştur |
| `depends_on` + `service_healthy` | PostgreSQL ve Redis **sağlıklı** olana kadar uygulama başlamaz |
| `${JWT_SECRET}` | Değer kabuktaki ortam değişkeninden veya `.env`'den gelir; dosyaya yazılmaz |

> `depends_on` olmasaydı uygulama veritabanından önce açılır, Flyway bağlanamaz, uygulama hata verip kapanırdı. (Docker konusunda `pg_isready` healthcheck'i tam bu an için tanımlamıştık.)

### ⚠️ En Sık Hata: localhost

```
jdbc:postgresql://localhost:5432/shop   ❌
jdbc:postgresql://postgres:5432/shop    ✅
```

| Durum | `localhost` Nedir? |
|---|---|
| Uygulama **senin bilgisayarında** (dev profili) | Senin bilgisayarın → port eşlemesiyle konteynere ulaşır ✅ |
| Uygulama **kendi konteynerinde** | **O konteynerin kendisi** → Kendi içinde PostgreSQL arar, bulamaz ❌ |

- Compose servisleri **ortak bir ağa** koyar, her servise **adıyla** ulaşılır: `postgres`, `redis`.
- Konteynerize uygulamanın "veritabanına bağlanamıyorum" hatasının **açık ara en yaygın sebebi.**

> PostgreSQL'in `ports` eşlemesi uygulama için gerekmez (ağ içinden ulaşıyor); sadece senin bilgisayarından erişim için. Dışarıya sadece gerekeni aç.

```bash
docker compose up --build
```

> Projeyi klonlayan biri **hiçbir şey kurmadan** tek komutla çalışan sisteme sahip olur. "Bende çalışıyor" sorununun sonu.

---

## Sorular

**1. Dockerfile'da talimatlar neden en az değişenden en çok değişene sıralanır?**
Katmanlar önbelleğe alınır; bir katman değişince sonrakiler yeniden oluşturulur. Sık değişen kod sona konursa nadiren değişen katmanlar önbellekten kullanılır.

**2. Multi-stage build hangi sorunları çözer?**
Derleme imajın içinde herkes için aynı ortamda yapılır ve son imaja sadece JRE ile jar girer; imaj küçülür.

**3. `pom.xml` neden kaynak koddan önce ayrı kopyalanır?**
Bağımlılık indirme ayrı katman olur; sadece kod değişince bağımlılıklar yeniden indirilmez.

**4. İmaj derlenirken testler neden atlanır?**
Testler CI hattında imajdan önce çalışır. Ayrıca Testcontainers testleri imaj derlemesi içinde Docker gerektirir.

**5. Konteyner neden root olmayan kullanıcıyla çalıştırılır?**
En az yetki ilkesi: Açık kullanılırsa saldırganın yetkisi uygulamanın yetkisiyle sınırlı kalır.

**6. Katmanlı jar ne sağlar?**
Bağımlılıklar ve kod ayrı katmanlara bölünür; deploy'da sadece değişen küçük uygulama katmanı indirilir.

**7. Buildpacks nedir?**
Dockerfile yazmadan, iyi uygulamaları otomatik içeren imaj oluşturma yöntemi (`spring-boot:build-image`). Dezavantajı daha az kontroldür.

**8. `.dockerignore` neden önemlidir?**
`.gitignore` Docker'ı etkilemez; `.env` gibi gizli dosyalar geniş `COPY` ile imaja girebilir.

**9. Dockerfile'a gizli bilgi yazıp sonraki katmanda silmek neden yetmez?**
Katmanlar kalıcıdır ve `docker history` ile incelenebilir; değer önceki katmanda durur.

**10. 1 GB konteynerde bellek boşken neden `OutOfMemoryError` alınabilir?**
JVM varsayılan olarak heap için belleğin %25'ini ayırır. `-XX:MaxRAMPercentage` ile artırılır, heap dışı alanlar için pay bırakılır.

**11. `OutOfMemoryError` ile `OOMKilled` farkı nedir?**
`OutOfMemoryError` heap'in dolduğunu gösterir ve JVM fırlatır. `OOMKilled` konteynerin toplam belleğinin aşıldığını ve işletim sisteminin uygulamayı öldürdüğünü gösterir.

**12. Graceful shutdown nedir?**
SIGTERM alınınca yeni istekleri reddedip işlenmekte olanların bitmesini bekleyip kapanmaktır. Spring Boot 3.4+ varsayılandır.

**13. Shell form `ENTRYPOINT` graceful shutdown'ı neden bozar?**
Java bir kabuk içinde çalışır ve SIGTERM kabuğa gider, Java'ya iletilmez. Uygulama SIGKILL ile öldürülür.

**14. Compose içindeki uygulama `localhost` ile veritabanına neden bağlanamaz?**
Konteyner içinde `localhost` konteynerin kendisidir. Servise adıyla (`postgres`) ulaşılmalıdır.

**15. `depends_on` ile `condition: service_healthy` ne sağlar?**
Bağımlı servisler gerçekten hazır olana kadar uygulamanın başlatılmasını bekletir.
