# PostgreSQL, Docker ve Flyway

## Bölüm 1: Docker ile PostgreSQL

### 1. Neden H2'den Ayrılıyoruz?

Geliştirmede H2, production'da PostgreSQL kullanmak = **iki farklı veritabanı için kod yazmak.** Native query'ler, veri tipleri, kısıt davranışları farklıdır; hatalar production'da görülür.

> **Kural:** Geliştirme ortamı production'a ne kadar benzerse, sürprizler o kadar az olur.

**PostgreSQL'i doğrudan kurmak yerine neden Docker?**

| Doğrudan Kurulum | Docker |
|---|---|
| İşletim sistemine göre farklı kurulum | Tek komut, her yerde aynı |
| Farklı sürümleri yönetmek zor | Her proje kendi sürümünü kullanır |
| Takımda "bende çalışıyor" sorunu | Herkes aynı ortamı kullanır |
| Kaldırınca kalıntı bırakır | Tek komutla iz bırakmadan silinir |

### 2. Temel Kavramlar

| Kavram | Açıklama | Java Benzetmesi |
|---|---|---|
| **Image** | Uygulamayı çalıştırmak için gereken her şeyi içeren **salt okunur paket** | Sınıf |
| **Container** | Bir imajın **çalışan örneği**. Birbirinden izole. | Nesne (`new`) |
| **Registry** | İmajların saklandığı yer (varsayılan: **Docker Hub**) | Maven Central |
| **Tag** | İmajın sürümü: `postgres:16-alpine` | Bağımlılık sürümü |

- `alpine`: Alpine Linux tabanlı, çok daha küçük varyant.
- ⚠️ **`latest` kullanılmaz.** Bugün 16, yarın 17'yi gösterebilir. Sürüm sabitlenir.
- **Container sanal makine değildir.** Ana makinenin çekirdeğini paylaşır, sadece dosyalarını ve süreçlerini izole eder. Saniyeler içinde başlar, az kaynak kullanır.

### 3. Tek Komutla PostgreSQL

```bash
docker run --name shop-postgres \
  -e POSTGRES_USER=shop \
  -e POSTGRES_PASSWORD=shop \
  -e POSTGRES_DB=shop \
  -p 5432:5432 \
  -v shop-pgdata:/var/lib/postgresql/data \
  -d postgres:16-alpine
```

| Parametre | Görevi |
|---|---|
| `--name` | Konteynere isim verir |
| `-e` | **Ortam değişkeni.** İmaj ayarlarını dışarıdan okur (externalized configuration). |
| `-p ana_makine:konteyner` | **Port eşlemesi.** Konteynerin kendi ağı vardır; bu olmadan uygulama veritabanına ulaşamaz. |
| `-v volume:yol` | **Volume.** Konteyner silinse bile veriler korunur. |
| `-d` | Arka planda çalıştırır |

> ⚠️ **Volume olmadan** konteyner silindiğinde **tüm veriler gider.** Konteynerler geçicidir.

> Bilgisayarda kurulu PostgreSQL 5432'yi kullanıyorsa: `-p 5433:5432` (uygulama 5433'e bağlanır).

### Günlük Komutlar

```bash
docker ps                          # çalışan konteynerler
docker ps -a                       # durmuşlar dahil hepsi
docker logs -f shop-postgres       # logları canlı takip et
docker stop shop-postgres          # durdur (veriler korunur)
docker start shop-postgres         # tekrar başlat
docker rm shop-postgres            # sil (volume korunur)

docker exec -it shop-postgres psql -U shop -d shop   # konteyner içinde psql
```

- `psql` içinde `\dt` → tabloları listeler.
- Konteyner çalışmıyorsa ilk bakılacak yer: **`docker logs`**

### 4. Docker Compose

Konteynerleri **dosyada tanımlayıp** tek komutla yönetmeyi sağlar. Proje kökünde `compose.yaml`:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: shop-postgres
    environment:
      POSTGRES_DB: shop
      POSTGRES_USER: shop
      POSTGRES_PASSWORD: shop
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U shop -d shop"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

