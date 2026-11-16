# Midnight Kitchen: Karavan — Teknik Devir Teslim

> **Tarih:** 2026-11-16
> **Durum:** Faz 0 tamamlandı; Faz 1 (prototip) başladı.
> **Daha fazla ayrıntı:** Düğüm düğüm Blueprint dökümü ve hata geçmişi `Docs/DevirTeslim_MidnightKitchen.md` dosyasında. Tasarımın kaynağı proje kökündeki `Midnight Kitchen_ Karavan - Game Design Document (v2.1, Unreal Engine 5.8.4).pdf` dosyası.

---

## 1. Proje Genel Durumu ve Mimari

### Amaç
İstanbul'da gece yarısı seyyar bir karavanda geçen, kara mizahlı bir **zaman yönetimi / yemek** oyunu.

- **Sabit kamera:** Oyuncu tek mekânda (karavanın içinde) kalır.
- **Tek müşteri:** Pencerede aynı anda yalnızca bir müşteri olur.
- **Müşteri:** Pencereye gelir, sipariş verir ve beklerken sabrı azalır.
- **Oyuncu:** Malzemeleri sürükleyip istasyonlarda işler (doğrar, pişirir) ve pencereye servis eder.
- **Sonuç:** Servis kalitesine göre ödül ya da ceza gelir: para, bahşiş, itibar, sağlık.
- **Oyun süresi:** 30 gece. Her gece 6-8 dakika gerçek süre sürer.
- **Platform ve dil:** Yalnızca PC, yalnızca Türkçe.
- **Görsel stil:** 3B stilize, kansız ve mizahi.

### Fazlar (GDD)

| Faz | İçerik | Çıkış kriteri | Durum |
|---|---|---|---|
| 0 Önüretim | Kurulum, klasörler, Data Table | Greybox'ta sürükle-bırak çalışır | ✅ Tamam |
| 1 Prototip | 3 istasyon, 3 tarif, Normal + Mafya, sabır | 10 dk oynanabilir döngü | 🔄 Devam ediyor |
| 2 Genişletme | Tüm istasyonlar, ekonomi, gün sonu, yükseltme | 10 gecelik dikey dilim | ⏳ |

### Motor ve altyapı
- **Motor:** Unreal Engine **5.8.4** (`C:\Program Files\Epic Games\UE_5.8`). Proje `MyProject4.uproject` dosyası.
- **Başlangıç şablonu:** Third Person. Şablondaki dosyalar hâlâ projede duruyor ama oyun onları kullanmıyor.
- **Kod:** **Yalnızca Blueprint.** C++ modülü, `Source/` klasörü ve `.sln` dosyası yok.
  - GDD sipariş, sabır ve ekonomi sistemlerinin C++ ile yazılmasını öngörüyor.
  - Bu sistemleri şimdilik Blueprint ile kurma kararı verildi. Faz 1'den sonra C++'a taşınıp taşınmayacağı henüz belli değil.
- **Sürüm kontrolü:** Yok. Klasör bir git reposu değil, makinede git de kurulu değil. Yedeklemeye dikkat et.

### Mimari ilkeler
- **İçerik yeri:** Oyunun bütün içeriği `Content/MidnightKitchen/` altında duruyor. Şablon klasörlerine dokunulmuyor.
- **Veriler tablolarda:**
  - Müşteri, malzeme ve tarif değerleri **Data Table**'lardan okunuyor.
  - Blueprint'lere sabit sayı yazılmıyor.
- **Arayüz push modeliyle çalışıyor:**
  - `BP_OrderManager`, widget fonksiyonlarını kendisi çağırıyor.
  - Property binding kullanılmıyor.
- **Sabır süresi:** Event Tick'te her karede `Delta Seconds` kadar azalıyor. Sayaç yalnızca `bCustomerWaiting` true iken işliyor.

