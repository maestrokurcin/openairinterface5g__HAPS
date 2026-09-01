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

> İlk kalibrasyon: **2026-09-01**. **Yeniden kalibre edildi 2026-09-01** —
> Deney 2'de bulunan int16-taşması / donmayan-rastgelelik bug'ları düzeltildikten
> sonra (Geliştirme Günlüğü Adım 39 + 40). Aşağıdaki 3.2 tablosu **düzeltme
> sonrası** değerleri gösterir; eski (taşmalı kalibrasyon) değerleri "eski" olarak
> not düşüldü. Bundan sonraki her deney bu tabloya atıf yapar.

> **Önemli**: `path_loss_dB` artık en-iyi-durum geometriye göreli bir **net
> kazanç** (`HAPS_DEBUG_38811=1` → `netgain` alanı). Referans (banliyö zenit LOS)
> ≈ −6 dB; her kötü geometri daha negatif. Bu artık senaryo karşılaştırmasının
> **birincil metriği** — UE-bildirimli SINR referans linkte CQI tavanına (~+40)
> railleniyor, o yüzden çalışan senaryolar arasında ayırt edici değil.

### 3.1 Konfigürasyon

| Öğe | Değer |
|---|---|
| gNB config | `gnb.haps_mobile_ntn_38811.conf` |
| UE config | `nrue.haps_mobile_ntn_38811.conf` |
| Kanal modeli | `HAPS_MOBILE_38811` (kırsal/banliyö, `HAPS_38811_SUBURBAN_RURAL`) |
| Bant / SCS / PRB | band 254 (S-bandı, SSB ~2.4886 GHz) / **µ=0 (15 kHz)** / 25 PRB / TDD |
| NTN | gerçek `ntn_Config_r17` + SIB19, `cellSpecificKoffset_r17 = 1`, `positionZ-r17 ≈ 20 km` |
| Platform | 20 km yükseklik, 2 km yörünge yarıçapı, ~100 km/h loiter |
| `HAPS_UE_SPEED_MPS` | (ayarlanmadı — koddaki varsayılan) |
| `HAPS_GROUND_OFFSET_M` | 0 (zenit altı, yükseklik açısı ~90°) |
| Yağmur / O2I / alt-TDL-profil | kapalı |
| Derleme | `ninja rfsimulator nr-softmodem nr-uesoftmodem`, develop @ `dfe5bd4786` |
| Çalıştırma | gNB başlat → 12 sn bekle → UE başlat → ~90 sn |
| Trafik | **Yok** — bu makinede AMF/5GC bağlı değil; bağlantı RRC_CONNECTED'te kalıyor, sadece SRB1 keepalive taşınıyor |

### 3.2 Ölçülen sonuç

| Metrik | Değer (düzeltme sonrası) | Not |
|---|---|---|
| **net kazanç (`netgain`)** | **≈ −5.7 dB** | LOS (donmuş), `PLb ≈ 126` ≈ `FSPLref`; referans nokta |
| RA sonucu | ✅ **ilk denemede** başarılı | kalan mesafe tahmini ~390 m (NTN ön-telafisi çalışıyor) |
| RRC bağlantısı | ✅ `NR_RRC_CONNECTED`, `RRCSetupComplete` | senkron ~frame 794 (bu makinede yavaş — temiz kodda da öyle, kalibrasyonla ilgisi yok) |
| Kopma / out-of-sync | **0** (~90 sn boyunca `in-sync`) | 0 re-sync, 0 RA-fail |
| DL HARQ turları | `78/0/0/0` | hiç yeniden iletim yok |
| UL HARQ turları | `765/0/0/0` | hiç yeniden iletim yok |
| DL / UL BLER | ≈ 0 / 0 | `dlsch_errors 0`, `ulsch_errors 0` |
| gNB PUSCH SNR | ~17.2 dB (hedef +2.2) | kararlı (güç kontrolü hedefe kilitliyor) |
| gNB PUCCH SNR | ~22 dB | kararlı |
| UE-bildirimli DL SINR (`average SINR`) | medyan **+39.8 dB**, aralık +39.1 … +40.0 | **CQI tavanına raillenmiş** — link "çok iyi"; çalışan senaryolar arası ayırt edici değil |
| gürültü tabanı (`noise_power_dB`) | −38.0 (5 MHz, µ=0) | `-174 + 10log10(BW) + 9 + 60` (NOISE_SIM_SCALE, Adım 40) |
| DL tek-yön gecikme | ~0.067 ms | çok kararlı |
| Doppler (SAT→UE) | ~21 Hz | HAPS platformu + yavaş UE, ihmal edilebilir |
| DL/UL MCS, goodput | 0 / 0 | **anlamlı değil** — trafik yok (5GC bağlı değil) |

