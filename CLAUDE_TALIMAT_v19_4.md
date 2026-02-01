# CLAUDE TALİMAT DOSYASI - BIST Arbitraj Trading System

> **ÖNEMLİ:** Bu dosyayı her yeni sohbetin başında Claude'a ver. Bu dosya projenin tüm kritik bilgilerini içerir.
> **VERSİYON:** v19.4 - 2026-02-01

---

## 1. PLATFORM BİLGİSİ

**KULLANILAN PLATFORM:** IdealData Portföy Ekranı
- IdealPro DEĞİL (bu ismi kullanma)
- MatriksIQ DEĞİL
- Robot C# kodu IdealData Portföy ekranından çalıştırılır
- **C# 5.0** - Null-conditional operator (`?.`) ve string interpolation (`$""`) DESTEKLENMEZ!

**ROBOT DURDURMA:**
- Lokal: `STOP.txt` oluştur + IdealData'da "Yok" seç
- Lokal: Form'daki "Çıkış" butonuna bas
- Uzaktan: Dashboard'dan STOP butonu (Gist üzerinden)

---

## 2. PİYASA SAATLERİ VE VADE BİLGİSİ

### Piyasa Saatleri
```
BIST Pay Piyasası:
10:00 - 18:00  : Sürekli işlem seansı (KESİNTİSİZ - öğle arası YOK!)

VİOP (Vadeli İşlem Piyasası):
09:30 - 18:10  : Sürekli işlem seansı (KESİNTİSİZ)

Robot çalışma saatleri: 10:00 - 18:09
SN Spot açılışı için: BIST açık olmalı (10:00-18:00)
Pair Trading saatleri: 10:05 - 17:59 (SPOT fiyat bağımlılığı!)
```