- `docker run` parametrelerinin karşılıkları: `-e` → `environment`, `-p` → `ports`, `-v` → `volumes`
- Komut **ne yapılacağını** söyler, dosya **ne istendiğini** tarif eder. Git'te saklanır, herkes aynı şekilde kullanır.
- **`healthcheck`:** Konteynerin sadece başlamış değil, gerçekten **hazır** olduğunu anlamayı sağlar (`pg_isready`).
- Eski eğitimlerdeki `version: "3.8"` satırı artık gerekmez.

```bash
docker compose up -d        # başlat
docker compose ps           # durum (healthcheck dahil)
docker compose logs -f      # loglar
docker compose stop         # durdur
docker compose down         # durdur + konteynerleri sil (volume korunur)
docker compose down -v      # durdur + sil + VOLUME'LARI SİL ⚠️
```

> ⚠️ `down -v` veritabanındaki **tüm verileri siler.** Temiz başlangıç için kullanışlı, alışkanlıkla yazılırsa tehlikeli.

> `compose.yaml`'daki şifre sadece **yerel geliştirme** veritabanına ait, gizli değil. Git'e girebilir. Production şifreleri asla bu dosyaya girmez.

### 5. Spring Boot'u Bağlamak

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

- `runtime`: Kod sürücüye değil JDBC / JPA arayüzlerine bağımlı; sürücü sadece çalışma zamanında gerekir.
- H2 kaldırılabilir (testlerde Testcontainers kullanılıyor).

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/shop
    username: shop
    password: shop
```

- `jdbc:postgresql://` sürücü, `localhost:5432` port eşlemesi, `/shop` veritabanı adı.
- Compose dosyasındaki değerlerle **birebir aynı** olmalı.

> Eski eğitimlerdeki `spring.jpa.database-platform: ...PostgreSQLDialect` **gerekmez.** Hibernate 6 veritabanını ve sürümünü kendisi algılar.

### Spring Boot Docker Compose Desteği (3.1+)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-docker-compose</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

Uygulama başlarken:
1. `compose.yaml`'ı bulur, `docker compose up` çalıştırır.
2. Konteynerlerin hazır olmasını bekler.
3. Bağlantı bilgilerini okuyup `DataSource`'u **otomatik yapılandırır.** (`datasource` ayarı gerekmez)
4. Uygulama kapanınca konteynerleri durdurur (silmez).

> Testcontainers'taki **`@ServiceConnection`** ile aynı fikir: Bağlantı bilgileri konteyneri başlatan araçtan okunur.

- `optional`: Paketlenmiş uygulamaya geçmez, sadece geliştirme kolaylığı.
- Testlerde varsayılan olarak devre dışı.

### 6. PostgreSQL'e Geçerken Dikkat

| Konu | Açıklama |
|---|---|
| **Ayrılmış kelimeler** | `user`, `order` ayrılmış. PostgreSQL H2'den daha katı. `@Table(name = "users")` kullanılmalı. |
| **Büyük/küçük harf** | Tırnaksız isimler küçük harfe çevrilir. Native query'de `"costPrice"` gibi tırnaklı isimler bulunamaz. |
| **`IDENTITY` performansı** | Her `INSERT` tek tek ve hemen çalışır, **JDBC batch insert kullanılamaz.** Toplu eklemede `SEQUENCE` daha performanslı. |

---

## Bölüm 2: Flyway ile Şema Yönetimi

### 1. Sorun: ddl-auto: update Neden Yetmez?

| Senaryo | `update` Ne Yapar? |
|---|---|
| **Kolon adı değişti** (`name` → `title`) | Yeni **boş** `title` kolonu ekler. Eski `name` verileriyle kalır, okunmaz. |
| **Alan silindi** | Hiçbir şey silmez. `NOT NULL` ise yeni kayıtlar hata verir. |
| **Dolu tabloya zorunlu alan** | Mevcut satırlar için değer belirleyemez. |
| **Kayıt ve tekrarlanabilirlik** | Hiçbir kayıt tutmaz. Ortamların şemaları sessizce farklılaşır. |

> Kod için Git ne ise, veritabanı şeması için **migration** odur.

### 2. Migration ve Flyway

**Migration:** Şemadaki bir değişikliği tarif eden, **numaralı SQL dosyası.**

```
V1__create_catalog_tables.sql
V2__create_users_table.sql
V3__create_order_tables.sql
V4__add_description_to_products.sql
```

