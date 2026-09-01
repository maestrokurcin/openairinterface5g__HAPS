# HAPS Kanal Modeli — Test Günlüğü (RF-Sim Deneyleri)

Bu doküman, HAPS kanal bloğu **tasarımı bittikten sonra** rfsimulator üzerinde
yapılan **senaryo testlerini / denemelerini** kaydeder. Her deneyde tek bir şey değiştirilir
(örn. `urban` yerine `suburban`, ya da `HAPS_UE_SPEED_MPS=0` yerine `30`),
sonuç ölçülür ve **neden öyle olduğu** yorumlanır.

Diğer dokümanlarla ilişkisi:

| Doküman | İçerik |
|---|---|
| `HAPS_MIMARI.md` | Kanal bloğunun nasıl çalıştığı (kod mimarisi) |
| `HAPS_GELISTIRME_GUNLUGU.md` | Kodun **nasıl yazıldığı** — geliştirme adımları (Adım 1…37) |
| `HAPS_CALISTIRMA_REHBERI.md` | Her senaryonun **nasıl çalıştırılacağı** (komutlar) |
| **`HAPS_TEST_GUNLUGU.md` (bu dosya)** | Kanal üzerinde yapılan **testler / denemeler ve sonuçları** |

---

## 1. Deney disiplini — kurallar

1. **Tek değişken.** Her deney, bir *baz senaryoya* göre yalnızca **bir**
   parametreyi değiştirir. İki şeyi aynı anda değiştirdiysen bu iki ayrı deneydir.
2. **Baz senaryo sabittir.** Aşağıdaki Bölüm 3'teki baz senaryo, tüm
   karşılaştırmaların referansıdır. Baz senaryo değişirse yeniden ölçülür ve
   buraya not düşülür.
