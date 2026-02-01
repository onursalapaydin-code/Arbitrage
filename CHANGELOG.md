# BIST Arbitraj Robot - Changelog

## v19.4 (2026-02-01)
- **YENİ**: Sermaye artırımı koruma sistemi
  - srm.json dosyasından sermaye artırımı takvimi okuma
  - srm_scraper.py ile Piramit Menkul PDF'den veri çekme
  - Bugün veya 7 gün içinde sermaye artırımı varsa SN/NF/Pair giriş engeli
  - 09:25'te otomatik güncelleme (temettü ile aynı zamanda)
  - Robot başlangıcında veri yükleme

## v19.3 (2026-01-31)
- **BUG FIX**: dailyPairPnl ve totalClosedPair günlük sıfırlama eklendi
- **BUG FIX**: Pair Trading döngüsüne marginStatus < 2 kontrolü eklendi
- **TEMİZLİK**: Kullanılmayan değişkenler kaldırıldı (spreadRetrySymbols, lastSpreadRetry, SPREAD_RETRY_INTERVAL_MIN, MAX_STALE_DATA_MINUTES)
- **YENİ**: Zengin başlangıç formu (ayarlar özeti + pozisyon sayısı + strateji durumu)
  - Robot başlamadan önce tüm ayarları görebilme
  - "Ayarlar" butonu ile settings.json düzenleyebilme
  - Değişiklikler kaydedilince özet otomatik güncellenir
  - ShowSettingsForm artık mainForm olmadan da çalışır
- **YENİ**: IsPairLimitBlocked fonksiyonu - Pair Trading taban/tavan kontrolü
  - Giriş ve çıkışta taban/tavan kontrolü
  - Tavanda BUY, tabanda SELL engellenir
  - Orphan leg riski azaltıldı
- **YENİ**: OpenPair/ClosePair TAM DOLUM kontrolü
  - İlk bacak kısmi dolarsa geri alınır
  - İkinci bacak kısmi dolarsa her iki bacak geri alınır
  - ClosePair'de kısmi kapanış desteği (kalan pozisyon güncellenir)
  - Dengesiz pozisyon riski ortadan kalktı
- **YENİ**: NF RECOVERY mekanizması (OTOMATİK DÜZELTME)
  - Near LONG ve Far SHORT pozisyonları ayrı takip edilir
  - nfPos ile broker karşılaştırılır
  - Broker'da yok/bizde var → nfPos'tan silinir
  - Broker'da var/bizde yok: Her iki bacak → ekle, Orphan → spread uygunsa tamamla değilse kapat
  - PAIR_BLOCKED sembolleri NF recovery'den hariç tutulur
- **YENİ**: PAIR RECOVERY mekanizması (OTOMATİK DÜZELTME)
  - Broker'da var/bizde yok → pairPos'a eklenir + oran düzeltilir
  - Broker'da yok/bizde var → pairPos'tan silinir
  - Orphan (tek taraf) → Eksik bacak beta oranına göre açılır
  - Oran hatası (sapma > PAIR_RATIO_TOLERANCE) → Alış/satış ile beta oranına getirilir
  - Miktar uyumsuzluğu → Broker'a göre güncellenir + oran düzeltilir
  - Bizdeki oran doğruysa bizim miktarlarımıza göre düzeltir
- **YENİ**: AÇILIŞ VOLATİLİTESİ KORUMASI (10:05 bekleme)
  - Tüm stratejiler (SN, NF, Pair) için 10:05'e kadar yeni giriş yapılmaz
  - ENTRY_WAIT_MINUTES değişkeni (varsayılan 5 dk)
  - IsAfterOpeningVolatility() fonksiyonu
- **YENİ**: TEMETTÜ GÜNÜ KORUMASI (SN, NF, Pair)
  - Ex-dividend günü ilgili sembole giriş yapılmaz
  - IsDividendDay(sym) fonksiyonu
  - SN: Spot düşerken Near henüz düzelmemiş olabilir
  - NF: BB verisi (5dk grafik) henüz düzeltilmemiş olabilir
  - Pair: Z-Score anlık spot fiyatla hesaplanıyor, temettü düşmüş fiyat yanlış sinyal verir