### Klasör düzeni
```text
Content/MidnightKitchen/
├─ Blueprints/
│  ├─ Core/
│  │  ├─ BP_OrderManager            ← oyun döngüsünün merkezi
│  │  ├─ BP_Ingredient              ← sürüklenebilir malzeme
│  │  ├─ BP_KaravanGameMode
│  │  └─ BP_KaravanPlayerController
│  ├─ Customers/                    (boş)
│  └─ Stations/                     (boş — istasyonlar buraya gelecek)
├─ Data/Structs/
│  ├─ S_CustomerData
│  ├─ S_IngredientData
│  └─ S_RecipeData
├─ DataTables/
│  ├─ DT_Customers
│  ├─ DT_Ingredients
│  └─ DT_Recipes
├─ Maps/Lvl_Karavan_Greybox         ← çalışılan level
├─ WBP_PatienceBar                  (hedef konum UI/, henüz taşınmadı)
├─ E_DishQuality                    (hedef konum Data/Structs/, henüz taşınmadı)
└─ UI/, Input/, Art/, Audio/        (boş)
```

---

## 2. Son Eklenen / Aktif Sistemler

### 2.1 Müşteri ve sabır döngüsü (`BP_OrderManager`)
Level'a yerleştirilmiş bir Actor. Oyun döngüsünün tamamını bu Blueprint yönetiyor.

1. **BeginPlay:**
   - `WBP_PatienceBar` widget'ını oluşturup ekrana ekler.
   - Klavye girdisini açar.
   - `SpawnNextCustomer` olayını çağırır.
2. **SpawnNextCustomer:**
   - `DT_Customers` tablosundan rastgele bir müşteri seçer.
   - Sabrı `BasePatience × PatienceMultiplier` olarak hesaplar.
   - `DT_Recipes` tablosundan rastgele bir tarif seçer ve widget'a sipariş olarak yazar.
3. **Event Tick:**
   - Kalan sabrı her karede azaltır ve barı günceller.
   - Sabır 0'a inerse `OnPatienceExpired` çalışır: ceza ekrana yazdırılır, 2 sn sonra yeni müşteri gelir.
4. **ServeOrder(Quality):** `E_DishQuality` değerine göre servisi sonuçlandırır.
   - **Mükemmel / İyi:** Ödül mesajı yazdırılır, 2 sn sonra yeni müşteri gelir.
   - **Yanık:** Müşteri kabul etmez. Maksimum sabrın %50'si düşülür ve müşteri beklemeye devam eder.

`PatienceMultiplier` değişkeni level'daki örnekten düzenlenebiliyor. Testte 0.2 (hızlı test), normal oyunda 1.0 olmalı.

### 2.2 Veri tabloları

**DT_Customers** (satır tipi `S_CustomerData`). Ceza değerleri pozitif sayı olarak tutuluyor; çıkarma işlemini Blueprint yapıyor.

| Row | DisplayName | BasePatience | ReputationPenalty | HealthPenalty | EarningsPenaltyPercent |
|---|---|---|---|---|---|
| Normal | Normal İnsan | 45 | 5 | 0 | 0 |
| Mafya | Mafya | 30 | 0 | 25 | 20 |

**DT_Ingredients** (satır tipi `S_IngredientData`; alanlar: `DisplayName`, `bNeedsGrill`, `bNeedsChop`)

| Row | Izgara | Doğrama |
|---|---|---|
| Kofte | ✔ | |
| Kokorec | ✔ | ✔ (önce ızgara, sonra doğrama) |
| Ekmek | | |
| Domates | | ✔ |
| Salatalik | | ✔ |

**DT_Recipes** (satır tipi `S_RecipeData`; alanlar: `DisplayName`, `Ingredients` = malzeme satır adlarından oluşan dizi)

| Row | Ingredients |
|---|---|
| KofteEkmek | Kofte, Ekmek |
| Kokorec | Kokorec, Ekmek |
| CobanSalata | Domates, Salatalik |

**E_DishQuality** (Enum): `Mukemmel`, `Iyi`, `Yanik`.

### 2.3 Sürükle-bırak (`BP_Ingredient`)
Level'da 5 tane örnek var: Köfte, Kokoreç, Ekmek, Domates, Salatalık.

**Malzeme verisi:**
- Construction Script, `IngredientId` değerine bakıp `DT_Ingredients` tablosundan ilgili satırı okur.
- Malzemenin adı, üstündeki `NameText` yazısına aktarılır.

**Bileşenler:**
- `DefaultSceneRoot`: Mobility ayarı **Movable** olmalı.
- `Mesh`: Sphere, ölçek 0.15.
- `NameText`: Text Render.

**Değişkenler:** `IngredientId`, `IngredientData`, `bIsCooked`, `bIsChopped`, `bIsDragging`, `StartLocation`.