**Eski (taşmalı kalibrasyon, düzeltmeden önce) — artık geçersiz**: DL SINR medyan
+0.4 dB (−16 … +12), "net ploss ~30 dB". Bu değerler int16 sarma bozulmasının
ürünüydü; bkz. Geliştirme Günlüğü Adım 40.

### 3.3 Yorum

- **Bağlantı sağlığı mükemmel**: RA ilk denemede geçti, ~90 sn boyunca tek bir
  kopma/yeniden iletim yok, DL ve UL BLER sıfıra yakın.
- **DL SINR CQI tavanında raillenmiş** (~+40): referans link çok iyi. Adım 40'tan
  sonra net kazanç yakın-zenit için −6 dB (sarma yok); modellenen gürültü tabanı
  bunu +40 dB üstünde bir SINR'a oturtuyor, CQI tablosu +40'ta kesiliyor. Yani
  *çalışan* senaryolar arasında SINR ayırt edici değil — **`netgain` dB birincil
  karşılaştırma metriği**.
- **Karşılaştırma metrikleri**: `netgain` (birincil), sonra BLER, HARQ tur
  dağılımı, kopma sayısı, RA başarı/deneme, gecikme, Doppler. MCS/goodput
  ölçülemiyor (5GC yok — gerçek verim için ayrı adımda docker/5GC ya da
  `--phy-test` gerekli).
- **RSRP dBm olarak loglanmıyor**: bu conf'ta CSI-RS RSRP raporlaması yok;
  `HAPS_DEBUG_38811=1` çıktısındaki `netgain` (ve `PLb`) yol kaybının temsilci
  metriği.

### 3.4 Yeniden üretme komutları

```
# Derleme
cd /home/furkan/openairinterface5g/cmake_targets
ninja -C ran_build/build rfsimulator nr-softmodem nr-uesoftmodem

# Terminal 1 — gNB
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-softmodem -O ../haps_test/gnb.haps_mobile_ntn_38811.conf --rfsim

# Terminal 2 — UE (gNB'den ~12 sn sonra)
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-uesoftmodem -O ../haps_test/nrue.haps_mobile_ntn_38811.conf --rfsim
```

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
| `net ploss_dB` (büyük-ölçek, ~dB) | ~30–31 | | |
| UE-bildirimli DL SINR: medyan / aralık (dB) | +0.4 / −16.6…+12.0 | | |
| gNB PUSCH SNR (dB) | ~16.3 | | |
| DL BLER | ≈0 | | |
| UL BLER | 0 | | |
| DL HARQ turları | 166/0/0/0 | | |
| UL HARQ turları | 1652/0/0/0 | | |
| RA sonucu | ilk denemede ✅ | | |
| RRC bağlantısı | ✅ CONNECTED | | |
| Kopma sayısı (~90 sn) | 0 | | |
| _(trafik varsa)_ DL/UL MCS, goodput | N/A (5GC yok) | | |

**Yorum**

<Ne oldu? Hipotez doğrulandı mı? Fiziksel açıklama: neden bu yönde değişti?
Beklenmedik bir şey varsa: neden olabilir, hangi dosyaya bakılmalı?>

