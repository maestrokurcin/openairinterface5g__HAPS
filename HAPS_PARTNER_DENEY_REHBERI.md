# HAPS Kanal Modeli — Partner için Deney Tekrarlama Rehberi

Bu doküman, **kendi HAPS kanal geliştirmesini** yapan bir partnerin, bizim
`HAPS_TEST_GUNLUGU.md`'de kayıtlı deneylerin **aynısını** kendi kodu üzerinde
koşabilmesi için hazırlanmıştır. Her deney için:

- **hangi fiziksel değişken** değiştiriliyor,
- **bizim referans implementasyonumuzda** hangi env var / config'e ne değer verdiğimiz,
- **partnerin kendi kodunda** neyi karşılık getirmesi gerektiği,
- **hangi metrikleri, nereden, nasıl** ölçmesi gerektiği

belirtilir. Sonunda bizim ölçtüğümüz değerler tablo hâlinde verilir ki partner
kendi sonuçlarını birebir karşılaştırabilsin.

İlgili dokümanlar: `HAPS_TEST_GUNLUGU.md` (bizim deney kayıtlarımız, tam detay),
`HAPS_CALISTIRMA_REHBERI.md` (koşum komutları), `HAPS_MIMARI.md` (kod mimarisi).

---

## 0. Ön koşullar — partnerin kurulumu bizimkiyle "eşdeğer" olmalı

Deneyler ancak aşağıdaki **sabit taban** her iki tarafta da aynıysa
karşılaştırılabilir. Bunlar deneyin parçası değil, deneyin **zemini**:

| Öğe | Değer | Neden sabit |
|---|---|---|
| Radyo | NR band 254 (S-bandı), DL ≈ 2.4886 GHz, µ=0 (15 kHz SCS), 25 PRB, TDD | NTN referans bandı; dar bant + µ=0 Msg3/PRACH zaten kalibre |
| NTN | Gerçek `ntn_Config_r17` + SIB19 broadcast, `cellSpecificKoffset_r17 = 1` | Açık-döngü TA ön-telafisi olmadan 20 km gecikme RA'yı bozuyor |
| Platform geometrisi | 20 km irtifa, 2 km loiter yarıçapı, ~100 km/h (varsayılan "mobil") | ITU HAPS aralığı; zenit altı ~78.7° yükseklik açısı |
| Kanal fiziği | TR 38.811 S6.6.2 temel yol kaybı (FSPL + gölge sönümleme + clutter) + kTB+NF gürültü tabanı | Deneylerin çoğu bu terimlerin birini süpürüyor |
| Trafik | **Yok** — bu makinede AMF/5GC bağlı değil, bağlantı RRC_CONNECTED'te kalıyor | MCS/goodput ölçülemiyor; metrikler bağlantı sağlığı + BLER ile sınırlı |
| Derleme | `ninja rfsimulator nr-softmodem nr-uesoftmodem` (üçü birlikte) | `librfsimulator.so` çalışma zamanında `dlopen` ile yükleniyor |
| Her koşuda | `MALLOC_ARENA_MAX=1` (env var) | Onsuz bu makinede `nr-softmodem` çöküyor |

> **Partner farklı bir taban kullanıyorsa** (örn. farklı bant, farklı PRB), bu
> tabloyu kendi değerleriyle bir kez sabitleyip dokümante etmeli; deney
> sonuçları ancak kendi tabanına göre okunur, bizimkiyle **mutlak** değil
> **eğilim** (trend) olarak karşılaştırılır.

---

## 1. Ortak koşum prosedürü — her deneyde aynı