**Akış:**
1. **Tıklama:** `bIsDragging` true olur. Mesh'in Visibility kanalı geçici olarak **Ignore** yapılır. Böylece imleç çizgisi malzemenin kendisine çarpmaz.
2. **Sürükleme (Tick):** Sol tuş basılıyken malzeme, imlecin altındaki noktanın 30 cm üstünde durur. Bunun için `Get Hit Result Under Cursor by Channel` kullanılıyor.
3. **Bırakma:** Malzeme o noktanın 8 cm üstüne konur. Visibility kanalı **en son** yeniden Block yapılır.

### 2.4 Pencereye teslim (en son eklenen)
Malzemeler tabak kullanılmadan, **tek tek** pencereye sürüklenip teslim ediliyor.

- **Teslim anı:** Malzeme `Servis` etiketli aktöre bırakılınca (`Actor Has Tag`), `Get Actor Of Class(BP_OrderManager)` üzerinden `DeliverIngredient(IngredientId)` çağrılır.
- **Takip:** `BP_OrderManager`, teslim edilenleri `DeliveredIngredients` dizisinde tutar.
- **Sipariş tamamlanması:** Tarifteki bütün malzemeler gelince sipariş biter. Dizi yeni müşteride temizlenir.
- ⚠️ **Henüz yok:** Malzemenin pişmiş ya da doğranmış olup olmadığı kontrol edilmiyor. Çiğ köfte de kabul ediliyor.
- ℹ️ **Doğrulanmadı:** Bu akış asset dosyalarından çıkarıldı. Hangi aktörün `Servis` etiketini taşıdığını (büyük ihtimalle `Musteri_Yeri`) ve `StartLocation`'ın nasıl kullanıldığını editörde kontrol et.

### 2.5 Arayüz (`WBP_PatienceBar`)
- **Yerleşim:** Ekranın üst ortasında. İçinde müşteri adı (`CustomerNameText`), sabır barı (`PatienceBar`) ve sipariş yazısı (`OrderText`) var.
- **Fonksiyonlar:** `SetCustomer(Name)`, `SetPatience(Percent)`, `SetOrder(Order)`.
- **Bar renkleri:** Doluluk %50'nin üstündeyse yeşil, %25-50 arasında sarı, %25'in altındaysa kırmızı.

### 2.6 Kamera, girdi ve level
- **`BP_KaravanGameMode`:**
  - Default Pawn yok (None). Oyunda karakter yok.
  - Player Controller olarak `BP_KaravanPlayerController` kullanılıyor.
- **`BP_KaravanPlayerController`:**
  - Fare imleci görünür; tıklama ve üzerine gelme olayları açık.
  - Input Mode: `Game And UI`.
- **`Lvl_Karavan_Greybox`:**
  - World Settings'te GameMode Override olarak `BP_KaravanGameMode` seçili.
  - Şef tezgâhın −X tarafında, müşteri +X tarafında duruyor.

| Aktör | Şekil | Location (X, Y, Z) | Scale (X, Y, Z) |
|---|---|---|---|
| Tezgah | Cube | 0, 0, 45 | 1, 4, 0.9 |
| Istasyon_Malzeme | Cube | 0, −130, 100 | 0.6, 0.8, 0.2 |
| Istasyon_Dograma | Cube | 0, 0, 95 | 0.6, 0.8, 0.1 |
| Istasyon_Izgara | Cube | 0, 130, 100 | 0.6, 0.8, 0.2 |
| Musteri_Yeri | Cylinder | 200, 0, 100 | 0.6, 0.6, 2 |
| CameraActor | — | −250, 0, 220 (Pitch −25) | Auto Activate: Player 0 |
| Malzemeler | BP_Ingredient | 0, −160…−100 (15 aralıkla), 118 | — |

İstasyon küpleri şu an yalnızca görsel. Henüz hiçbir mantıkları yok.

---

## 3. Kritik Dosyalar ve Konumları

İnceleme için önerilen sıra:

1. **`HANDOFF.md`**: Bu dosya.
2. **`Docs/DevirTeslim_MidnightKitchen.md`**: Bütün Blueprint'lerin düğüm düğüm dökümü, verilen kararlar ve çözülen hataların listesi.
3. **GDD PDF'i** (proje kökünde): Tasarımın kaynağı. Bu belgeyle çelişirse GDD geçerli.
4. **`Content/MidnightKitchen/Maps/Lvl_Karavan_Greybox`**: Bu level'ı aç ve Play'e bas.
5. **`Blueprints/Core/BP_OrderManager`**: Event Graph'ta `SpawnNextCustomer`, `Tick`, `OnPatienceExpired`, `ServeOrder` ve `DeliverIngredient` bölümlerine bak.
6. **`Blueprints/Core/BP_Ingredient`**: Sürükle-bırak ve teslim mantığı.
7. **`DataTables/DT_*`** ve **`Data/Structs/S_*`**: Veri şeması.
8. **`WBP_PatienceBar`**: Arayüz.
9. **`Config/DefaultEngine.ini`**: Lumen ayarları burada (bkz. §5).

**Play'de görmen gerekenler:**
- Sabit kamera ve görünür fare imleci.
- Üstte müşteri adı, sipariş ve rengi değişen sabır barı.
- Malzemeler sürüklenebiliyor ve pencereye bırakılınca teslim ediliyor.
- 1 / 2 / 3 tuşları Mükemmel / İyi / Yanık servisi tetikliyor (geçici).

---

## 4. Sıradaki Hedefler (TODO)

### Faz 1 — şu an
- [ ] **Doğrama tahtası:** Malzeme tahtaya bırakılır, üzerine basılı tutulunca `bIsChopped = true` olur.
  - Basılı tutma süresi henüz karar bekliyor (GDD "1-3 sn" diyor).
  - Seçilen süre `DT_Ingredients` tablosuna yeni bir alan olarak eklenecek.
- [ ] **Izgara:** Bir zamanlama çubuğu olacak, içinde "mükemmel bölge" bulunacak. Sonuç `bIsCooked` değerini ve servis kalitesini (Mükemmel / İyi / Yanık) belirleyecek.
- [ ] **Teslim kontrolü:** `DeliverIngredient` yalnızca gerekli işlemleri bitmiş malzemeyi kabul edecek (`bNeedsGrill` → `bIsCooked`, `bNeedsChop` → `bIsChopped`).
  - Kokoreç için kural: pişmeden doğranamaz.
- [ ] **Kalitenin servise bağlanması:** Sipariş tamamlanınca `ServeOrder` gerçek kaliteyle çağrılacak, ardından **1/2/3 test tuşları kaldırılacak.**
- [ ] **Tabak kararı:** "Birleştir" adımı için servis tabağı önerilmişti, şu an tabaksız teslim var. İkisi de GDD'ye uyuyor; birine karar verilecek.
- [ ] **Mafya ardışık etkisi:** Mafya memnun ayrılırsa sonraki müşterinin sabrı +%10, ceza verdiyse −%10 olacak.
  - `CurrentCustomerId` tanımlı ama hiçbir yerde atanmıyor. Bu iş için önce atanması gerekiyor.