- Dosyalar sırayla, her biri **tam olarak bir kez** uygulanır.
- Boş veritabanına hepsi uygulanınca bugünkü şema elde edilir.
- Kodla birlikte Git'te yaşar, kod gibi gözden geçirilir.

**`flyway_schema_history` tablosu:** Her migration'ın sürümü, açıklaması, zamanı, başarısı ve **checksum**'ı (parmak izi) saklanır.

**Her başlangıçta:**
```
1. Geçmiş tablosuna bak: hangi sürümler uygulanmış?
2. db/migration klasörüne bak: hangi dosyalar var?
3. Uygulanmışların checksum'ı aynı mı? (Değilse DUR)
4. Uygulanmamışları sırayla çalıştır
5. Geçmiş tablosuna kaydet
```

> Alternatif: **Liquibase** (XML / YAML / SQL, daha fazla özellik, daha dik öğrenme eğrisi).

### 3. Kurulum

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

- ⚠️ **Flyway 10+**: Veritabanı desteği ayrı modülde. Eklenmezse **"Unsupported Database"** hatası.
- Sürüm yazılmaz, Spring Boot yönetir.
- Spring Boot Flyway'i **otomatik çalıştırır**, Hibernate'ten **önce**.
- Konum: `src/main/resources/db/migration/`

### İsimlendirme

```
V1__create_catalog_tables.sql
│ │ │
│ │ └─ açıklama
│ └─── İKİ alt çizgi
└───── V + sürüm
```

> ⚠️ **Tek alt çizgi** (`V1_create...`) → Dosya tanınmaz ve **sessizce atlanır.**

- Sürümler sayı olarak karşılaştırılır: `V10` > `V9`. `V1.1` gibi ara sürümler yazılabilir.

### ddl-auto: validate

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```

- Hibernate şemayı **değiştirmez**, sadece entity'lerin tablolarla uyuştuğunu **kontrol eder.**
- Migration unutulursa uygulama **başlarken** patlar: `Schema-validation: missing column [description] in table [products]`

> **Flyway şemayı değiştirir, Hibernate sadece kontrol eder.** Şemanın tek sahibi var.

### 4. İlk Migration'lar

```sql
-- V1__create_catalog_tables.sql