- **YENİ**: ASYNC GIST UPLOAD
  - UpdateDashboard artık ayrı thread'de çalışır
  - MainLoop GitHub API timeout'undan etkilenmez
  - gistUploadInProgress flag ile çift upload önlenir
  - Veri hazırlama main thread'de, HTTP isteği arka planda
- **YENİ**: GÜVENLİ DOSYA YAZIMI (FileShare.ReadWrite)
  - Log ve SafeWrite artık diğer process'leri bloklamaz
  - Antivirus, yedekleme yazılımı, Notepad eş zamanlı erişebilir
  - SafeWrite'da 3 deneme retry mekanizması
  - Log yazma hatası robotu durdurmaz (try-catch içinde)
- **YENİ**: NF AÇILIŞ/KAPANIŞ ÖN KONTROLÜ
  - OpenNF: Far SAT öncesi Near Ask derinliği kontrol edilir
  - CloseNF: Far AL öncesi Near Bid derinliği kontrol edilir
  - Near likiditesi yoksa işlem başlamaz
  - Far orphan riski azaltıldı (geri alma/satma maliyeti önlendi)
  - Vade günü (EXPIRY) kapanışında kontrol atlanır (zorla kapat)
- **İYİLEŞTİRME**: PAIR BETA SAPMA TOLERANSI
  - PAIR_RATIO_TOLERANCE değişkeni eklendi (ayarlanabilir)
  - %15 → %10'a düşürüldü (daha sıkı kontrol)
  - Kaldıraçlı pozisyonlarda gereksiz teminat kullanımı önlenir
- **YENİ**: PAIR PENDING COMPLETION SİSTEMİ
  - OpenPair: İkinci bacak dolmadığında hemen kapatmak yerine beklemeye alınır
  - ClosePair: İkinci bacak (B) dolmadığında beklemeye alınır (A zaten kapatılmış)
  - PAIR_PENDING_WAIT_SEC (30sn) sonra tekrar denenir
  - PAIR_PENDING_MAX_RETRY (2) deneme hakkı
  - Açılışta: Z-Score hala uygunsa yeni fiyattan emir gönderilir
  - Kapanışta: Derinlik varsa B bacağı tekrar denenir
  - Max retry aşıldığında: Açılışta geri kapatılır, Kapanışta Recovery'ye bırakılır
  - pairPendingCompletion dictionary'si ile takip edilir (tier=-1 kapanış modu)
  - CheckPairPending() fonksiyonu ana döngüde çağrılır

## v19.2 (2026-01-31)
- **BUG FIX**: Pair Trading Rollover düzeltildi
  - pairContracts dictionary eklendi - kontrat takibi
  - OpenPair artık contractOffset parametresi alıyor (0=Near, 1=Far)
  - Rollover'da Far kontrat (offset=1) kullanılıyor
  - ClosePair artık pairContracts'tan doğru kontratı alıyor
  - positions.json'a pair kontratları eklendi (ca, cb alanları)
  - Backward compatible: Eski array formatı da okunabiliyor

## v19.1 (2026-01-31)
- **YENİ**: Kademe bütçe dağılımı değişken olarak (PAIR_ALLOC_1/2/3)
  - Varsayılan: 30/30/40 (%)
  - Settings Form'dan ayarlanabilir
  - %100 validasyonu ile kaydetme
- Settings Form'a kademe dağılımı satırı eklendi (Dağılım %: K1/K2/K3)
- Dashboard'a kademe dağılımı gösterimi eklendi
- tierBudgetPct artık hardcoded değil, değişkenlerden hesaplanıyor