**ÖNEMLİ:** Bayram arefelerinde yarım gün işlem olabilir (genellikle 12:30'da kapanış)

### Vade Sonu
- **Pay vadeli sözleşmelerin vade sonu: Her ayın SON İŞ GÜNÜ**
- Ayın son günü hafta sonuna denk gelirse, bir önceki Cuma son iş günüdür
- Örnek: Ocak 2026 → 30 Ocak (Cuma), Şubat 2026 → 27 Şubat

---

## 3. ÜÇ STRATEJİ SİSTEMİ

### 3.1 Spot-Near (SN) Stratejisi
**Ana arbitraj stratejisi - Production ready**

| Özellik | Değer |
|---------|-------|
| Hedef | Spot AL + Near SAT, vade sonunda birleş |
| Risk | Düşük (arbitraj) |
| Bütçe | SN_BUDGET (VİOP) + SPOT_BUDGET |

⚠️ **BB93+ pozisyonlarına gün içi kâr ve skor sistemi UYGULANMAZ!**

### 3.2 Near-Far (NF) Stratejisi  
**Calendar spread - Test aşamasında**

| Özellik | Değer |
|---------|-------|
| Hedef | Near AL + Far SAT, spread daralınca kapat |
| Risk | Orta (spread değişebilir) |
| Bütçe | NF_BUDGET (ayrı VİOP bütçesi) |

⚠️ **NF'de gün içi kâr ve skor sistemi YOK! Sadece BB ve sabit kâr eşiği.**

### 3.3 Pair Trading Stratejisi (v19)
**Kointegrasyon bazlı VIOP Near-Near çift işlemleri**

| Özellik | Değer |
|---------|-------|
| Hedef | Kointegre çiftlerde Z-Score normalleşmesi |
| Risk | Orta-Yüksek (kointegrasyon bozulabilir) |
| Bütçe | PAIR_BUDGET = 100,000 TL (izole) |
| Çiftler | TCELL-TTKOM (β=1.80), CIMSA-OYAKC (β=2.03) |
| **Saatler** | **10:05 - 17:59** (SPOT fiyat bağımlılığı!) |

**Z-Score Yorumlama:**

| Z-Score | Anlam | İşlem |
|---------|-------|-------|
| Z > +2 | A pahalı (spread yüksek) | A SELL, B BUY |
| Z < -2 | A ucuz (spread düşük) | A BUY, B SELL |

Örnek: TCELL-TTKOM Z=+2.5 → TCELL SAT, TTKOM AL

### Çakışma Kuralları
- **PAIR_BLOCKED sembolleri:** SN ve NF'de bu semboller kullanılmaz
  - TCELL, TTKOM, CIMSA, OYAKC

---

## 4. v19.4 DEĞİŞİKLİKLER (2026-02-01)

### Yeni Özellikler

**1. SERMAYE ARTIRIMI KORUMA SİSTEMİ**
- Piramit Menkul PDF'den sermaye artırımı takvimi çekilir
- `srm_scraper.py` - Python scraper
- `srm.json` - Sermaye artırımı verileri
- **Giriş engeli:** Bugün VEYA 7 gün içinde sermaye artırımı varsa SN/NF/Pair giriş yapılmaz
- **Otomatik kapatma:** 10:10'da mevcut pozisyonlar kapatılır
- 09:25'te günlük güncelleme (temettü ile aynı zamanda)

**Sermaye Artırımı Riski:**
```
ÖNCE:  SPOT 100 lot LONG + NEAR 100 lot SHORT = Hedge ✓
SONRA: SPOT 150 lot LONG + NEAR 100 lot SHORT = AÇIK POZİSYON! ✗
```
VİOP lot sayısı değişmez, SPOT otomatik artar → Hedge bozulur!

**Yeni Fonksiyonlar:**
```csharp
LoadSermayeArtirimi()        // JSON'dan yükle
UpdateSermayeArtirimi()      // Python scraper çalıştır
CheckSermayeArtirimi(sym)    // 0=Güvenli, 1=Bugün, 2=7 gün içinde
IsSermayeArtirimiBlocked(sym) // true=İşlem yapma
```

**Yeni Değişkenler:**
```csharp
var SRM_FILE = BASE_PATH + "srm.json";
var SRM_SCRAPER = BASE_PATH + "srm_scraper.py";
var sermayeArtirimlari = new Dictionary<string, DateTime>();
var lastSrmCheck = DateTime.MinValue;
var lastSrmCloseCheck = DateTime.MinValue;
```

**2. C# 5.0 UYUMLULUK DÜZELTMELERİ**
- Null-conditional operator (`?.`) kaldırıldı
- Explicit null check kullanıldı:
```csharp
// YANLIŞ (C# 6.0+):
var price = depth.Bids[0]?.Price ?? 0;

// DOĞRU (C# 5.0):
var price = depth.Bids != null && depth.Bids.Length > 0 && depth.Bids[0] != null 
    ? (decimal)depth.Bids[0].Price : 0m;
```

**3. FORWARD DECLARATION**
- `IsDividendDay`, `IsAfterOpeningVolatility`, `CheckSermayeArtirimi`, `IsSermayeArtirimiBlocked` fonksiyonları dosyanın başına taşındı
- Pair Trading döngüsünden önce tanımlı olması gerekiyor

---

## 5. RECOVERY SİSTEMİ

### Recovery Tetikleyiciler

| Olay | Recovery | Açıklama |
|------|----------|----------|
| Robot başlangıç | ✅ SyncAndRecoverPositions | İlk açılış |
| 11:50 | ✅ SyncAndRecoverPositions | Gün ortası |
| 17:35 | ✅ SyncAndRecoverPositions | Kapanış öncesi |
| `brokerConnected` geri geldi | ✅ SyncAndRecoverPositions | Hesap bağlantısı |
| `dataConnected` geri geldi | ❌ | Sadece fiyat akışı |

### Pair Recovery (viopOk Bağımlılığı)
```csharp
if (PAIR_ENABLED && viopOk && PAIR_DEFS.Count > 0) {
    // Pair recovery çalışır
}
```
- VIOP hesap null ise Pair recovery ATLANIR
- Bu doğru davranış: VIOP verisi yoksa recovery yapılamaz

### NF Recovery Orphan Akıllı Tamamlama
```
Broker'da Near var, Far yok (orphan):
  → Spread kontrolü (≥ %0.5)
  → Uygunsa Far aç, pozisyonu tamamla
  → Değilse Near kapat

Broker'da Far var, Near yok (orphan):
  → Spread kontrolü (≥ %0.5)
  → Uygunsa Near aç, pozisyonu tamamla
  → Değilse Far kapat
```

---

## 6. TEMETTÜ VE SERMAYE ARTIRIMI SİSTEMİ

### Temettü Dosyaları
- `D:\arbit\Temettu.json` - Temettü verileri
- `D:\arbit\Temettu_Scraper.py` - InfoYatırım PDF'den veri çeken script

### Sermaye Artırımı Dosyaları (v19.4)
- `D:\arbit\srm.json` - Sermaye artırımı verileri
- `D:\arbit\srm_scraper.py` - Piramit Menkul PDF'den veri çeken script

### Kontrol Saatleri

| Saat | İşlem |
|------|-------|
| 09:25 | Temettü verilerini güncelle |
| 09:25 | Sermaye artırımı verilerini güncelle |
| 10:10 | Sermaye artırımı olan pozisyonları kapat |

### Koruma Mantığı

| Durum | Giriş | Mevcut Pozisyon |
|-------|-------|-----------------|
| Bugün temettü | ⛔ Engelle | - |
| Bugün sermaye artırımı | ⛔ Engelle | 🚨 10:10'da kapat |
| 7 gün içinde sermaye artırımı | ⛔ Engelle | 🚨 10:10'da kapat |

---

## 7. ÖNEMLİ TEKNİK NOTLAR

### C# 5.0 Kısıtlamaları (IdealData)

**KULLANMA:**
```csharp
// Null-conditional (C# 6.0+)
var x = obj?.Property ?? default;

// String interpolation (C# 6.0+)
var s = $"Value: {x}";

// Nameof (C# 6.0+)
nameof(variable)
```

**KULLAN:**
```csharp
// Explicit null check
var x = obj != null ? obj.Property : default;

// String concatenation
var s = "Value: " + x.ToString();
```

### IdealData API Davranışları

**GrafikVerileriniOku:**
```csharp
// IdealData'da grafik verisi için önce grafiğin açık olması gerekir!
// Pair Trading sembolleri için günlük grafiklerini aç: TCELL, TTKOM, CIMSA, OYAKC
dynamic data = Sistem.GrafikVerileriniOku(sym, "G");  // G = Günlük
```

**API Rate Limiting:**
- IdealData API'leri hızlı çağrılınca NULL dönebilir
- Çözüm: 2 saniyelik cache wrapper fonksiyonları kullan

### Forward Declaration Gerekliliği
Fonksiyonlar kullanılmadan önce tanımlanmalı. Şu fonksiyonlar dosyanın başında (satır ~350):
- `IsDividendDay`
- `IsAfterOpeningVolatility`
- `CheckSermayeArtirimi`
- `IsSermayeArtirimiBlocked`

**NOT:** Bu fonksiyonlarda `Log()` çağrısı YOK çünkü Log henüz tanımlı değil. Log, çağrı noktalarında yazılır.

---

## 8. DOSYA YAPISI

### Çalışma Klasörü: `D:\arbit\`

| Dosya | Açıklama |
|-------|----------|
| `ArbCodeFormVersion_v19_4.txt` | C# robot kodu (~10,600 satır) |
| `gist_credentials.json` | Gist token ve ID |
| `settings.json` | Robot ayarları |
| `positions.json` | Açık pozisyonlar + orphanLegs + pairPos |
| `teminat.json` | Sembol teminat değerleri |
| `Temettu.json` | Temettü verileri |
| `Temettu_Scraper.py` | Temettü scraper |
| `srm.json` | Sermaye artırımı verileri (v19.4) |
| `srm_scraper.py` | Sermaye artırımı scraper (v19.4) |
| `logs/arb.log` | Log dosyası (1000 satır) |

---

## 9. KONTROL SAATLERİ

| Saat | Kontrol |
|------|---------|
| 09:20 | Teminat dosyası kontrolü |
| 09:25 | Temettü + Sermaye artırımı güncelleme |
| 09:56 | Derinlik cache warmup |
| 10:01 | Robot başlar |
| **10:05** | **Açılış volatilitesi biter, giriş serbest** |
| 10:05 | Orphan recovery, Pair trading başlar |
| **10:10** | **Sermaye artırımı pozisyon kapatma** |
| 11:50 | Pozisyon senkronizasyonu |
| 16:50 | T+2 negatif kontrolü |
| **17:59** | **Pair trading durur** (SPOT kapanışı öncesi) |
| 17:50 | Pozisyon senkronizasyonu |
| 18:09 | Robot durur |

---

## 10. KULLANICI TERCİHLERİ

- Uzun yanıtları dosya olarak paylaş
- Prompt sonunda `*` varsa kodu değiştirme, sadece analiz et
- `!` gönderilirse master dosyaları güncellemeleri ekleyerek son haline getir
- **Kod değişikliğinde:** Hangi fonksiyonların etkileneceğini önceden belirt
- TALIMAT dosyalarını her işlemde değil, sohbet sonunda toplu güncelle
- Kod çıktılarında sadece dosya linki ver, uzun kod blokları gösterme

---

## 11. HATA AYIKLAMA

### Sık Karşılaşılan Compiler Hataları

**CS1525 "Geçersiz ifade terimi '.'":**
- Sebep: Null-conditional operator (`?.`) kullanılmış
- Çözüm: Explicit null check'e dönüştür

**CS0841 "Yerel değişken bildirilmeden önce kullanılamaz":**
- Sebep: Fonksiyon kullanılmadan önce tanımlanmamış
- Çözüm: Forward declaration - fonksiyonu yukarı taşı

**CS0841 "Log" hatası:**
- Sebep: Forward declaration'daki fonksiyonda Log() çağrılmış
- Çözüm: Forward declaration'da log yok, çağrı noktasında logla

### Sık Karşılaşılan Runtime Sorunları

**"Yetersiz veri" hatası:**
- Sebep: Grafik penceresi açık değil
- Çözüm: TCELL, TTKOM, CIMSA, OYAKC günlük grafiklerini aç

**Recovery'de VIOP null:**
- Log: "VIOP hesap veya pozisyonlar null - NEAR recovery atlanıyor"
- Sebep: API bağlantı sorunu
- Etki: SN Near recovery + NF recovery + Pair recovery ATLANIR
- Çözüm: Bağlantı gelince otomatik recovery tetiklenir

---

## 12. CHANGELOG

### v19.4 (2026-02-01)
- **YENİ:** Sermaye artırımı koruma sistemi
  - srm.json + srm_scraper.py
  - Giriş engeli (bugün + 7 gün)
  - Otomatik pozisyon kapatma (10:10)
- **FIX:** C# 5.0 uyumluluk (null-conditional kaldırıldı)
- **FIX:** Forward declaration (fonksiyon sırası)

### v19.3 (2026-01-31)
- Temettü günü koruması (SN, NF, Pair)
- Açılış volatilitesi koruması (10:05 bekleme)
- Async Gist upload
- Güvenli dosya yazımı (FileShare.ReadWrite)
- NF açılış/kapanış ön kontrolü
- Beta sapma toleransı %15 → %10
- Pair Pending Completion sistemi (açılış + kapanış)
- LoadPositions güvenliği (used sıfırlama)

### v19.2 (2026-01-31)
- Rollover bug fix (pairContracts)
- Form thread timing fix
- Settings Form iki sütunlu layout
- Pair Z-Score dashboard gösterimi
- Pair saatleri 17:59'a uzatıldı

### v19.1 (2026-01-31)
- Kademe Dağılımı: PAIR_ALLOC_1/2/3 değişkenleri

### v19 (2026-01-30)
- Pair Trading Stratejisi

### v18 (2026-01-30)
- Erken Çıkış: Dinamik k formülü
- Orphan Recovery 10:05 Bekletme

---

## 13. SIK KULLANILAN KOD PATERNLERİ

### Null-Safe Derinlik Okuma (C# 5.0)
```csharp
var nearBid = nearDepth.Bids != null && nearDepth.Bids.Length > 0 && nearDepth.Bids[0] != null 
    ? (decimal)nearDepth.Bids[0].Price : 0m;
```

### Sembol Normalizasyonu
```csharp
// F_THYAO0226 -> THYAO
string normalized = symbol.ToUpper();
if (normalized.StartsWith("F_")) {
    normalized = normalized.Substring(2);
    if (normalized.Length > 4)
        normalized = normalized.Substring(0, normalized.Length - 4);
}
```

### Güvenli Dosya Yazımı
```csharp
using (var fs = new FileStream(path, FileMode.Create, FileAccess.Write, FileShare.ReadWrite)) {
    var bytes = Encoding.UTF8.GetBytes(content);
    fs.Write(bytes, 0, bytes.Length);
}
```

---

## 14. PYTHON SCRAPER BAĞIMLILIKLARI

### Temettü Scraper (Temettu_Scraper.py)
```bash
pip install requests beautifulsoup4 pdfplumber
```

### Sermaye Artırımı Scraper (srm_scraper.py)
```bash
pip install requests beautifulsoup4 pdfplumber
```

### Manuel Çalıştırma
```bash
# Temettü
python D:\arbit\Temettu_Scraper.py

# Sermaye artırımı
python D:\arbit\srm_scraper.py --force
```