CREATE TABLE categories (
    id   BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE products (
    id          BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name        VARCHAR(100)   NOT NULL UNIQUE,
    price       NUMERIC(10, 2) NOT NULL,
    cost_price  NUMERIC(10, 2) NOT NULL,
    stock       INTEGER        NOT NULL DEFAULT 0,
    category_id BIGINT         NOT NULL,
    CONSTRAINT fk_products_category FOREIGN KEY (category_id) REFERENCES categories (id)
);

CREATE INDEX idx_products_category_id ON products (category_id);
```

| SQL | Entity |
|---|---|
| `BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY` | `@Id @GeneratedValue(strategy = IDENTITY)` |
| `VARCHAR(100) NOT NULL UNIQUE` | `@Column(nullable = false, length = 100, unique = true)` |
| `NUMERIC(10, 2)` | `@Column(precision = 10, scale = 2) BigDecimal` |
| `cost_price` | `costPrice` |
| `FOREIGN KEY ... REFERENCES` | `@ManyToOne @JoinColumn(name = "category_id")` |

- **Kısıtlara isim ver** (`CONSTRAINT fk_...`). Verilmezse rastgele isim üretilir, ileride değiştirmek zorlaşır.
- ⚠️ **PostgreSQL foreign key'lere otomatik index oluşturmaz.** Index olmadan JOIN'ler ve "bu kategorinin ürünleri" sorguları tablo büyüdükçe yavaşlar. **FK kolonlarına index eklenir.**

```sql
-- V2__create_users_table.sql

CREATE TABLE users (
    id       BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    username VARCHAR(50)  NOT NULL UNIQUE,
    password VARCHAR(100) NOT NULL,  -- BCrypt 60 karakter, pay bırakıldı
    role     VARCHAR(20)  NOT NULL   -- EnumType.STRING
);
```

> `ddl-auto` ile oluşturulmuş tablolar varsa V1 "tablo zaten var" hatası verir. Geliştirmede: `docker compose down -v` ile sıfırla.

### 5. Şemayı Değiştirmek

**Alan eklemek:**

```sql
-- V4__add_description_to_products.sql
ALTER TABLE products ADD COLUMN description TEXT;
```

```java
@Column(columnDefinition = "TEXT")
private String description;
```

> Migration ve entity değişikliği **aynı commit'te** olmalı.

**Yeniden adlandırma (veriler korunur):**

```sql
-- V5__rename_product_name_to_title.sql
ALTER TABLE products RENAME COLUMN name TO title;
```

### Dolu Tabloya Zorunlu Alan: Üç Adımlı Kalıp

```sql
-- V6__add_sku_to_products.sql

-- 1. EKLE: NULL kabul edecek şekilde
ALTER TABLE products ADD COLUMN sku VARCHAR(50);

-- 2. DOLDUR: Mevcut satırlar
UPDATE products SET sku = 'SKU-' || id;

-- 3. KISITLA: Artık her satırın değeri var
ALTER TABLE products ALTER COLUMN sku SET NOT NULL;
ALTER TABLE products ADD CONSTRAINT uq_products_sku UNIQUE (sku);
```

> Doğrudan `ADD COLUMN sku ... NOT NULL` → Mevcut satırlar yüzünden **reddedilir.**

- Migration'lar `UPDATE` ve `INSERT` de içerebilir: Şemayla birlikte **veri** de dönüştürülür.
- **PostgreSQL'de** DDL dahil migration'lar transaction içinde çalışır. Bir adım başarısız olursa hepsi geri alınır. (MySQL'de DDL geri alınamaz.)

### 6. Altın Kural: Uygulanmış Migration'a Dokunma

```
Validate failed: Migration checksum mismatch for migration version 1
```

- Uygulanmış dosya değişirse checksum uyuşmaz, Flyway uygulamayı **başlatmaz.**
- **Neden?** Production şeması dosyanın **eski** halinden oluştu. Dosya değişirse yeni ortamlar farklı şema elde eder ve kimse bilmez.

> **Doğru yol:** Hatayı **yeni bir migration** ile düzelt. Migration'lar **yalnızca ileri** gider. (Git'te yayınlanmış commit'i değiştirmemek gibi.)

| Durum | Düzenlenebilir mi? |
|---|---|
| Sadece senin bilgisayarında, Git'e gönderilmemiş | ✅ Düzelt + `docker compose down -v` |
| Başka bir ortama ulaşmış (takım, test, production) | ❌ Yeni migration yaz |

- Uygulanmış migration **silinmez** de; Flyway yine hata verir.

### Rollback

- Flyway'in ücretsiz sürümünde otomatik **undo yok** (ücretli özellik).
- Production'da veri içeren şemayı geri almak çoğu zaman güvenli değildir (silinen kolonun verisi geri gelmez).
- **Geri alma da ileri doğru bir migration'dır:** V6'da eklenen kolon V7'de `DROP COLUMN` ile kaldırılır.

### 7. Takım Çalışması

```
Found more than one migration with version 7
```

- İki geliştirici aynı sürüm numarasıyla migration oluşturursa uygulama başlamaz.
- Çözüm: Biri dosyasını yeniden numaralandırır (henüz uygulanmadığı için serbest).
- Büyük takımlarda **tarih-saat tabanlı** sürümler: `V20261005_1430__add_sku_to_products.sql`

### 8. Başlangıç Verileri

| Veri | Nereye? |
|---|---|
| Her ortamda olması gereken (varsayılan kategoriler) | **Migration** |
| Sadece geliştirme için (test ürünleri, deneme kullanıcıları) | **`@Profile("dev")` seeder** |

```sql
-- V8__insert_default_categories.sql
INSERT INTO categories (name) VALUES ('Elektronik'), ('Kitap'), ('Giyim');
```

### 9. Testler ve Flyway

`@DataJpaTest` ve `@SpringBootTest`, Flyway'i de çalıştırır. Testcontainers ile test başlarken:
1. Boş PostgreSQL'e **tüm migration'lar** sırayla uygulanır.
2. Hibernate `validate` ile entity'leri kontrol eder.

**Ekstra test yazmadan doğrulananlar:**
- Her migration geçerli PostgreSQL SQL'i
- Sıralama doğru
- Şema sıfırdan kurulabiliyor
- Entity'ler şemayla uyuşuyor

> H2 PostgreSQL'e özel SQL'i anlamayabilirdi. Testcontainers ile **test, migration ve production aynı veritabanında buluşur.**

### Mevcut Veritabanına Flyway Eklemek

`ddl-auto` ile oluşturulmuş, veri içeren veritabanında Flyway boş olmayan şema görünce hata verir.

```yaml
spring.flyway.baseline-on-migrate: true
```

Mevcut şemayı "başlangıç noktası" olarak işaretler, sonrasını Flyway yönetir. Yeni projelerde gerekmez.

---

## Örnek Sorular

**1. Docker'da image ile container farkı nedir?**
Image salt okunur bir pakettir, container onun çalışan örneğidir. Bir imajdan birden fazla izole konteyner başlatılabilir (sınıf ve nesne gibi).

**2. Volume olmadan başlatılan PostgreSQL konteyneri silinirse ne olur?**
Tüm veriler silinir. Konteynerler geçicidir; volume verileri konteynerin dışında kalıcı saklar.

**3. Container ile sanal makine farkı nedir?**
Sanal makine kendi tam işletim sistemini çalıştırır. Container ana makinenin çekirdeğini paylaşır, sadece dosya ve süreçleri izole eder; daha hafif ve hızlıdır.

**4. Port 5432 doluysa Docker'daki PostgreSQL nasıl çalıştırılır?**
Ana makine tarafındaki port değiştirilir: `-p 5433:5432`. Uygulama 5433'e bağlanır.

**5. İmaj etiketi olarak neden `latest` kullanılmaz?**
Zamanla farklı sürümleri gösterebilir, ortam habersizce değişir. Sürüm sabitlenmelidir.

**6. `docker compose down` ile `down -v` farkı nedir?**
İkisi de konteynerleri siler; `-v` volume'ları da siler ve tüm veriler kaybolur.

**7. Spring Boot Docker Compose desteği ne yapar?**
Uygulama başlarken `compose.yaml`'daki servisleri başlatır ve bağlantı bilgilerini okuyup `DataSource`'u otomatik yapılandırır. Testcontainers'taki `@ServiceConnection` ile aynı fikirdir.

**8. `ddl-auto: update` kolon adı değişikliğinde ne yapar?**
Yeniden adlandırmayı anlayamaz; yeni boş bir kolon ekler, eski kolon verileriyle kalır.

**9. Flyway uygulanan migration'ları nasıl takip eder?**
`flyway_schema_history` tablosunda sürüm, açıklama, zaman, başarı ve checksum saklar. Sadece uygulanmamış dosyaları çalıştırır.

**10. `V3_add_column.sql` neden çalışmaz?**
Sürüm ve açıklama arasında iki alt çizgi olmalıdır. Tek alt çizgili dosya sessizce atlanır.

**11. Flyway ile birlikte `ddl-auto` neden `validate` olmalıdır?**
Şemanın tek sahibi Flyway olmalıdır. `validate` Hibernate'in şemayı değiştirmesini engeller ve uyumsuzlukta uygulamayı başlatmaz.

**12. Dolu tabloya `NOT NULL` kolon nasıl eklenir?**
Önce nullable eklenir, sonra mevcut satırlar doldurulur, en son `SET NOT NULL` uygulanır.

**13. Uygulanmış bir migration değiştirilirse ne olur?**
Checksum uyuşmazlığı nedeniyle uygulama başlamaz. Hata yeni bir migration ile düzeltilmelidir; sadece hiçbir ortama gönderilmemiş yerel migration'lar düzenlenebilir.

**14. Flyway'de bir migration nasıl geri alınır?**
Ücretsiz sürümde undo yoktur. Geri alma, değişikliği tersine çeviren yeni bir migration ile yapılır.

**15. PostgreSQL'de FK kolonlarına neden index eklenir?**
PostgreSQL FK'lere otomatik index oluşturmaz; index olmadan JOIN'ler ve ilişki sorguları yavaşlar.

**16. Testcontainers + Flyway kullanımı hangi kontrolleri otomatik sağlar?**
Migration'ların geçerli SQL olduğu, sıralarının doğru olduğu, şemanın sıfırdan kurulabildiği ve entity'lerin şemayla uyuştuğu.