1. **gNB'yi başlat**, ~10-12 sn bekle (rfsim sunucusu hazır olsun).
2. **UE'yi başlat.**
3. **Koşu süresi**:
   - Tam bağlantı testi (BLER, HARQ, kopma): **en az 90 sn** UE çalıştıktan sonra.
   - Sadece büyük-ölçek metrik (netgain, LOS/NLOS durumu, yağmur/O2I dB'si):
     **~15-25 sn** yeter (senkron beklemeye gerek yok, debug satırı için).
4. **Kaç koşu?**
   - **Stokastik / küçük-ölçekli** etki (LOS/NLOS çekimi, gölge sönümleme, hızlı
     fade): **en az 3, tercihen 5-8 koşu**, dağılımı raporla. Tek koşu yanıltıcı
     outlier verir (bkz. bizim Deney 4).
   - **Deterministik büyük-ölçek** etki (yağmur oranı, sabit yükseklik açısı,
     MIMO stream sayısı): tek koşu yeter.
5. **Rakamları logdan al, tahmin etme.** Aşağıdaki Bölüm 2'deki satırlardan
   birebir kopyala.
6. **Tek değişken.** Her deney tabana göre **yalnızca bir** şeyi değiştirir.
   İki şey değiştiyse bu iki ayrı deneydir.
7. **Her koşuyu koştuğu gün yaz** — hangi parametreyle koştuğun birkaç saat
   sonra hatırlanmıyor.

---

## 2. Ölçüm metrikleri — ne, nereden, nasıl

### 2.1 Birincil metrik: `netgain` (net kazanç, dB)

Bizde `HAPS_DEBUG_38811=1` env var'ı gNB/UE loguna saniyede bir şu satırı bastırıyor:

```
TR 38.811: scenario=suburban/rural band=S elev=78.7° state=LOS sigma_SF=1.14dB
  CL=0.00dB gas=0.039dB rain=0.000dB(0.0mm/h) scint=0.000dB o2i=0.000dB
  PLb=126.0dB FSPLref=122.63dB netgain=-5.7dB
```

- **`netgain`** = en-iyi-durum (zenit LOS) geometriye **göreli** net kanal
  kazancı. Referans (banliyö zenit LOS) ≈ **−6 dB**; her kötü geometri daha negatif.
- **Bu, senaryo karşılaştırmasının birincil metriğidir.** Çünkü UE'nin bildirdiği
  SINR, çalışan her senaryoda CQI tablo tavanına (~+40 dB) railleniyor — çalışan
  senaryolar arasında ayırt edici değil (nedeni: rfsim `path_loss_dB`'yi int16
  örnek kazancı olarak uyguluyor; kalibrasyon detayı `HAPS_GELISTIRME_GUNLUGU.md`
  Adım 40).
- **Partnerin kendi kodunda karşılığı**: kanal modelinin uyguladığı toplam
  büyük-ölçek kazanç/kaybı, sabit bir referans geometriye göre normalize edilmiş
  hâlde, her koşuda loglanmalı. Mutlak dB değeri implementasyona göre kayabilir;
  önemli olan **aynı referansa göreli** olması ve **koşu boyunca sabit** olması
  (büyük-ölçek terimler bağlantı başına bir kez çekilip donmalı — bkz. 2.4).

Ayrıca aynı satırdan: `state` (LOS/NLOS — bu koşuda hangisi çekildi), `elev`
(anlık yükseklik açısı), `sigma_SF`, `CL` (clutter, sadece NLOS), `gas`, `rain`,
`scint`, `o2i` — her terimin ayrı katkısı.

### 2.2 Bağlantı kilometre taşları (UE logu)

| Log satırı | Anlamı |
|---|---|
| `UE synchronized!` / `Starting sync detection` … `synch Failed` | PBCH/SSB kilidi kuruldu / kurulamadı |
| `Found RAR with the intended RAPID` | RA (rastgele erişim) Msg2 kabul edildi |
| `Generating RRCSetupComplete` | **RRC bağlantısı kuruldu = başarı** |
| gNB: `RA failed at state ...` | RA denemesi başarısız (kaç kez tekrarlandı?) |

**Bağlan / bağlanma** ikili sonucu her koşuda kaydedilir. Stokastik deneylerde
**bağlanma oranı** (kaç koşuda kaç) asıl metriktir.

### 2.3 gNB periyodik MAC istatistik dökümü (`dump_mac_stats`)

gNB birkaç saniyede bir şu bloğu basar:

```
UE RNTI 4a1c ... in-sync PH 45 dB ... average RSRP -84 ... average SINR 39.8 (32 meas)
UE 4a1c: dlsch_rounds 78/0/0/0, dlsch_errors 0, ... BLER 0.001 MCS (0) 0 ... goodput 0.00 Mbps
UE 4a1c: ulsch_rounds 765/0/0/0, ulsch_errors 0, ulsch_DTX 0, BLER 0.000 MCS (0) 0 ... SNR 17.2 (+2.2) dB ...
```

Bir deneyde bakılacak alanlar:

| Alan | Ne yansıtır | Deneyde nasıl kullanılır |
|---|---|---|
| `dlsch_rounds a/b/c/d` | HARQ tur dağılımı (1./2./3./4. iletim) | `b,c,d` > 0 = yeniden iletim = kötü kanal. Baz: `x/0/0/0` |
| `ulsch_rounds a/b/c/d` | UL HARQ turları | aynı |
| DL / UL `BLER` | Blok hata oranı | Sönümleme + SINR marjı. Mean **ve** max ayrı raporla |
| `average SINR` (dB) | UE-bildirimli DL SINR | **Sadece bağlan/bağlanma** göstergesi — çalışan senaryoda ~+40'a raillenir, ayırt edici değil |
| `average RSRP` (dBm) | UE-ölçülen referans sinyal gücü | Yol kaybı eğilimi |
| gNB `PUSCH SNR x (+y) dB` | UL alıcı SNR (hedef +y) | Güç kontrolü hedefe kilitliyor mu (kararlılık) |
| `in-sync` / `out-of-sync` | UL senkron durumu | `out-of-sync` sayısı = **kopma sayısı** |
| `dlsch_errors` / `ulsch_errors` | Kurtarılamamış blok | > 0 = HARQ toparlayamadı (ciddi) |

> **`goodput` / `MCS` anlamlı değil** — trafik yok (5GC bağlı değil). Partnerde
> 5GC varsa bunlar da metrik olur; yoksa bizimki gibi göz ardı edilir.

### 2.4 Kritik disiplin — büyük-ölçek terimler DONMUŞ olmalı

`is_los` (LOS/NLOS durumu), gölge sönümleme gerçekleşimi ve O2I giriş kaybı
**büyük-ölçekli** etkilerdir: sabit (hareketsiz) bir UE için bağlantı ömrü
boyunca **bir kez çekilip sabit kalmalı**. Bizde bu bir bug'dı (saniyede bir
yeniden çekiliyordu) — düzeltildi (`HAPS_GELISTIRME_GUNLUGU.md` Adım 39/41).