**Sonuç**: ✅ hipotez doğrulandı / ❌ çürütüldü / ⚠️ kısmen — <tek cümle>
```

---

## 6. Deneyler

<!-- Yeni deneyler buraya, kronolojik sırayla eklenir. En yenisi en altta. -->

### Deney 1 — Kırsal/banliyö → Kentsel (düzeltilmiş kalibrasyon, 2026-09-01)

- **Değişen tek şey**: kanal senaryosu `SUBURBAN_RURAL` → `URBAN`
  (config çifti `..._38811.conf` → `..._38811_urban.conf`), zenit (ground offset yok)
- **Beklenti**: kentsel clutter/gölgeleme daha fazla → sinyal kötüleşir
- **Süre**: 1×90 sn + 2×25 sn (SF varyansı için), trafik yok
- **Kalibrasyon**: Adım 39 + 40 sonrası (donmuş SF/LOS-NLOS, referans-göreli netgain)

**Kısaca ne oldu**

3 koşunun 3'ünde de LOS çekildi (`los_prob[urban,90°] = 0.968` — beklenen) ve
link **temiz bağlandı**: ilk RA denemesi, 0 kopma, 0 yeniden iletim, BLER ≈ 0 —
baz senaryoyla (banliyö zenit) pratikte aynı sağlık. Tek fark: **netgain'in
koşudan koşuya değişkenliği arttı** (−4.3 … −7.0 dB, baz senaryo ~−6'da sıkı),
çünkü kentsel LOS gölge-sönümleme sigması 4.0 dB, banliyönün 0.72 dB'sine karşı.

**Rakamlar**

| Metrik | Baz (banliyö zenit) | Deney 1 (kentsel zenit), 3 koşu |
|---|---|---|
| LOS/NLOS (donmuş) | LOS | LOS / LOS / LOS |
| netgain | ≈ −5.7 dB (sıkı) | **−6.7 / −4.3 / −7.0 dB** (σ_SF=4 dB yüzünden yayvan) |
| `PLb` | ~126 dB | 124.6 … 127.3 dB |
| RA / RRC | ilk denemede ✅ | ilk denemede ✅ (3/3) |
| Kopma / out-of-sync | 0 | 0 (3/3) |
| DL / UL HARQ | 78/0/0/0 · 765/0/0/0 | ~38/0/0/0 · ~370/0/0/0 — **0 yeniden iletim** |
| DL / UL BLER | ≈0 / 0 | ≈0 / 0 |
| gNB PUSCH SNR | ~17.2 dB | ~17.1 dB |
| DL SINR (UE) | +39.8 (CQI tavanı) | +39.8 (CQI tavanı) — ayırt edici değil |
| Gecikme / Doppler | 0.067 ms / 21 Hz | 0.068 ms / 12 Hz | 

**Neden böyle**

Zenitte (yükseklik açısı 90°) kentsel LOS olasılığı çok yüksek (0.968), NLOS
neredeyse hiç çekilmiyor — ve **clutter kaybı sadece NLOS'ta var**. Yani
zenitte kentselin banliyöye göre tek cezası clutter değil, **daha geniş
gölgeleme varyansı** (σ_SF 4.0 vs 0.72 dB). Bu da netgain'i sistematik olarak
düşürmüyor, koşu-başına daha geniş bir aralığa yayıyor. TR 38.811 fiziğiyle
tutarlı: yüksek yükseklik açısında kentsel vs banliyö farkı **ortalama yol
kaybında değil, gölgeleme varyansında**.

**Eski (taşmalı kalibrasyon) koşusuyla fark**: O koşu "medyan SINR −3.5 dB,
9 UL yeniden iletim, başıboş RA hatası" gösteriyordu — hepsi Adım 39/40
bug'larının artefaktıydı (saniyede-bir NLOS-flip → anlık +25 dB clutter, +
int16 sarma). Düzeltmeyle tamamen kayboldu.

**Sonuç**: ✅ Hipotez kısmen doğrulandı — kentsel, sinyali *sistematik* olarak
kötüleştirmiyor (zenitte); etkisi **gölgeleme belirsizliğinin artması**. Gerçek
kötüleşme kentsel + düşük yükseklik açısında (NLOS + clutter) görülüyor → Deney 2.

---

### Deney 2 — Kentsel + terminal 35 km uzakta (düşük yükseklik açısı)

- **Tarih**: 2026-09-01
- **Değişen tek şey**: `HAPS_GROUND_OFFSET_M=35000` eklendi (Deney 1'in kentsel
  kurulumuna). Terminal, platform izdüşümünden 35 km uzağa taşınıyor.
- **Beklenti**: yükseklik açısı ~90° → ~27° düşer; eğik mesafe ~20 → ~44 km;
  NLOS + kentsel clutter kaybı devreye girer → sinyal **çok daha kötü** olmalı,
  gecikme ve Doppler artmalı.
- **Süre**: ~90 sn, trafik yok.

**Kısaca ne oldu**

Geometri beklendiği gibi değişti: gecikme iki katına, Doppler 10 katına çıktı.
Yol kaybının fiziği de doğru hesaplandı — `HAPS_DEBUG_38811` çıktısı yükseklik
açısını 27°, durumu NLOS, clutter kaybını +29 dB, toplam `PLb`'yi ~166 dB
gösterdi (zenitte ~126 dB). **Ama sonuç ters çıktı: bağlantı kötüleşeceğine
_iyileşti_.** SINR medyanı +0.4 → **+15.5 dB** fırladı, uplink yeniden
iletimleri Deney 1'e göre azaldı (9 → 2), PUSCH SNR yine ~16 dB'de kararlı.

**Rakamlar**

| Metrik | Baz (banliyö, zenit) | Deney 1 (kentsel, zenit) | Deney 2 (kentsel, 35 km) |
|---|---|---|---|
| Yükseklik açısı | ~90° | ~90° | **~27°** |
| Eğik mesafe | ~20 km | ~20 km | **~44 km** |
| Tek-yön gecikme | 0.067 ms | 0.068 ms | **0.141 ms** |
| Doppler (SAT→UE) | 21 Hz | 22 Hz | **199 Hz** |
| Büyük-ölçek yol kaybı `PLb` | ~126 dB | ~128 dB | **~166 dB** (NLOS, CL +29 dB) |
| DL SINR — medyan | +0.4 dB | −3.5 dB | **+15.5 dB** (?!) |
| DL SINR — aralık | −16.6…+12.0 | −23…+11 | −17…+39 |
| gNB PUSCH SNR | ~16 dB kararlı | 1–14 dB oynak | ~16 dB kararlı |
| UL HARQ yeniden iletim | 0 | 9 | 2 |
| DL / UL BLER | ≈0 / 0 | ≈0 / 0 | ≈0 / 0 |
| RA / RRC | ilk denemede ✅ | ilk denemede ✅ | ilk denemede ✅ |
| Kopma (~90 sn) | 0 | 0 | 0 |

**⚠️ Beklenmedik sonuç — muhtemel bug**

Fizik doğru hesaplanıyor ama sinyale yanlış yansıyor. `PLb` zenite göre ~40 dB
arttığı halde uygulanan sinyal kazancı **artması gereken yönde değil, azalan
yönde** değişiyor; net etki bağlantının güçlenmesi oluyor. Delay/Doppler'in
doğru çıkması, `haps_geometry.c` (geometri/kinematik) yolunun sağlam olduğunu
gösteriyor — sorun büyük-ölçek yol kaybının **kazanca çevrildiği** yerde.

İncelenecek yer: `openair1/SIMULATION/TOOLS/haps_propagation.c:200` —
`return eirp_dBW + HAPS_38811_SIM_CALIBRATION_DB - PLb_dB;`. Bu değer
`channelDesc->path_loss_dB`'ye (bir **kazanç**, negatif = kayıp) atanıyor.
Zenitte `PLb ≈ 126` → kazanç ≈ `42.99 + 114.93 − 126 ≈ +31 dB`; Deney 2'de
`PLb ≈ 166` → kazanç ≈ `−8 dB`. Yani düşük kazanç = zayıf sinyal olması
gerekirken ölçülen SINR tam tersi. Ayrıca `net ploss_dB` logunun ortalaması
(+12) el hesabıyla (~−4) uyuşmuyor — gölgeleme (`sigma_SF·gaussZiggurat`)
teriminin ortalaması/işareti ya da `HAPS_38811_SIM_CALIBRATION_DB = 114.93`
sabitinin yalnızca zenit senaryosuna göre kalibre edilmiş olması şüpheli.

**Sonuç (ilk koşu, düzeltmeden önce)**: ⚠️ Hipotez **çürütüldü** — düşük
yükseklik açısı + kentsel, sinyali kötüleştirmesi gerekirken iyileştirdi.
Geometri/gecikme/Doppler doğruydu; sorun büyük-ölçek yol kaybının sinyal
kazancına çevrilmesindeydi.

**Yeniden üretme**

```
# gNB ve UE komutlarının başına (ikisine de):
HAPS_GROUND_OFFSET_M=35000  [+ isteğe bağlı HAPS_DEBUG_38811=1]
# config çifti: gnb.haps_mobile_ntn_38811_urban.conf / nrue.haps_mobile_ntn_38811_urban.conf
```

---

#### Deney 2 — BUG DÜZELTİLDİ (2026-09-01, Geliştirme Günlüğü Adım 39 + 40)

İki kök neden bulunup düzeltildi:

1. **Adım 39 — büyük-ölçekli rastgelelik donmuyordu**: `haps_38811_path_loss_dB()`
   gölge sönümlemeyi ve LOS/NLOS durumunu **saniyede bir** yeniden çekiyordu. İkisi
   de büyük-ölçekli, bağlantı başına tek gerçekleşim olmalı. Artık ilk çağrıda bir
   kez çekilip donuyor.
2. **Adım 40 — int16 taşması**: rfsim `path_loss_dB`'yi bir kazanç olarak
   uyguluyor ve eski kalibrasyon yakın-zeniti ~+30 dB net kazanca koyuyordu → RX
   örnekleri int16'yı aşıp **sarıyordu**; SNR'ı belirleyen bu sarma bozulmasıydı.
   Daha kötü geometri → daha az sarma → **daha yüksek** ölçülen SINR (inversiyon).
   Artık net kazanç **en iyi durum geometriye göreli** hesaplanıyor (zenit LOS ≈
   −6 dB, her kötü geometri kesinlikle daha zayıf, asla pozitif → asla sarma) ve
   gürültü tabanı rfsim'in genlik rejimine ölçeklendi.

**Düzeltme sonrası Deney 2** (`HAPS_DEBUG_38811=1` ile `netgain` görünür):

| LOS/NLOS çekimi (donmuş) | netgain | Sonuç |
|---|---|---|
| **NLOS** (koşuların ~%50-65'i, `los_prob[urban,27°] ≈ 0.49`) | −38 … −50 dB | ❌ **Link kurulamıyor** — UE senkron olamıyor (700+ synch-fail) |
| **LOS** | ~−11 dB | ✅ Kurulur, netgain zenitin ~5 dB altında |

**Yorum**: Inversiyon çözüldü. Düşük açılı kentsel NLOS link (44 km eğik mesafe,
+29 dB clutter kaybı) artık **fiziksel olarak doğru şekilde başarısız oluyor**.
LOS çekildiğinde link çalışıyor ama zenitten zayıf. Deney artık **stokastik**:
27° yükseklik açısında platforma görüş hattın olup olmaması gerçekten ~yazı-tura
ve linki yapar/bozar — bu bir bug değil, doğru davranış.

**Regresyon kontrolü** (hepsi düzeltme sonrası bağlanıyor): baz `_38811`,
plain NTN (Senaryo 3), Deney 1 (kentsel zenit), band78 `HAPS_STATIONARY`
(Senaryo 1), MIMO 2x2 (Senaryo 7).

**Kalan sınır**: referans link SINR'ı hâlâ CQI tablo tavanında (~+40) railleniyor
→ *çalışan* senaryolar arasındaki ince farklar (banliyö vs kentsel zenit gibi)
SINR üzerinden hâlâ görünmüyor; sadece pass/fail sınırı ve yön artık doğru.
Karşılaştırmalar için `HAPS_DEBUG_38811=1` çıktısındaki **`netgain` dB** en
temiz metrik.

**Sonuç**: ✅ Bug düzeltildi (Adım 39 + 40). Hipotez artık **doğrulanıyor** —
düşük yükseklik açısı + kentsel NLOS, linki (çoğu zaman tamamen) bozuyor.

---

### Deney 3 — Yağmur (25 mm/h), S-bandı

- **Tarih**: 2026-09-01
- **Değişen tek şey**: `HAPS_RAIN_RATE_MM_H=25` eklendi (baz senaryoya — banliyö
  zenit, band 254 ≈ 2.49 GHz)
- **Beklenti**: S-bandında yağmur sönümlemesi ihmal edilebilir (ITU-R P.618/P.838:
  yağmur kaybı ~f²–f²·⁵ ile ölçeklenir, 2.5 GHz'de 20 GHz'e göre ~1000× küçük)
- **Süre**: ~85 sn, trafik yok, Adım 39+40 kalibrasyonu

**Sonuç**

| Metrik | Baz (yağmursuz) | Deney 3 (25 mm/h) |
|---|---|---|
| **yağmur terimi** (`HAPS_DEBUG_38811`) | 0.000 dB | **0.013 dB** |
| netgain | ≈ −5.7 dB | −6.1 dB (fark SF çekimi kaynaklı, yağmur değil) |
| RA / RRC / kopma | ✅ / ✅ / 0 | ✅ / ✅ / 0 |
| DL / UL HARQ | temiz | temiz (0 yeniden iletim) |
| PUSCH SNR / DL SINR | ~17.2 / +39.8 | ~17.4 / +39.8 |

**Yorum**

25 mm/h "orta-şiddetli" yağmur, 2.49 GHz'de **0.013 dB** ek kayıp ekliyor —
ölçüm gürültüsünün altında, hiçbir gözlemlenebilir etkisi yok. Model doğru
davranıyor: ITU-R P.838 k/alpha katsayıları S-bandında çok küçük.

Bir **Ka-bandı** HAPS senaryosunda (model destekliyor, `HAPS_38811_S_VS_KA_THRESHOLD_GHZ = 6 GHz`)
aynı yağmur oranı ~birkaç dB kayıp verirdi ve fark net görülürdü — ama test
paketinde Ka-bandı config'i yok (hepsi band 254/78, S-bandı). Ka senaryosu
eklenirse bu deney orada anlamlı olur.

**Sonuç**: ✅ Hipotez doğrulandı — S-bandında yağmurun etkisi yok (0.013 dB).
Yağmur, yalnızca Ka-bandı için anlamlı bir değişken.