- [ ] **Faz 1 çıkış testi:** 10 dakikalık oynanabilir döngü. Erken playtest yapılmalı (GDD'deki "sıkıcı pişirme döngüsü" riski).

### Faz 2 — sonra
- **Ekonomi:** Para, itibar ve sağlık hesabı; Haydar'ın payı. Ödül ve cezalar şu an yalnızca ekrana yazdırılıyor.
- **Gece akışı:** Gün sonu ve yükseltme ekranları, gece yapısı (vardiya saati, müşteri sayısı).
- **İçerik:** Yeni müşteri tipleri ve gece olayları.

### Toparlama (her an yapılabilir)
- `WBP_PatienceBar` → `UI/` klasörüne, `E_DishQuality` → `Data/Structs/` klasörüne taşınacak. İçerik Tarayıcısı'nda taşıdıktan sonra **Fix Up Redirectors** çalıştır.
- Git ve Git LFS kurulup proje sürüm kontrolüne alınacak.

### GDD'de cevabı bekleyen sorular (sayı uydurmadan önce ekiple karar verin)
- **Tempo:**
  - Tek müşteri kuralı ve 6-8 dakikalık vardiyayla 10. gecedeki 26 müşteri, müşteri başına yaklaşık 14-18 saniye ediyor.
  - Oysa mikro döngü 10-40 saniye sürüyor. Faz 1 dengesinden önce çözülmeli.
- **Tanımsız müşteriler:** Vampir, Zombi, Kurt adam, Kaçakçı ve Kırbaççı için değer yok.
- **Haydar ödemesi:** Sıklığı tutarsız yazılmış, tutarı yok.
- **Ekonomi değerleri:** Yemek fiyatı, bahşiş tabanı ve başlangıç parası yok.
- **Hikâye ve geç oyun:** 3 sonun koşulları, 11-30. gecelerin zorluğu, bıçak mini oyunu ve "gizli menü" kuralları tanımsız.

---

## 5. Dikkat Edilmesi Gerekenler / Bilinen Detaylar

### Blueprint tuzakları (hepsi bu projede yaşandı)
1. **Exec teli bağlı değil:**
   - Compiler "...was pruned because its Exec pin is not connected" uyarısı veriyorsa düğüm hiç çalışmıyor demektir.
   - Beyaz exec tellerini her zaman kontrol et.
2. **Bool'u true yapan Set düğümünde kutucuk işaretsiz:** Değer hep false kalıyor (`bCustomerWaiting` bu yüzden çalışmamıştı).
3. **Get yerine Set kullanmak:**
   - Girişi boş bir `Set PatienceWidget` düğümü, referansı None yapıyor.
   - Okumak için **Get** kullan (Ctrl + sürükle). Set yalnızca BeginPlay'de kalmalı.
4. **Yanlış SetText sürümü:**
   - Boş alana sağ tıklayıp SetText eklersen RichTextBlock sürümü gelebiliyor.
   - Değişkenin pininden sürükleyerek ekle.
5. **Get Data Table Row Names:** `then` exec çıkışını bağlamayı unutma.
6. **Get Player Controller:**
   - Player State'e ait sürümü (Target pinli) değil, **global sürümü** (Player Index pinli) kullan.
7. **"DefaultSceneRoot has to be 'Movable'" hatası:**
   - `BP_Ingredient` içinde ve level'daki örneklerinde Mobility = **Movable** olmalı.
8. **Bırakma sırası:**
   - Collision'ı `Set Actor Location`'dan **sonra** Block yap.
   - Önce yaparsan malzeme kendine çarpar ve her bırakışta 8 cm yükselir.
9. **Malzeme Play'de görünmez oluyorsa:**
   - Level örneğinde **Actor Hidden In Game** işaretli olabilir. Details'te `Hidden` diye ara.
   - Köfte'de bu yaşandı.

### Motorla ilgili notlar
- UE 5.8'de `Get Hit Result Under Cursor by Channel` saf (pure) bir düğüm; exec pini yok.
- `bIsDragging` Instance Editable değil, bu yüzden level örneğinin Details panelinde görünmez. Normal bir durum.

### GDD'den bilinçli sapmalar ve geçici kararlar
- **Lumen açık.** GDD Lumen ve Nanite kapalı, kontrollü ışık istiyor (`Config/DefaultEngine.ini`: `r.DynamicGlobalIlluminationMethod=1`, `r.ReflectionMethod=1`). Performans çalışmasında kapatılmalı.
- **Common UI** projede etkin değil. GDD UMG / Common UI ve gamepad desteği istiyor.
- **Sabır barı renk eşikleri** (%50 / %25) GDD'de yok; geçici karar.
- **Yanık servis:** Kalan sabrın değil, **maksimum** sabrın %50'si düşülüyor. Şimdilik bütün müşteriler yanık yemeği reddediyor.
- **Ödül ve cezalar** sadece ekrana yazdırılıyor; gerçek değerler Faz 2'deki ekonomi sistemiyle gelecek.
- **Test ayarı:** `PatienceMultiplier` testte 0.2 olabilir; normal oyunda 1.0 yap.

### Çalışma notları
- **Sayı uydurma:** GDD'de olmayan bir sayı gerekirse ekiple karar verin, kararı belgeye "geçici karar" olarak yazın.
- **Değerlerin yeri:** Değerleri Blueprint'e sabit yazmak yerine Data Table'a koyun.
- **Sürüm kontrolü:** `.uasset` dosyaları ikili. Git gelirse **Git LFS** şart. Aynı asset'i iki kişi aynı anda düzenlemesin, çünkü birleştirilemez.
- **Belgeleri güncel tutma:** Her yeni sistem ya da çözülen hata için bu dosyayı ve `Docs/DevirTeslim_MidnightKitchen.md` dosyasını güncelleyin.