3. **Aynı süre, aynı bekleme.** Her koşu aynı süre çalıştırılır (öneri: gNB'yi
   başlat, 6 sn bekle, UE'yi başlat, **en az 60 sn** çalıştır) — kısa koşularda
   BLER/goodput oturmuyor.
4. **Rakamları logdan al, tahmin etme.** Bölüm 2'deki satırlardan gerçek
   değerleri kopyala.
5. **Her deney, koştuğu gün buraya yazılır** — "sonra yazarım" olmaz, çünkü
   hangi env var'la koştuğun birkaç saat sonra hatırlanmıyor.
6. **Kod/config kalıcı olarak değiştiyse** (env var değil, dosya düzenlemesi),
   bu bir *geliştirme adımıdır* — `HAPS_GELISTIRME_GUNLUGU.md`'ye de `### Adım N`
   olarak eklenir, sadece buraya değil.

---

## 2. Metrikleri nereden okurum

### gNB logu — periyodik MAC istatistik dökümü

gNB birkaç saniyede bir şuna benzer bir blok basar (`dump_mac_stats`,
`openair2/LAYER2/NR_MAC_gNB/main.c`):

```
Frame.Slot 512.0
UE RNTI 4a1c CU-UE-ID (none) in-sync PH 40 dB PCMAX 23 dBm, average RSRP -84 (12 meas), average SINR 21.3 (30 meas)
UE 4a1c: CQI 15, RI 1, PMI (0,0)
UE 4a1c: dlsch_rounds 1234/56/7/0, dlsch_errors 3, pucch0_DTX 0 (SNR 25.0+2.0 dB), BLER 0.00123 MCS (1) 27 CCE fail 0, goodput 12.34 Mbps
UE 4a1c: ulsch_rounds 890/12/1/0, ulsch_errors 0, ulsch_DTX 2, BLER 0.00010 MCS (1) 20 (Qm 4 deltaMCS 0 dB) NPRB 20 SNR 22.0 (+1.0) dB CCE fail 0, goodput 5.67 Mbps
```

Bir deneyde bakılacak alanlar:

| Alan | Anlamı | HAPS kanalında neyi yansıtır |
|---|---|---|
| `average RSRP` (dBm) | UE'nin ölçtüğü referans sinyal gücü | Yol kaybı (TR 38.811 senaryosu, mesafe, yağmur, O2I) |
| `average SINR` (dB) | Sinyal / (girişim+gürültü) | Yol kaybı + gürültü tabanı (kTB+NF) + sönümleme derinliği |
| `CQI` / `RI` / `PMI` | UE'nin bildirdiği kanal kalitesi / rank / precoder | RI>1 ancak gerçek 2x2 MIMO + iyi koşullu kanalda |
| `dlsch_rounds a/b/c/d` | HARQ tur dağılımı (1./2./3./4. iletim sayısı) | Yüksek `b,c,d` = sık yeniden iletim = kötü kanal |
| DL/UL `BLER` | Blok hata oranı | Sönümleme + SINR marjı |
| DL/UL `MCS (tbl) idx` | Kullanılan modülasyon-kodlama şeması | Kanal iyiyse yükselir, kötüyse düşer |
| DL/UL `goodput` (Mbps) | Fiili faydalı verim | Tüm yukarıdakilerin bileşik sonucu |
| `in-sync` / `out-of-sync` | UL senkron durumu | `out-of-sync` = bağlantı kopmuş |

### UE logu — bağlantı kilometre taşları

| Log satırı | Anlamı |
|---|---|
| `[PHY] UE synchronized!` | PBCH/SSB kilidi kuruldu |
| `[PHY] ... RSRP = ... dBm` | UE tarafı ham RSRP ölçümü (`csi_rx.c`) |
| `Found RAR with the intended RAPID` | RA (rastgele erişim) Msg2 kabul edildi |
| `[NR_RRC] ... Generating RRCSetupComplete` | RRC bağlantısı kuruldu = **başarı** |
| `RA failed at state ...` (gNB) | RA denemesi başarısız (kaç kez tekrarlandığına bak) |

### Debug env var'ları (opsiyonel derinlemesine iz)

`HAPS_DEBUG_38811=1` (yol kaybı hesabı), `HAPS_DEBUG_TDL=1` (sönümleme tapları),
`HAPS_DEBUG_O2I=1`, `HAPS_DEBUG_FRACDELAY=1`, `HAPS_DEBUG_RAIN_INTERNAL=1`,
`HAPS_DEBUG_TA=1`. Tam liste: `HAPS_CALISTIRMA_REHBERI.md` Bölüm 3.

---

## 3. Baz (referans) senaryo

> **DOLDURULACAK** — ilk temiz koşudan sonra bu tabloyu gerçek rakamlarla doldur.
> Bundan sonraki her deney "baz senaryoya göre şu değişti" diye buna atıf yapar.

| Öğe | Değer |
|---|---|
| gNB config | `gnb.haps_mobile_ntn_38811.conf` |
| UE config | `nrue.haps_mobile_ntn_38811.conf` |
| Kanal modeli | `HAPS_MOBILE_38811` (kırsal/banliyö, `HAPS_38811_SUBURBAN_RURAL`) |
| Bant / SCS / PRB | band 254 / µ=1 / 25 PRB |
| Platform | 20 km, 2 km yörünge, ~100 km/h |
| `HAPS_UE_SPEED_MPS` | (varsayılan 3 km/h) |
| `HAPS_GROUND_OFFSET_M` | 0 (zenit altı) |
| Yağmur / O2I / alt-TDL-profil | kapalı |
| Çalıştırma süresi | ___ sn |
| **Ölçülen — RSRP** | ___ dBm |
| **Ölçülen — SINR** | ___ dB |
| **Ölçülen — DL MCS / BLER / goodput** | ___ / ___ / ___ Mbps |
| **Ölçülen — UL MCS / BLER / goodput** | ___ / ___ / ___ Mbps |
| **RRC bağlantısı** | ___ (kaçıncı RA denemesinde) |
| **60 sn'de kopma sayısı** | ___ |

---

## 4. Değiştirilebilir parametreler kataloğu

Her satır bir deney fikri. "Beklenen yön", *baz senaryoya göre* metriğin hangi
yönde değişmesini beklediğimiz — deneyin işi bunu doğrulamak ya da çürütmek.

### 4a. Mekân senaryosu (config dosyası seçimi ile)

| Değişiklik | Nasıl | Beklenen yön |
|---|---|---|
| Kırsal/banliyö → **kentsel** | `..._urban.conf` çiftini kullan | Daha yüksek clutter kaybı → RSRP↓, SINR↓, MCS↓ |
| Kırsal/banliyö → **yoğun kentsel** | `..._dense_urban.conf` çiftini kullan | Kentselden de fazla kayıp → RSRP↓↓ |
| NLOS olasılığı | Senaryo + yükseklik açısı birlikte belirler (`haps_geometry.c`) | Düşük açı + kentsel = NLOS = derin sönümleme |

### 4b. Geometri (env var ile)

| Değişiklik | Nasıl | Beklenen yön |
|---|---|---|
| Terminali kapsama kenarına al | `HAPS_GROUND_OFFSET_M=35000` | Yükseklik açısı↓, eğik mesafe↑ → RSRP↓, gecikme↑, NLOS olasılığı↑ |
| Ara mesafe | `HAPS_GROUND_OFFSET_M=10000` / `20000` | Kademeli RSRP↓ |

### 4c. Kinematik / Doppler (env var ile)

| Değişiklik | Nasıl | Beklenen yön |
|---|---|---|
| UE'yi sabitle | `HAPS_UE_SPEED_MPS=0` | Küçük-ölçekli sönümleme donar → BLER daha kararlı, kopma yok |
| Hızlı UE | `HAPS_UE_SPEED_MPS=30` (108 km/h) | Doppler yayılımı↑ → daha hızlı fade → BLER↑, ara sıra kopma |
| Sabit platform | `HAPS_STATIONARY_38811*` config | Platform Doppler'i yok, sadece UE Doppler'i |

### 4d. Atmosfer (env var ile)

| Değişiklik | Nasıl | Beklenen yön |
|---|---|---|
| Yağmur | `HAPS_RAIN_RATE_MM_H=25` (orta), `=50` (şiddetli) | Ka-bantta belirgin ek kayıp → RSRP↓; S-bantta ihmal edilebilir |
| Troposferik sintilasyon | Otomatik (düşük açıda artar) | Düşük açıda RSRP'de küçük dalgalanma |

### 4e. Bina girişi (env var ile)

| Değişiklik | Nasıl | Beklenen yön |
|---|---|---|
| O2I bina girişi kaybı | `HAPS_O2I_ENABLE=1` | Sabit ~15-25 dB ek kayıp → RSRP↓, MCS↓ |
| O2I bina sınıfını "ısıl verimli" yap | `HAPS_O2I_ENABLE=1 HAPS_O2I_THERMAL=1` | ITU-R P.2109'da ısıl verimli binalar (metalize cam) daha **çok** RF engeller → O2I kaybı↑ (varsayılan "geleneksel" bina) |

### 4f. Küçük-ölçekli sönümleme profili (env var ile)

| Değişiklik | Nasıl | Beklenen yön |
|---|---|---|
| NTN-TDL-A/C → **B/D** | `HAPS_TDL_USE_ALT_PROFILE=1` | Farklı gecikme yayılımı / K-faktörü → BLER profili değişir |

### 4g. MIMO (config dosyası seçimi ile)

| Değişiklik | Nasıl | Beklenen yön |
|---|---|---|
| SISO → **2x2 MIMO** | `..._2x2.conf` çifti | İyi koşullu kanalda RI=2 → DL goodput ~2×; kötü kanalda fark yok |
| 2x1 SIMO/MISO | `..._2x1.conf` çifti | ⚠️ RRC'ye ulaşmaz (Adım 35) — sadece TDL kodu testi |

### 4h. Kalıcı config parametreleri (dosya düzenlemesi — ayrıca Geliştirme Günlüğü'ne yazılır)

`noise_power_dB`, `ploss_dB` (yalnızca ilk değer — sonra `haps_propagation.c`
üzerine yazar), `zeroCorrelationZoneConfig`, `prach_ConfigurationIndex`,
`cellSpecificKoffset_r17`, PRB sayısı, TX gücü. Bunları değiştirmek kod
davranışını kalıcı etkiler → deney + geliştirme adımı ikisi birden.

---

## 5. Deney girişi şablonu

Kopyala-yapıştır, her yeni deney için:

```markdown
### Deney N — <kısa başlık>

- **Tarih**: 2026-__-__
- **Hipotez**: <ne görmeyi bekliyorum ve neden>
- **Baz senaryoya göre değişen (tek şey)**: <parametre: eski değer → yeni değer>
- **gNB komutu**:
  ```
  <MALLOC_ARENA_MAX=1 ... nr-softmodem -O ... --rfsim>
  ```
- **UE komutu**:
  ```
  <MALLOC_ARENA_MAX=1 ... nr-uesoftmodem -O ... --rfsim>
  ```
- **Çalıştırma süresi**: __ sn

**Ölçülen sonuç**

| Metrik | Baz senaryo | Bu deney | Δ |
|---|---|---|---|
| RSRP (dBm) | | | |
| SINR (dB) | | | |
| DL MCS | | | |
| DL BLER | | | |
| DL goodput (Mbps) | | | |
| UL MCS | | | |
| UL BLER | | | |
| UL goodput (Mbps) | | | |
| RRC bağlantısı | | | |
| Kopma sayısı | | | |

**Yorum**

<Ne oldu? Hipotez doğrulandı mı? Fiziksel açıklama: neden bu yönde değişti?
Beklenmedik bir şey varsa: neden olabilir, hangi dosyaya bakılmalı?>

**Sonuç**: ✅ hipotez doğrulandı / ❌ çürütüldü / ⚠️ kısmen — <tek cümle>
```

---

## 6. Deneyler

<!-- Yeni deneyler buraya, kronolojik sırayla eklenir. En yenisi en altta. -->

_(Henüz deney yok — ilk koşu baz senaryoyu kalibre etmek için: Bölüm 3'ü doldur.)_