## v19 (2026-01-30)
- **YENİ**: Pair Trading Stratejisi (VIOP Near-Near)
  - CIMSA-OYAKC (β=2.03) ve TCELL-TTKOM (β=1.80) çiftleri
  - Z-Score bazlı kademeli giriş (2.0, 2.5, 3.0 eşikleri)
  - Ayrı bütçe: PAIR_BUDGET = 100,000 TL (SN/NF'den izole)
  - Kontrollü GIE: Az likit bacak önce, 3 deneme ile ikinci bacak
  - Günlük grafik verisinden spread ve Z-Score hesaplama
  - PAIR_BLOCKED: 4 sembol SN/NF'den hariç tutuldu
- **YENİ**: pairPos dictionary, OpenPair(), ClosePair(), CheckPairSignals()
- **YENİ**: LoadPairData(), UpdatePairZScores() fonksiyonları
- positions.json ve stats.json'a pair pozisyonları eklendi
- Settings'e pair parametreleri eklendi
- SN/NF bütçe hesaplamasında PAIR_BUDGET ayrıldı

## v18 (2026-01-30)
- RecoverMissingLeg: orphanLegs kaydında giriş fiyatı snPos'tan alınıyor
- **YENİ**: Temettü yönetim sistemi (Temettu.json)
  - Robot başlangıcında ve 09:25'te otomatik güncelleme
  - Temettu_Scraper.py ile InfoYatırım PDF'den veri çekme
  - NF stratejisinde Near-Far arası temettü kontrolü
  - GetDividendsBetween() - İki vade arası temettü hesaplama
- **YENİ**: SN vade son günü (daysN=0) güvenlik önlemleri
  - Yeni pozisyon açılmıyor
  - Büyütme/Replacement yapılmıyor
  - Kapanış/Rollover sadece 12:00 sonrası (yarım gün vadeler için)
- **YENİ**: NF vade son günü (daysN=0) 12:00 kontrolü eklendi
- **YENİ**: NF kontrat kodları saklanıyor (vade geçişinde doğru kontratların kullanılması için)
  - nfContracts dictionary eklendi
  - positions.json'da nearC/farC alanları eklendi (backward compatible)
- **BUG FIX**: GetContract vade geçişi düzeltildi (vade tarihinden sonra otomatik sonraki aya geçiş)
- **BUG FIX**: Settings.json kaydetme - saat/dakika int.Parse ile (00 → 0 JSON hatası düzeltildi)
- **BUG FIX**: Cache path kaydetme - TrimEnd ile trailing backslash normalize
- **BUG FIX**: Sıfıra bölünme korumaları eklendi:
  - expectedPnl sıfır kontrolü (erken çıkış log)
  - posSpotValue sıfır kontrolü (T+2 kapatma)
  - Temettü parse sonsuz döngü önleme (ilerleme kontrolü)
- Tüm v18 özellikleri doğrulandı ve test edildi:
  - Erken çıkış dinamik k formülü (EARLY_EXIT_K)
  - Orphan 10:05 bekletme sistemi (CheckPendingOrders + RecoverMissingLeg)
  - MainLoop'ta TİP 1/TİP 2 orphan karar mekanizması
  - BB93+ pozisyon izolasyonu
  - NF devre kesici (INDEX_CIRCUIT_PCT, SPOT_CIRCUIT_PCT)

## v17.10 (2026-01-29)
- **YENİ**: Near-Far (NF) Takvim Spread Stratejisi
  - Near AL + Far SAT (spread BB >= 99 giriş)
  - BB <= 27 VEYA binde 3 kar çıkış
  - Maksimum 1 NF pozisyonu (şimdilik)
  - Çalışma saatleri: 10:00-12:00
  - Ayrı bütçe: NF_BUDGET = en büyük teminat × 2
  - Çakışma kontrolü: SN pozisyonu varsa NF açılamaz (aynı sembol)
- VIOP_RESERVE = en büyük teminat × 4 + 5000 (SN için 2x + NF için 2x)
- **YENİ**: nfPos dictionary, OpenNF(), CloseNF(), CheckNFClose()
- Form'a NF ayarları eklendi (aktif/pasif, max poz, BB eşikleri, kar çıkış)
- stats.json ve positions.json'a NF pozisyonları eklendi
- WarmupDepthCache ve LoadTavanTaban'a Far derinlik/tavan/taban eklendi
- **BUG FIX**: BIST kapalıyken (18:00+) SN açılış/kapanış/rollover engellendi
  - BIST 18:00'da kapanıyor, VİOP 18:10'da
  - SN işlemleri Spot gerektirir, BIST açık olmalı
  - isBistOpen kontrolü eklendi (10:01-18:00)
- **BUG FIX**: NF_BUDGET 2x → 3x (açılış 2 + kapanış GİE 1)
- VIOP_RESERVE 4x → 5x (SN 2x + NF 3x)
- **İYİLEŞTİRME**: NF_BUDGET ve VIOP_RESERVE artık dinamik (2n+1 formülü)
  - NF_BUDGET = maxTeminat × (NF_MAX_POSITIONS × 2 + 1)
  - VIOP_RESERVE = maxTeminat × (2 + 2n+1) + 5000
  - n=1 → 3x, n=2 → 5x, n=3 → 7x
- **YENİ**: orphanLegs dictionary - yarım kalan pozisyonları takip
  - Near satıldı ama Spot alınamadı → orphanLegs'e kaydet
  - Recovery spread kontrolünde açılış fiyatını kullanır
  - Pending order veya recovery tamamlayınca orphanLegs'ten siler
  - positions.json'a kaydedilir/yüklenir
- **İYİLEŞTİRME**: Pending order manuel iptal kontrolü
  - BekleyenEmirler'de yok + GerceklesenEmirler'de yok → Manuel iptal
  - pendingOrders'tan temizlenir, bütçe serbest bırakılır
  - orphanLegs Recovery'ye bırakılır (pozisyon hala açık olabilir)
- **İYİLEŞTİRME**: Recovery'de orphanLegs manuel tamamlama kontrolü
  - orphanLegs'te kayıt var ama broker'da pozisyon yok → Manuel tamamlanmış
  - orphanLegs'ten temizlenir
- **YENİ**: Erken kapanış dinamik k formülü (eski skor sistemi yerine)
  - earnedTarget = expectedPnl × elapsedRatio × (1 + k)
  - k = EARLY_EXIT_K × remainingRatio (vade sonuna yaklaştıkça k düşer)
  - closePnl >= earnedTarget VE BB5 < BB_EXIT_HELPER → KAPAT
  - Avantaj: Vade sonuna yaklaşınca imkansız eşikler yok, her zaman çıkış mümkün
  - Yeni parametre: EARLY_EXIT_K (0.20), BB_EXIT_HELPER (30)
- **İYİLEŞTİRME**: NF stratejisine devre kesici kontrolü eklendi
  - Endeks ≥%5 düşüş: Yeni NF açılışı durdurulur (kar alma devam eder)
  - Spot ≥%8 düşüş: O hisse için yeni NF açılışı durdurulur (kar alma devam eder)
  - Devre kesicide spread daralabilir, kar alma fırsatı kaçırılmasın

## v17.9 (2026-01-28)
- **YENİ**: ReadDepth() merkezi derinlik okuma fonksiyonu
  - Tüm derinlik okumaları tek fonksiyondan yapılıyor
  - 1. kademe 0 ise 3 kez 100ms arayla retry
  - 3 deneme sonunda 0 ise "❌ SEMBOL BID/ASK derinlik yok (3 deneme)" logu
  - Yüzeysel veri (AlisLot/SatisLot) KULLANILMIYOR - 26ms gecikme riski
- **GÜNCELLENDİ**: OpenSN, AddToSN, CloseSN, PartialCloseSN, RolloverSN → ReadDepth kullanıyor
- **DÜZELTME**: Rezerv moduna girmeden önce "kapatılabilir pozisyon var mı" kontrolü
  - Yeni pozisyon açmadan önce threshold geçen en az 1 pozisyon olmalı
  - Aksi halde rezerv açık kalıyordu (kritik bug)

## v17.8 (2026-01-28)
- **KALDIRILDI**: Günlük BB kontrolü (SN_BB_ENTRY) - analiz sonucu işe yaramıyor
  - BB her zaman %80 altında kaldığı için filtre hiç çalışmıyordu
  - skipFilters (DTE < 15) mantığı da artık gereksiz
- Form'dan "BB Entry %" alanı kaldırıldı
- Config'den sn_bb_entry parametresi kaldırıldı
- 5dk BB (BB93 stratejisi) gün içi için aktif kalmaya devam ediyor

## v17.7 (2026-01-28)
- Spread cache KALDIRILDI - artık grafik verisinden hesaplanıyor
- **YENİ**: LoadSpreadBBFromChart() - 5dk grafik verisinden BB pozisyonu hesaplar
- snBB → snBBPos: Artık sadece son BB değeri saklanıyor (0-100 arası)
- Her 5 dakikada bir tüm semboller için BB güncelleniyor
- spread_cache.json artık kullanılmıyor

## v17.5 (2026-01-28)
- İkinci recovery saati 17:50 → 17:35 (spread analizi sonucu: 17:40'ta genişleme başlıyor)
- **REZERV AÇILIŞ/BÜYÜTME**: Near VE Spot derinlik kontrolü eklendi (2 kademe)
- Miktar artık her iki tarafın derinliğine göre ayarlanıyor (min(Near, Spot))
- Spot derinlik yoksa 3 deneme retry (emir göndermeden önce)
- Spot kısmi dolumda fazla Near geri alınıyor (ExecGuar zaten 2x derinliğe emir atıyor)
- ATR filtreleri tamamen kaldırıldı (NormATR, SpotATR) - saat analizi ile değiştirilecek
- **İYİLEŞTİRME**: GetDays artık YüzeyselVeri.DaysToExpiry kullanıyor (VİOP'un resmi verisi)
- **İYİLEŞTİRME**: GetHours da DaysToExpiry'den hesaplanıyor
- Fallback: YüzeyselVeri alınamazsa eski manuel hesaplama kullanılır
- **YENİ**: Rezerv modunda (yeni açılış + büyütme) anlık zarar kontrolü eklendi
  - Kapatılacak pozisyon zararda ise, yeni işlemin beklenen karı bu zararı karşılamalı
  - Formül: expectedPnl >= |worstCurrentPnl| × (1 + REPLACEMENT_THRESHOLD)
  - Pozisyon kârda ise kontrol atlanır
  - Hem yeni pozisyon açılışı hem de büyütme için geçerli

## v17.3
- **YENİ**: SPOT_BUDGET_RESERVE = en yüksek fiyatlı spot × 50 lot
- **YENİ**: spotBalance hesaplamasından SPOT_BUDGET_RESERVE düşülüyor
- **MANTIK**: VIOP'ta VIOP_RESERVE nasıl düşülüyorsa, SPOT'ta da SPOT_BUDGET_RESERVE düşülür
- **AMAÇ**: Her zaman en az 1 pozisyon açabilme garantisi (50 lot = 0.5 Near lot karşılığı)

## v17.2
- **DEĞİŞİKLİK**: "Önce kapat sonra aç" modu tamamen kaldırıldı (spread kaçırma riski)
- **DEĞİŞİKLİK**: Replacement/büyütme için bütçe yetersizse:
  - Rezerv yeterliyse → Rezerv modu (önce aç sonra kapat)
  - Rezerv yetersizse → PAS GEÇ (işlem yapılmaz)
- **MANTIK**: Rezervler yetmiyorsa zaten sistem düzgün çalışmıyor demektir

## v17.1
- **YENİ**: SPOT_RESERVE dinamik hesaplama (T+2 - 5000 TL = kredi olarak kullanılabilir)
- **YENİ**: Rezerv modunda hem VIOP hem SPOT eksikse, her ikisi için de rezerv kullanılabiliyor
- **İYİLEŞTİRME**: AdjustT2Balance içinde SPOT_RESERVE otomatik güncelleniyor

## v17.0
- **BUG FIX**: Robot başlarken eski log dosyası siliniyor (temiz başlangıç)
- **BUG FIX**: Veri bağlantısı kopunca: hemen → 1dk sonra → saatte bir Gist güncelleme
- **BUG FIX**: Veri bağlantısı kopukken de broker bağlantı kontrolü yapılıyor
- **BUG FIX**: Dashboard'a spot_bal olarak spotBudgetTotal gönderiliyor
- **BUG FIX**: Başlangıçta dataConnected gerçek değerle set ediliyor
- **BUG FIX**: expectedPnl 0 ise positions.json yüklenirken yeniden hesaplanıyor
- **BUG FIX**: Replacement/büyütme için bütçe sınırda threshold kontrolü eklendi
- **BUG FIX**: Replacement döngüsünde her pozisyon sonrası spread kontrolü (bozulursa dur)
- **BUG FIX**: Dashboard currentPnl hesabı DayanakSatAl ile düzeltildi
- **YENİ**: Pending orders için bütçe ayrılıyor (SPOT ve VIOP)
- **YENİ**: Pending order olan sembol için OpenSN/AddToSN/CloseSN engelleniyor
- **YENİ**: Replacement'da pending order olan semboller atlanıyor
- **YENİ**: Büyütmede Near Ask derinlik kontrolü (ilk 3 kademe >= pozisyon x 5)
- **YENİ**: Bütçe sınırda iken (boşluk < 2x pozisyon) threshold'a SN_TOTAL_COST ekleniyor
- **YENİ**: Teminat kaydedilirken sembol listeleri D:\ideal\SembolListeleri'ne yazılıyor
- **YENİ**: VIOP_RESERVE = maxTeminat × 2 + 5000 (önce aç sonra kapat için 2x gerekli)

## v16.30
- **BUG FIX**: CACHE_PATH artık LOG_PATH ve STATS_FILE tarafından kullanılıyor
- RAMDisk desteği düzgün çalışıyor: CACHE_PATH=Z:\ yapınca log ve stats Z:'ye yazılır

## v16.29
- **BUG FIX**: entryTicks p[7]'ye taşındı (gün içi kar ve erken kar düzgün çalışıyor)
- **BUG FIX**: BuildStatsJson'da spotEntry sıfır kontrolü eklendi
- **BUG FIX**: CheckSNClose'da spotValue sıfır kontrolü eklendi (bölme hatası önlendi)
- **BUG FIX**: CloseSN'de nQty sıfır kontrolü eklendi (kısmi kapanışta bölme hatası önlendi)
- **BUG FIX**: HandleMarginCritical'da teminatToplam sıfır kontrolü eklendi
- **BUG FIX**: CheckFilledQty lastOrderNo yerine son 10sn içindeki emirlere bakıyor
- **İYİLEŞTİRME**: UpdateDashboard her zaman tam pozisyon bilgisi gönderiyor (fullStats kaldırıldı)
- **İYİLEŞTİRME**: STOP anında ve bağlantı geri geldiğinde de tam stats gönderiliyor
- **İYİLEŞTİRME**: Recovery önce lokal stats.json'dan pozisyon bilgisi almayı deniyor
- **YENİ**: Cache path ayarı (RAMDisk desteği - stats.json ve loglar için Z:\ gibi)
- **YENİ**: Log dosyası sabit isim (arb.log), 1000 satır limit, truncate (RAMDisk için optimize)
- **YENİ**: VIOP_RESERVE = 25000 TL (Near kapatma için geçici teminat rezervi)
- **Veri yapısı**: [spotQty, nearQty, spotEntry, nearEntry, entrySpread, expectedPnl, addCount, entryTicks]
- positions.json ve stats.json'a entryTicks eklendi
- Geriye uyumlu: Eski 7 elemanlı dosyalar okunabiliyor (entryTicks=şimdi)