**Partner kontrolü**: `netgain` (veya eşdeğeri) tek bir koşu **içinde** tam
sabit olmalı. Saniyede bir zıplıyorsa, partnerin modeli aynı bug'a sahip
demektir ve deney sonuçları geçersizdir — önce bu düzeltilmeli. (Bizde
`HAPS_DEBUG_LOS_SWEEP=1` env var'ı, `is_los` çekiminin 200k örnekte TR 38.811
Tablo 6.6.1-1'i birebir ürettiğini doğrulayan bir öz-kontrol koşuyor — partner
kendi çekimini benzer şekilde doğrulamalı.)

Küçük-ölçekli sönümleme (NTN-TDL) ise **koşu-başına farklı** (RNG
`/dev/urandom`'dan) — bu yüzden stokastik deneylerde 3+ koşu.

---

## 3. Deney kataloğu

Her satırda: **fiziksel değişken** → **bizim knob'umuz + değeri** → **partnerin
kendi kodunda karşılığı** → **taban** → **beklenen yön**.

Taban senaryo (aksi belirtilmedikçe): banliyö/kırsal, zenit (yükseklik açısı
~78.7°), yağmur/O2I kapalı, UE 3 km/h, mobil platform.

### Deney 1 — Mekân senaryosu: banliyö → kentsel

| | |
|---|---|
| **Fiziksel değişken** | Clutter/gölgeleme ortamı (TR 38.811 senaryo sınıfı) |
| **Bizim knob** | Config çifti `..._38811.conf` → `..._38811_urban.conf` (kanal tipi `HAPS_MOBILE_38811` → `HAPS_MOBILE_38811_URBAN`) |
| **Partner karşılığı** | Kendi modelinde senaryo seçicisini `suburban/rural` → `urban` yap; `sigma_SF` (LOS) 0.72 → 4.0 dB, NLOS clutter tablosu farklı |
| **Ölç** | 3 koşu: netgain dağılımı, LOS/NLOS durumu, RRC, BLER, HARQ |
| **Beklenen** | Zenit'te sistematik kötüleşme YOK; tek fark netgain'in koşudan koşuya **daha geniş yayılması** (σ_SF 4 dB) |

### Deney 2 — Kentsel + düşük yükseklik açısı

| | |
|---|---|
| **Fiziksel değişken** | Terminalin platform izdüşümünden yatay uzaklığı → yükseklik açısı ↓, eğik mesafe ↑, NLOS olasılığı ↑ |
| **Bizim knob** | `HAPS_GROUND_OFFSET_M=35000` (env var, gNB+UE ikisine) + kentsel config |
| **Partner karşılığı** | Geometri hesabında yer istasyonunu platform-altı noktadan 35 km yatay kaydır → yükseklik açısı ~27°, eğik mesafe ~44 km |
| **Ölç** | ~90 sn koşu: netgain (LOS vs NLOS ayrı), gecikme, Doppler, RRC, senkron |
| **Beklenen** | Stokastik: LOS çekilirse netgain ~−11 (çalışır), NLOS çekilirse ~−40…−50 (link ölü). ~%50 bağlanma. Gecikme ~2×, Doppler ~10× |

### Deney 3 — Yağmur, S-bandı

| | |
|---|---|
| **Fiziksel değişken** | Yağmur oranı (ITU-R P.838 özgül sönümleme) |
| **Bizim knob** | `HAPS_RAIN_RATE_MM_H=25` (env var) |
| **Partner karşılığı** | P.838 k/α katsayılarıyla yağmur terimi, eğik yol `L_eff = h_rain / sin θ` |
| **Ölç** | Tek koşu: `rain` terimi (dB), netgain, RRC |
| **Beklenen** | S-bandında **ihmal edilebilir**: ~0.013 dB. Hiçbir gözlemlenebilir etki. (Ka-bandında birkaç dB olurdu) |

### Deney 4 — Hızlı UE (otoyol, 108 km/h)

| | |
|---|---|
| **Fiziksel değişken** | UE yerel-saçılma Doppler bandı `fd_local = v_UE · fc / c` → NTN-TDL sönümleme dekorelasyon hızı |
| **Bizim knob** | `HAPS_UE_SPEED_MPS=30` (env var; varsayılan 0.833 = 3 km/h) |
| **Partner karşılığı** | Küçük-ölçekli fading AR(1) katsayısını UE hızından türet; `fd_local` 7 → 249 Hz |
| **Ölç** | **En az 3-6 koşu**: UL BLER (mean **ve** max), UL HARQ turları, kopma, DL BLER |
| **Beklenen** | Çoğu koşu temiz (devasa SINR marjı). ~1/6 koşu sürekli ~%10 UL BLER episodu (onlarca HARQ yeniden iletimi), **asla kopma yok** |

### Deney 5 — Hızlı UE + kentsel (etkileşim testi)

| | |
|---|---|
| **Fiziksel değişken** | Deney 4 + Deney 1 birlikte |
| **Bizim knob** | `HAPS_UE_SPEED_MPS=30` + kentsel config |
| **Ölç** | 3 koşu: UL BLER, HARQ, netgain |
| **Beklenen** | Etkiler **birikmiyor** — kentsel + hızlı UE ≈ banliyö + hızlı UE. `fd_local` senaryodan bağımsız |

### Deney 6 — Alternatif NTN-TDL profili (A/C → B/D)

| | |
|---|---|
| **Fiziksel değişken** | Küçük-ölçekli fading tap yapısı / Ricean K-faktörü (TR 38.811 Tablo 6.9.2) |
| **Bizim knob** | `HAPS_TDL_USE_ALT_PROFILE=1` (env var) — LOS'ta fiilen TDL-C → TDL-D |
| **Partner karşılığı** | TDL tap gecikme/güç tablosunu A/C setinden B/D setine değiştir |
| **Ölç** | 3 + 2 koşu: UL/DL BLER, delay spread |
| **Beklenen** | **Anlamlı fark yok** (zenit LOS, geniş marj). DS her ikisinde CP'nin çok altında |

### Deney 7 — O2I bina girişi kaybı

| | |
|---|---|
| **Fiziksel değişken** | Outdoor→indoor geçiş kaybı (ITU-R P.2109) |
| **Bizim knob** | `HAPS_O2I_ENABLE=1` (env var; `HAPS_O2I_THERMAL=1` daha ağır bina sınıfı) |
| **Partner karşılığı** | P.2109 persentil-tabanlı giriş kaybı, bağlantı başına **bir kez** çekilip donmuş |
| **Ölç** | 3 koşu: `o2i` terimi (dB, sabit olmalı!), netgain, RRC |
| **Beklenen** | Link **kurulamıyor** (3/3) — medyan ~30 dB giriş kaybı netgain'i −37'ye indiriyor. Handheld + bina duvarı + 20 km HAPS = imkânsız |

### Deney 8 — Ara mesafe (~55°)

| | |
|---|---|
| **Fiziksel değişken** | Deney 2'nin daha ılımlı hâli |
| **Bizim knob** | `HAPS_GROUND_OFFSET_M=10000` (eğik mesafe ~22 km, açı ~55°) |
| **Ölç** | 3 koşu: netgain, LOS/NLOS her yön için, RRC |
| **Beklenen** | 2/3 temiz (netgain −7…−10), 1/3 tek-yönlü şanssız NLOS çekimi tüm bağlantıyı düşürüyor (DL/UL bağımsız çizim) |

### Deney 10 — UE arabada (50 km/h)

| | |
|---|---|
| **Bizim knob** | `HAPS_UE_SPEED_MPS=13.89` |
| **Ölç** | 6 koşu: UL BLER, HARQ, `fd_local` |
| **Beklenen** | Yürüyüşten **ayırt edilemez** (`fd_local` 115 Hz, hâlâ HARQ RTT'sine göre yavaş) |

### Deney 11 — Yoğun kentsel + düşük açı (en kötü durum)

| | |
|---|---|
| **Bizim knob** | `HAPS_GROUND_OFFSET_M=35000` + `..._dense_urban.conf` |
| **Ölç** | 8 koşu: netgain (LOS/NLOS), bağlanma oranı |
| **Beklenen** | **~0/8 bağlanır.** NLOS ~7/8 (`los_prob[dense_urban,27°]=0.398`), NLOS netgain −27…−65 dB (σ_SF 12.4 dB) |

### Deney 12/13/15 — netgain ↔ yükseklik açısı süpürmesi (3 senaryo)

| | |
|---|---|
| **Fiziksel değişken** | Yükseklik açısı (7 nokta) × senaryo |
| **Bizim knob** | `HAPS_GROUND_OFFSET_M` = 0 / 5000 / 10000 / 15000 / 20000 / 25000 / 35000 (→ ~79°/66°/55°/47°/40°/35°/27°), her senaryo için ayrı config |
| **Ölç** | Nokta başına 3-5 kısa koşu (~22 sn): `netgain` ve `elev`, LOS/NLOS. NLOS çekilenleri LOS gelene kadar tekrar koş |
| **Beklenen** | **LOS netgain her senaryoda `−6 + 20·log₁₀(sin θ)` eğrisini izler** (geometri senaryodan bağımsız). Senaryo farkı: (a) LOS σ_SF yayılımı (banliyö ±1, kentsel ±4, yoğun kentsel ±3 dB), (b) LOS olasılığının açıyla düşme hızı (banliyö 10°'ye kadar >%90, yoğun kentsel 55°'de yazı-tura) |

### Deney 14 — LOS olasılık tablosu doğrulaması

| | |
|---|---|
| **Bizim knob** | `HAPS_DEBUG_LOS_SWEEP=1` (env var) — tek seferlik 200k-çekim/hücre Monte-Carlo öz-kontrol |
| **Partner karşılığı** | Kendi `is_los` çekiminin TR 38.811 Tablo 6.6.1-1'i (27 hücre: 3 senaryo × 9 açı) ±0.01 içinde ürettiğini doğrulayan bir test |
| **Beklenen** | 27/27 hücre eşleşir. Eşleşmiyorsa çekim kodunda bug var → diğer tüm senaryo deneyleri geçersiz |

### Deney 16 — UE hareketsiz

| | |
|---|---|
| **Bizim knob** | `HAPS_UE_SPEED_MPS=0` |
| **Ölç** | 5 koşu: tap enerjisi (`HAPS_DEBUG_TDL=1`), UL BLER, RRC |
| **Beklenen** | Fade donuyor ama BLER sıfırlanmıyor (~%0.2, MCS-0 gürültü tabanı hataları). **Dikkat**: hız 0 iken AR(1) fading durumu geçerli bir gerçekleşimle tohumlanmalı, yoksa NLOS'ta tüm taplar 0 = ölü kanal (bizde bug'dı, Adım 43) |

### Deney 17 — O2I ısıl verimli bina

| | |
|---|---|
| **Bizim knob** | `HAPS_O2I_ENABLE=1 HAPS_O2I_THERMAL=1` |
| **Beklenen** | Giriş kaybı ~2× (traditional ~29 → thermally-efficient ~56 dB medyan). Link yine ölü |

### Deney 18 — Platform hareketi açık/kapalı

| | |
|---|---|
| **Fiziksel değişken** | Platform loiter Doppler'i + zamanla değişen eğik gecikme |
| **Bizim knob** | `HAPS_PLATFORM_SPEED_MPS=0 HAPS_LOITER_RADIUS_M=0` (env var — platformu zenitte dondurur, NTN-TDL fading + saniyelik yol-kaybı tazelemesi aynen kalır) |
| **Partner karşılığı** | Platform kinematiğinde loiter hızını ve yarıçapını 0 yap; küçük-ölçek fading ve yol kaybı hesabını **kapatma** (yalnızca hareketi kapat) |
| **Ölç** | 3+3 koşu: Doppler (Hz), gecikme kayması, RRC, BLER, HARQ |
| **Beklenen** | **Gözlemlenebilir fark yok.** Loiter Doppler'i ≤~230 Hz (bizim koşularda 0-7 Hz), gecikme kayması ns mertebesinde; NTN ön-telafisi hepsini yutuyor |

### Deney 19 — Yoğun kentsel, ZENİT (tam bağlantı testi)

| | |
|---|---|
| **Bizim knob** | `..._dense_urban.conf`, ground offset yok (zenit ~78.7°) |
| **Ölç** | 6 tam koşu: LOS/NLOS, netgain, RRC, kopma, BLER, HARQ |
| **Beklenen** | Zenit'te 6/6 LOS; senkron olan koşular ilk denemede bağlanır, 0 kopma, 0 retx. **Yoğun kentselin sorunu senaryo değil, düşük açı** |

### Deney 20 — Yağmur + düşük açı birlikte

| | |
|---|---|
| **Bizim knob** | `HAPS_GROUND_OFFSET_M=35000` + `HAPS_RAIN_RATE_MM_H` = 50 / 100 |
| **Ölç** | Kondisyon başına 3 kısa koşu: `rain` terimi (dB), `L_eff`, netgain (`HAPS_DEBUG_RAIN_INTERNAL=1` iç değerler) |
| **Beklenen** | **Birleşim etkisi yok** — S-bandında bağımsız, toplanır. 100 mm/h @ 27° → 0.032 dB. Sintilasyon S-bandında tam 0 (TR 38.811 <6 GHz kuralı) |

### Deney 21 — Yoğun kentsel NLOS oranının nicelendirilmesi

| | |
|---|---|
| **Fiziksel değişken** | Yükseklik açısı → LOS çekme oranı → bağlanma oranı |
| **Bizim knob** | `..._dense_urban.conf`, `HAPS_GROUND_OFFSET_M` = 0/5000/10000/15000/25000/35000, açı başına 10 kısa koşu + 55°'de 6 tam koşu |
| **Ölç** | Her koşuda: gNB logundan **UL kanal** LOS/NLOS + netgain, UE logundan **DL kanal** LOS/NLOS + netgain, RRC sonucu. Açı başına: %1-yön-LOS, %iki-yön-LOS, %bağlanma |
| **Beklenen** | Bağlanma **her zaman DL=LOS gerektiriyor** (UL sığ NLOS'a, netgain > −27, tolerans var). DL/UL çizimleri **bağımsız** → iki-yön-LOS ≈ (tek-yön)² → bağlanma uçurumu: ~65°+ %70, 55° %30, <47° genelde başarısız |

### Deney 22 — O2I ısıl verimli bina + düşük yükseklik açısı birlikte

| | |
|---|---|
| **Fiziksel değişken** | Bina girişi kaybı (ısıl-verimli sınıf) × yükseklik açısı |
| **Bizim knob** | `HAPS_GROUND_OFFSET_M=35000` (~27°) + `HAPS_O2I_ENABLE=1 HAPS_O2I_THERMAL=1`; 7 kısa + 3 tam koşu |
| **Partner karşılığı** | P.2109 giriş kaybınıza yükseklik-açısı terimini ekleyin: `Le = 0.212·|elev°|`, `mu1 = Lh + Le` (P.2109-0 eq 9–10). Bu olmadan bu deney anlamsız |
| **Ölç** | Koşu başına donmuş O2I çekimi (dB), UL/DL netgain, RRC |
| **Beklenen** | **Birleşim cezası yok.** O2I kaybı yükseklik açısıyla *artar* (dik geliş = daha çok bina malzemesi) → 27°'de zenite göre ~10 dB *daha az*. Bu, düşük açının ~7 dB FSPL cezasıyla neredeyse iptal oluyor. Link yine ölü (0/10) |

### Deney 23 — Çoklu-kullanıcı (Multi-UE)

| | |
|---|---|
| **Fiziksel değişken** | Aynı gNB'ye N eşzamanlı UE, her biri bağımsız kanal gerçekleşimiyle |
| **Bizim knob** | Yeni config çifti `..._multiue.conf` (UE başına ayrı `rfsimu_channel_ue<N>`/`_enB<N>`, 3× uicc/RU/cell, **3× `position<N>`**) + `nr-uesoftmodem --num-ues 3` |
| **Partner karşılığı** | Kanal modelinizi UE-örneği (bağlantı) başına ayrı instance olarak kurun — büyük/küçük ölçek çekimleri UE başına bağımsız olmalı. UE konumunu her örnek için ayrı okuyun (eksikse {0,0,0} → sahte ~6398 km TA → simülasyon kilitlenir) |
| **Ölç** | UE başına: bağlanma (evet/hayır), netgain, DL/UL BLER, HARQ, kopma; RA preamble çakışması; PRB paylaşımı |
| **Beklenen** | Zenit: 3/3 bağlanır, RA çekişmesiz (~8 çerçeve arayla, farklı preamble), bağlantı sağlığı tek-UE ile aynı, güç kontrolü UE başına bağımsız. Düşük açı: bağlanma UE'ler arası bağımsız (biri NLOS çekerse yalnız o düşer) |

---

## 4. Bizim ölçtüğümüz değerler (karşılaştırma için)

Partner kendi sonuçlarını bu tabloyla karşılaştırır. **Mutlak dB'ler
implementasyona göre kayabilir**; asıl kontrol edilecek şey **eğilimin yönü ve
büyüklük mertebesi**.

### 4.1 Taban senaryo (banliyö, zenit, mobil, yağmur/O2I yok, UE 3 km/h)

| Metrik | Değer |
|---|---|
| netgain | ≈ −5.7 dB (LOS, donmuş) |
| RA / RRC | ilk denemede başarılı |
| Kopma (~90 sn) | 0 |
| DL / UL HARQ turları | `78/0/0/0` · `765/0/0/0` (0 yeniden iletim) |
| DL / UL BLER | ≈ 0 / 0 |
| gNB PUSCH SNR | ~17.2 dB (hedef +2.2), kararlı |
| UE-bildirimli DL SINR | ~+39.8 dB (CQI tavanı — ayırt edici değil) |
| Tek-yön gecikme | ~0.067 ms |
| Doppler (SAT→UE) | ~21 Hz |

### 4.2 Deney sonuçları özeti

| Deney | Değişken | Sonuç (bizim ölçüm) |
|---|---|---|
| 1 | banliyö→kentsel (zenit) | netgain −4.3…−7.0 (σ_SF geniş); temiz bağlanır; sistematik kötüleşme yok |
| 2 | kentsel + 27° | LOS: netgain ~−11, bağlanır · NLOS: ~−40…−50, ölü · ~%50 bağlanma |
| 3 | yağmur 25 mm/h (S) | rain = 0.013 dB · etki yok |
| 4 | UE 108 km/h | çoğu koşu temiz · ~1/6 koşu ~%10 UL BLER episodu · 0 kopma |
| 5 | 108 km/h + kentsel | etkiler birikmiyor · banliyö+hızlı ile aynı |
| 6 | TDL A/C→B/D | anlamlı fark yok |
| 7 | O2I açık | o2i medyan ~30 dB · netgain ~−37 · 3/3 bağlanamıyor |
| 8 | ara mesafe ~55° | 2/3 temiz (netgain −7…−10) · 1/3 tek-yön NLOS → düşüş |
| 10 | UE 50 km/h | yürüyüşten ayırt edilemez (`fd_local` 115 Hz) |
| 11 | yoğun kentsel + 27° | ~0/8 bağlanır · NLOS 7/8 · NLOS netgain −27…−65 |
| 12/13/15 | açı süpürmesi ×3 senaryo | LOS netgain = `−6 + 20·log₁₀(sin θ)` her senaryoda · fark σ_SF ve LOS-olasılık eğiminde |
| 14 | LOS tablo doğrulama | 27/27 hücre ±0.002 |
| 16 | UE hız 0 | fade donuyor · BLER ~%0.2 (sıfırlanmıyor) |
| 17 | O2I ısıl verimli | giriş kaybı ~2× (~29→~56 dB) · link yine ölü |
| 18 | platform hareketi kapalı | fark yok · loiter Doppler 0-7 Hz · gecikme kayması ns |
| 19 | yoğun kentsel zenit | 6/6 LOS · senkron olunca temiz bağlanır, 0 kopma |
| 20 | yağmur + düşük açı | bağımsız toplanır · 100 mm/h @ 27° = 0.032 dB |
| 21 | yoğun kentsel NLOS oranı | bağlanma DL-LOS'a zorunlu · DL/UL bağımsız · uçurum: 65°+ %70, 55° %30, <47° ~%10 |
| 22 | O2I ısıl verimli + düşük açı | birleşim cezası yok · O2I kaybı açıyla artar (`Le=0.212·elev`) → 27°'de ~10 dB daha az, FSPL cezasıyla iptal · 0/10 bağlanır |
| 23 | Multi-UE (3 UE, `--num-ues 3`) | zenit 3/3 bağlanır · RA çekişmesiz · bağlantı sağlığı tek-UE ile aynı · düşük açıda bağlanma UE-bağımsız (2/3) · ön koşul: `position<N>` blokları |

### 4.3 Yükseklik açısı — LOS netgain teorik eğrisi (tüm senaryolar için ortak)

| θ | netgain | θ | netgain |
|---|---|---|---|
| 90° | −6.0 | 40° | −9.8 |
| 80° | −6.1 | 30° | −12.0 |
| 70° | −6.5 | 20° | −15.3 |
| 60° | −7.2 | 10° | −21.2 |
| 50° | −8.3 | | |

### 4.4 TR 38.811 Tablo 6.6.1-1 — LOS olasılığı (çekim doğrulaması için)

Açılar 10°…90° (10° adım):

| Senaryo | LOS olasılığı |
|---|---|
| Banliyö/kırsal | 0.782, 0.869, 0.919, 0.929, 0.935, 0.940, 0.949, 0.952, 0.998 |
| Kentsel | 0.246, 0.386, 0.493, 0.613, 0.726, 0.805, 0.919, 0.968, 0.992 |
| Yoğun kentsel | 0.282, 0.331, 0.398, 0.468, 0.537, 0.612, 0.738, 0.820, 0.981 |

---

## 5. Partnerin her deney için dolduracağı kayıt şablonu

```
### Deney N — <başlık>

- Tarih:
- Taban: <partnerin taban senaryosu>
- Değişen tek şey: <fiziksel değişken> = <değer>
  (bizim referans: <env var / config>; partner implementasyonu: <ne yapıldı>)
- Koşu sayısı ve süre:

Ölçülen:
| Koşu | LOS/NLOS (UL / DL) | netgain (eşdeğeri) | senkron | RRC | kopma | DL/UL HARQ | DL/UL BLER | avg SINR |
|------|-------------------|--------------------|---------|-----|-------|------------|-----------|----------|

Bizim sonucumuzla karşılaştırma:
- Eğilim yönü aynı mı? (evet/hayır + açıklama)
- Büyüklük mertebesi tutuyor mu?
- Sapma varsa muhtemel nedeni (implementasyon farkı / kalibrasyon / bug):

Sonuç: ✅ doğrulandı / ⚪ çürütüldü / ⚠️ bug bulundu
```

---

## 6. Özet — partnere kısa yol

1. Tabanı sabitle (Bölüm 0), `netgain` eşdeğerini logla ve **koşu içinde sabit
   olduğunu** doğrula (Bölüm 2.4).
2. `is_los` çekimini TR 38.811 tablosuna karşı doğrula (Deney 14).
3. Deney 1 → 21'i sırayla koş, her birinde **tek değişken**, stokastik olanlar
   için **3+ koşu**.
4. Her koşuda şu 6 şeyi kaydet: **LOS/NLOS (UL ve DL ayrı)**, **netgain**,
   **senkron oldu mu**, **RRC bağlandı mı**, **kopma sayısı**, **DL/UL BLER +
   HARQ turları**.
5. Bölüm 4'teki bizim sonuçlarımızla **eğilim** olarak karşılaştır, Bölüm 5
   şablonuyla dokümante et.
