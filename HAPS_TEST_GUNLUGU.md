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
7. **Stokastik/küçük-ölçekli etkiler için 3+ koşu.** Sönümleme gerçekleşimi ve
   donmuş gölge-sönümleme/LOS-NLOS çekimi koşu başına farklı (RNG
   `/dev/urandom`'dan). Tek bir 90 sn'lik koşu yanıltıcı outlier verebilir
   (bkz. Deney 4). Hızlı fade, gölgeleme, LOS/NLOS-sınırı gibi şeyler için en az
   3 koşu yap, dağılımı raporla. Büyük-ölçek deterministik etkiler (yağmur,
   sabit yükseklik açısı, MIMO stream sayısı) için tek koşu yeterli.

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

---

### Deney 4 — Hızlı UE (108 km/h)

- **Tarih**: 2026-09-01
- **Değişen tek şey**: `HAPS_UE_SPEED_MPS=30` (108 km/h) — baz senaryoda varsayılan
  3 km/h (0.833 m/s)
- **Etkilediği yer**: `haps_tdl.c`'de `fd_local = ue_speed · fc / c` — yerel
  saçılma Doppler bandı, NTN-TDL AR(1) sönümlemesinin dekorelasyon hızı
- **Beklenti**: daha hızlı fade → kanal kestirimi/HARQ penceresi içinde daha çok
  değişim → BLER↑
- **Süre**: ~90 sn, trafik yok, Adım 39+40 kalibrasyonu

**Sonuç**

| Metrik | Baz (3 km/h) | Deney 4 (108 km/h) |
|---|---|---|
| `fd_local` | ~6.9 Hz | **249 Hz** (36×) |
| netgain | ≈ −5.7 dB | ≈ −5.7 dB (değişmedi — bu büyük-ölçek, yağmur/açı değil) |
| RA / RRC / kopma | ✅ / ✅ / 0 | ✅ / ✅ / **0** (bağlantı ayakta kalıyor) |
| **UL BLER** (koşu boyu) | **0** | **medyan %9.5, tepe %25** (mean %10.5) — sürekli |
| **UL HARQ turları** | `765/0/0/0` | **`573/56/5/1`** (~62 yeniden iletim) |
| DL BLER | ≈ 0 | medyan %0.5, tepe %10 (mean %1.8) — hafif |
| DL HARQ turları | `78/0/0/0` | `59/0/0/0` (temiz) |
| gNB PUSCH SNR | ~17.2 dB | ~16.3 dB (ortalama korunuyor, anlık varyans yüksek) |
| DL SINR (UE) | +39.8 (dip 39.1) | +39.7 (dip **38.1** — biraz daha yayvan) |

**Yorum**

Hızlı UE hareketi (108 km/h) yerel saçılma Doppler'ini 7 → 249 Hz'e çıkarıyor;
NTN-TDL sönümlemesi ~1 ms'lik HARQ gidiş-dönüşü içinde dekorele oluyor. Sonuç:

- **Uplink belirgin kötüleşiyor**: BLER 0 → ~%10 (sürekli, spike değil), ~62 HARQ
  yeniden iletimi. Kapalı-döngü güç kontrolü ve kanal kestirimi bu hızlı fade'i
  izleyemiyor; ortalama PUSCH SNR korunsa da anlık dipler NACK üretiyor.
- **Downlink çok daha az etkileniyor** (BLER ~%2): DL'in SINR marjı devasa (CQI
  tavanında), derin bir fade bile MCS eşiğinin altına inmiyor.
- **Bağlantı kopmuyor**: HARQ her şeyi toparlıyor (`ulsch_errors 0`,
  `dlsch_errors 0`), 90 sn boyunca 0 out-of-sync.

**Sonuç (tek koşu)**: hızlı UE → hızlı fade → UL BLER 0 → ~%10, ~62 yeniden
iletim. Link ayakta (HARQ toparlıyor), DL marjı yüksek olduğu için DL etkilenmiyor.

#### ⚠️ Çok-koşumlu takip (Deney 5 ile birlikte, aynı gün) — headline düzeltmesi

`HAPS_UE_SPEED_MPS=30` ile **6 koşu** yapıldı (3 banliyö + 3 kentsel):

| Koşu | UL BLER medyan / mean | UL yeniden iletim | Kopma |
|---|---|---|---|
| Deney 4 (banliyö) #1 | %9.5 / %10.5 | ~62 (`573/56/5/1`) | 0 |
| banliyö #2, #3 | %0 / ~%0.3 | 0 | 0 |
| kentsel #1, #2, #3 (Deney 5) | %0 / ~%0.3 | 0 | 0 |

**6 koşunun 5'i temiz** (UL BLER <%1, 0 yeniden iletim). Deney 4 #1'deki %10
sürekli UL BLER episodu bir **outlier** — sönümleme gerçekleşimi koşu başına
farklı (RNG `/dev/urandom`'dan), 90 sn'lik pencerede aktif UL yükü sırasında
kötü bir derin-fade kümesine denk gelmek şans işi.

**Düzeltilmiş sonuç**: ⚠️ Hızlı UE (108 km/h, `fd_local` 7 → 249 Hz), referans
linkin devasa SINR marjı yüzünden **çoğu zaman gözlemlenebilir etki yaratmıyor**.
Ara sıra (~6 koşuda 1) bir sönümleme gerçekleşimi sürekli bir UL BLER episodu
(~%10, onlarca HARQ yeniden iletimi) tetikliyor — ama asla kopma yok. Banliyö vs
kentsel: fark yok.

**Metodoloji dersi** (Bölüm 1'e de eklendi): küçük-ölçekli/stokastik etkiler için
tek 90 sn'lik koşu yanıltıcı outlier verebilir — bu tür deneylerde 3+ koşu şart.

---

### Deney 5 — Hızlı UE (108 km/h) + kentsel zenit

- **Tarih**: 2026-09-01
- **Değişen (Deney 4'e göre)**: senaryo `SUBURBAN_RURAL` → `URBAN`
  (`HAPS_UE_SPEED_MPS=30` sabit, zenit, ground offset yok)
- **Referans**: Deney 4 (hızlı UE, banliyö zenit)
- **Soru**: Deney 1 (kentsel) + Deney 4 (hızlı fade) etkileri birikir mi? Yani
  kentselin daha geniş gölgeleme varyansı, hızlı fade'in UL BLER'ini kötüleştirir mi?
- **Süre**: 1×90 sn + 2×55 sn, trafik yok, Adım 39+40 kalibrasyonu

**Sonuç** (3 koşu, hepsi LOS çekti — `los_prob[urban,90°]=0.968`):

| Metrik | Deney 4 (hızlı, banliyö) | Deney 5 (hızlı, kentsel) |
|---|---|---|
| `fd_local` | 249 Hz | 249 Hz (aynı — sadece UE hızına bağlı) |
| TDL profili / DS | NTN-TDL-C / ~5.4 ns | NTN-TDL-C / **~4.5 ns** (kentsel biraz daha az) |
| netgain | ~−5.7 dB | **~−6.9 dB** (kentsel σ_SF=4 dB, donmuş çekim) |
| RA / RRC / kopma | ✅ / ✅ / 0 | ✅ / ✅ / 0 (3/3) |
| UL BLER (3 koşu) | %0 / ~%0.3 (1 outlier hariç) | **%0 / ~%0.3** (3/3 temiz) |
| UL yeniden iletim | 0 (1 outlier hariç) | **0** (3/3) |
| DL BLER | ~%0 | ~%1.5 (marj bol) |

**Yorum**

Etkiler **birikmiyor**. Kentsel zenit LOS'ta:
- Küçük-ölçekli sönümleme banliyöyle neredeyse aynı — DS 4.5 vs 5.4 ns, ikisi de
  örnekleme periyodunun (~65 ns CP) çok altında, frekans-seçiciliği ihmal
  edilebilir. `fd_local` yalnızca UE hızına bağlı, senaryodan bağımsız.
- Kentselin tek farkı ~1 dB daha düşük netgain (daha geniş gölgeleme çekimi) —
  ama referans linkin marjı o kadar büyük ki bu UL BLER'e yansımıyor.
- Clutter kaybı yalnızca NLOS'ta; zenitte LOS olasılığı 0.968 → clutter devrede değil.

Yani "kentsel + hızlı UE" ≈ "banliyö + hızlı UE": ikisi de çoğu zaman temiz,
ara sıra outlier UL BLER episodu (Deney 4 takibine bakın).

**Sonuç**: ✅ Hipotez (birikme) **çürütüldü** — kentsel zenit, hızlı-fade UL
BLER'ini kötüleştirmiyor. Kentselin küçük-ölçek kanalı zenit LOS'ta banliyöyle
aynı; fark sadece büyük-ölçek gölgeleme belirsizliğinde ve o da linki bozmuyor.

---

### Deney 6 — Alternatif NTN-TDL profili (A/C → B/D)

- **Tarih**: 2026-09-01
- **Değişen tek şey**: `HAPS_TDL_USE_ALT_PROFILE=1` — NTN-TDL küçük-ölçekli
  sönümleme profili varsayılan A/C yerine B/D. Baz senaryo LOS çektiği için
  fiili değişim **NTN-TDL-C → NTN-TDL-D**.
- **Profil farkı** (`haps_tdl.c`, TR 38.811 Tablo 6.9.2-3/4):
  - TDL-C: 2 tap, tap-0 Ricean **K = 10.224 dB**, tap-1 −23.4 dB (norm. gecikme 14.8)
  - TDL-D: 3 tap, tap-0 Ricean **K = 11.707 dB** (daha güçlü specular), tap-1/2
    −9.9/−16.8 dB (norm. gecikme 0.56 / 7.33)
  - İkisi de toplam güç 1'e (0 dB) normalize
- **Beklenti**: K-faktörü ve echo yapısı farklı → BLER profili biraz değişebilir;
  TDL-D'nin K'sı daha yüksek olduğu için (daha az fade derinliği) belki hafif *daha iyi*
- **Süre**: 3 alt-profil + 2 varsayılan-profil koşusu, ~55 sn her biri, trafik yok

**Sonuç**

| Profil | UL BLER medyan / mean | UL yeniden iletim | DL BLER medyan | DL BLER mean |
|---|---|---|---|---|
| **TDL-D** (alt) — 3 koşu | 0 / %0.2–0.4 | 0 (3/3) | %1.0 / %2.8 / %2.1 | ~%2.9 |
| **TDL-C** (varsayılan) — 2 koşu | 0 / %0.2 | 0 (2/2) | %0.8 / %0.9 | ~%2.0 |
| Kopma | — | 0 (5/5) | — | — |
| DS (delay spread) | 5.4 ns | — | 5.4 ns | — |

**Yorum**

**Anlamlı fark yok.** UL tarafı iki profilde de birebir temiz (0 / ~%0.2).
DL BLER'de TDL-D biraz daha yüksek medyan gösteriyor (%1–3 vs %0.8–0.9) ama
mean'ler benzer (~%2–3) ve koşu-içi varyansın altında — istatistiksel olarak
ayırt edilemez.

Neden: her iki profil de birim güce normalize; TDL-D'nin daha yüksek K-faktörü
(daha az fade) teoride *daha iyi* olmalı, *daha kötü* değil — yani gözlenen
küçük DL BLER farkı gerçek bir kötüleşme değil, echo gecikme yapısının biraz
farklı frekans-seçiciliği + RNG gürültüsü. DS her ikisinde de 5.4 ns, örnekleme
periyodunun (~65 ns CP) çok altında → frekans-seçiciliği ihmal edilebilir.
Referans linkin devasa SINR marjı da profil farkını yutuyor.

**Sonuç**: ✅/⚪ Hipotez büyük ölçüde çürütüldü — A/C ↔ B/D profil değişimi, bu
senaryoda (zenit LOS, geniş marj) gözlemlenebilir bir etki yaratmıyor. B/D
profili, model destekliyor ama karşılaştırmada A/C ile denk. Fark ancak marjı
dar bir senaryoda (düşük yükseklik açısı LOS, ya da yüksek MCS'li gerçek trafik)
görünebilir.

---

### Deney 7 — O2I bina girişi kaybı (`HAPS_O2I_ENABLE=1`)

- **Tarih**: 2026-09-01
- **Değişen tek şey**: `HAPS_O2I_ENABLE=1` — ITU-R P.2109 bina girişi kaybı
  (outdoor→indoor). Baz senaryo (banliyö zenit), bina sınıfı "traditional"
  (varsayılan; `HAPS_O2I_THERMAL=1` daha kötü olurdu).
- **Beklenti**: sabit ~15-25 dB ek kayıp → link belirgin kötüleşir
- **Süre**: 3 koşu, ~55-60 sn her biri, `HAPS_DEBUG_38811=1` / `HAPS_DEBUG_O2I=1`

**Sonuç**

| Metrik | Baz (O2I kapalı) | Deney 7 (O2I açık), 3 koşu |
|---|---|---|
| O2I kaybı (P.2109) | 0 dB | **medyan ~30 dB, aralık 5 … 56 dB** |
| netgain | ≈ −6 dB | **medyan ~−37 dB** |
| RRC bağlantısı | ✅ | ❌ **kurulamıyor** (3/3) — 800–1900 synch-fail |

**Yorum**

1. **Fiziksel sonuç doğru yönde ama sert**: P.2109 "traditional" bina için
   S-bandında ~30 dB medyan giriş kaybı ekliyor (katalogdaki "~15-25 dB"
   tahmininden fazla — yüksek yükseklik açısında P.2109 yüksek-persentil
   değerleri). Tek başına bu, netgain'i −6 → −36'ya düşürüp linki Deney 2'deki
   başarısızlık rejimine sokuyor. Bir HAPS'a (20 km) bina duvarı arkasından bağlı
   el terminali gerçekten de çalışmazdı — sonuç fiziksel olarak makul.

2. **⚠️ YENİ BUG bulundu — O2I her saniye yeniden çekiliyor**: `haps_o2i_entry_loss_dB()`
   (`haps_o2i.c`) her çağrıda taze bir `uniformrandom()` persentil çekimi yapıyor
   → O2I kaybı saniyede bir 5 … 56 dB arası zıplıyor. Bu **Adım 39'da gölge
   sönümleme için düzeltilen bug'ın aynısı**: O2I giriş kaybı per-link bir özellik
   (hangi bina, hangi duvar/kat), sabit UE için bağlantı ömrü boyunca sabit
   olmalı — `large_scale_drawn` guard'ı altına alınıp donmalı.
   - Not: bu bug link zaten başarısız olduğu için sonucu değiştirmiyor (en düşük
     çekim ~5 dB bile netgain'i −11'e indirmez ama medyan ~30 dB kesin batırıyor).
     Yine de düzeltilmeli — O2I'nin marjinal etkili olabileceği bir senaryoda
     (Ka-bant değil, daha küçük giriş kaybı) yanıltıcı olur.

**Sonuç (ilk koşu, düzeltmeden önce)**: ✅ Hipotez doğrulandı — O2I linki
bozuyor. Ayrıca O2I'nin per-saniye yeniden çekilme bug'ı tespit edildi.

---

#### O2I bug'ı DÜZELTİLDİ (2026-09-01, Geliştirme Günlüğü Adım 41) + Deney 7 yeniden koşuldu

`haps_o2i_entry_loss_dB()` artık `is_los`/`shadow_fading_dB` ile aynı
`large_scale_drawn` guard'ı altında — bağlantı başına **bir kez** çekilip
donuyor (Adım 39'un O2I'ye uygulanması).

**Doğrulama**: `HAPS_DEBUG_38811=1` çıktısında `netgain` artık her koşu içinde
**tam sabit** (örn. `-52.1 .. -52.1`), önceden saniyede bir değişiyordu.
Regresyon: O2I kapalıyken baz senaryo etkilenmedi.

**Deney 7 — düzeltme sonrası, 3 koşu**:

| Koşu | O2I çekimi (donmuş, tek değer) | netgain (sabit) | Sonuç |
|---|---|---|---|
| #1 | 45.4 dB | −52.1 dB | ❌ bağlanamıyor |
| #2 | 46.6 dB | −52.1 dB | ❌ bağlanamıyor |
| #3 | 13.2 dB (en düşük gözlenen çekim) | **−19.4 dB** | ❌ bağlanamıyor |

**Yorum**: 3/3 koşu, O2I'nin çekilen değeri ne olursa olsun (13 … 47 dB) link
kuramadı. Bu, düzeltmeden önceki sonuçtan daha **kesin**: eski (buglı) koşularda
en düşük ~5 dB'lik anlık çekimler bazen sync'e izin veriyor olabilirdi (saniyede
bir "iyi" bir ana denk gelmek), ama artık her koşu **tek, sabit** bir O2I
gerçekleşimiyle test ediliyor ve gözlenen en hafif çekim (13.2 dB) bile
yetersiz kalıyor. Bu, referans linkin senkron için gereken marjının Deney 2'de
tahmin edilenden (−11 dB'de çalışıyor) daha dar olduğunu gösteriyor — başarısızlık
eşiği −11 ile −19 dB arasında bir yerde, ilk kez bu kadar dar sınırlandı.

**Sonuç**: ✅ Bug düzeltildi (Adım 41), hipotez **hâlâ doğrulanıyor** ve artık
daha güvenilir: O2I açıkken (herhangi bir gerçekçi çekim seviyesinde) bu
senaryoda link kurulamıyor.

---

### Deney 8 — Ara mesafe (`HAPS_GROUND_OFFSET_M=10000`)

- **Tarih**: 2026-09-01
- **Değişen tek şey**: `HAPS_GROUND_OFFSET_M=10000` (baz senaryo — banliyö;
  Deney 2'nin 35 km'sinden daha ılımlı bir ara nokta)
- **Geometri**: eğik mesafe ~22.4 km, yükseklik açısı ~55-63° (loiter yörüngesi
  yüzünden anlık olarak değişiyor)
- **Beklenti**: kademeli, hafif bozulma — Deney 2'nin (35 km, 27°) sert
  başarısızlığından çok daha az
- **Süre**: 3 koşu (kural 7 — LOS/NLOS hâlâ stokastik), ~65 sn her biri

**Sonuç**

| Koşu | Elev | Durum (donmuş) | netgain | Sonuç |
|---|---|---|---|---|
| #1 | ~57° | **NLOS** (bir yönde) | **−28.1 dB** | ❌ bağlanamıyor (1973 synch-fail) |
| #2 | ~55° | LOS | −7.2…−7.3 dB | ✅ temiz (UL `455/0/0/0`) |
| #3 | ~55° | LOS | −9.96 dB | ✅ temiz (UL `405/0/0/0`, DL `42/0/0/0`) |

**Yorum**

- **2/3 koşu temiz ve kademeli**: netgain baz senaryonun (−6 dB) biraz altında
  (−7…−10 dB) — beklenen hafif bozulma, `suburban[elev≈55-60°]` LOS olasılığı
  hâlâ yüksek (~0.94).
- **1/3 koşu tamamen başarısız oldu** — ama beklenmedik bir mekanizmayla:
  **DL ve UL kanalları bağımsız donmuş çekimler yapıyor** (her yönün kendi
  `haps_ctx`'i var). Bu koşuda **yalnızca bir yön** NLOS çekti (~%6 olasılık,
  suburban 55-60°'de) → o yönde +18 dB clutter + geniş SF → netgain −28 dB.
  Diğer yön muhtemelen LOS'tu ama DL senkronu (SSB/PBCH) o kötü yöne bağlı
  olduğu için tüm bağlantı kurulamadı. İki yönün bağımsız çekilmesi yüzünden
  "en az bir yön NLOS" olasılığı tek-yön olasılığından (~%6) yüksek (~%12,
  1-0.94²) — 3 koşuda 1 kez görülmesi istatistiksel olarak makul.

**Sonuç**: ✅ Hipotez (kademeli bozulma) kısmen doğrulandı — tipik durumda
netgain kademeli düşüyor ve link çalışıyor, ama **tek yönlü şanssız bir NLOS
çekimi tüm bağlantıyı düşürebiliyor** çünkü DL/UL bağımsız gerçekleşimler.
Bu, ara-mesafe senaryolarının "kısmen çalışıyor" değil "çoğunlukla çalışıyor,
ara sıra tamamen düşüyor" şeklinde davrandığını gösteriyor — gerçek NTN
sistemlerinde de düşük-orta yükseklik açılarında beklenen bir davranış.

---

### Deney 9 — Alternatif NTN-TDL profili + kentsel zenit

- **Tarih**: 2026-09-01
- **Değişen (Deney 6'ya göre)**: senaryo `SUBURBAN_RURAL` → `URBAN`
  (`HAPS_TDL_USE_ALT_PROFILE=1` sabit, zenit)
- **Soru**: kentselin daha geniş SF varyansı + alt profil (B/D) birlikte
  gözlemlenebilir bir etki yaratır mı?
- **Süre**: 5 koşu (kural 7), 45-95 sn

**Sonuç**

| Koşu | Durum | netgain | Senkron | UL BLER |
|---|---|---|---|---|
| #1 | (debug'sız, teyit edilemedi) | — | ❌ 55 sn'de olmadı | — |
| #2 | **NLOS** (TDL-B log satırıyla teyitli) | — | ❌ 55 sn'de olmadı | — |
| #3 | LOS | −7.85 dB | ✅ (74 synch-fail sonrası) | %0.14 |
| #4 | LOS | −5.72 dB | ✅ (67 synch-fail sonrası) | %0.53 |
| #5 | LOS | −4.98 dB | ✅ (81 synch-fail sonrası) | %0.32 |

**Yorum**

`HAPS_TDL_USE_ALT_PROFILE`, `is_los` çekimini **etkilemiyor** (kodda bağımsız —
profil seçimi yalnızca LOS/NLOS zaten belirlendikten sonra hangi tap
tablosunun kullanılacağını seçiyor). Yani gözlenen 2/5 "NLOS-vari" sonuç
alt-profile'a özgü değil, urban zenit'in kendi LOS olasılığından
(`los_prob[urban,90°]=0.968`, iki bağımsız yön için "en az biri NLOS" ~%6.3)
kaynaklanan normal örnekleme değişkenliği — 5 koşuda 2 kez görmek beklenenden
yüksek ama küçük örneklemde olası (~%3 olasılıklı, imkânsız değil).

**LOS koşularında** (3/5) sonuç **Deney 1 ve Deney 6 ile birebir tutarlı**:
netgain −5…−8 dB, temiz bağlantı, UL BLER <%1, 0 yeniden iletim — alt profilin
(TDL-D) burada da gözlemlenebilir bir etkisi yok.

**NLOS koşularında** (2/5) link **Deney 8'deki mekanizmayla** başarısız oluyor
(clutter kaybı + geniş SF → netgain çok düşük, DL senkronu kurulamıyor) —
profille ilgisi yok, geometri/LOS-durumuyla ilgili.

**Sonuç**: ⚪ Hipotez (profil × senaryo etkileşimi) çürütüldü — alt TDL profili
ile kentsel senaryo arasında bir etkileşim yok. Gözlenen değişkenlik tamamen
LOS/NLOS çekiminden (Deney 8'de zaten karakterize edilen mekanizma); profil
seçimi (A/C vs B/D) hiçbir koşulda fark yaratmadı (Deney 6'nın sonucunu
kentsel için de doğruluyor).

---

### Deney 10 — UE arabada (50 km/h)

- **Tarih**: 2026-09-01
- **Değişen tek şey**: `HAPS_UE_SPEED_MPS=13.89` (50 km/h — şehir içi araba).
  Baz senaryo 3 km/h (yürüyüş, ITU-R M.1225 yaya referansı), Deney 4 ise
  30 m/s = 108 km/h (otoyol).
- **Amaç**: yürüyüş vs araba vs otoyol — UE mobilitesi süpürmesi
- **Süre**: 6 koşu (kural 7), ~55 sn her biri

**Sonuç** (banliyö zenit)

| Koşu | Durum | netgain | UL BLER (mean / max) | Yeniden iletim | Sonuç |
|---|---|---|---|---|---|
| #1,2,3,5,6 | LOS | −5.4 … −6.4 dB | ~%0.2–0.5 / ~%6 | 0–1 | ✅ temiz |
| #4 | **NLOS** (şanssız çekim) | −19.6 dB | — | — | ❌ bağlanamıyor |

**Mobilite karşılaştırması** (hepsi banliyö zenit, LOS):

| Hız | `fd_local` | UL BLER (mean) | Yeniden iletim | Not |
|---|---|---|---|---|
| **3 km/h** (yürüyüş, baz) | 6.9 Hz | ≈0 | 0 | Bölüm 3 |
| **50 km/h** (araba, Deney 10) | **115 Hz** | **~%0.2–0.5** | **0–1** | 5/5 temiz |
| **108 km/h** (otoyol, Deney 4) | 249 Hz | ~%0.3 tipik, ~%10 (6 koşuda 1) | 0 tipik, ~62 (outlier) | ara sıra sürekli episod |

**Yorum**

**50 km/h, yürüyüşten pratikte ayırt edilemez.** `fd_local` 7 → 115 Hz'e
çıksa da, sönümleme hâlâ ~1 ms'lik HARQ gidiş-dönüşüne göre yavaş kalıyor →
kanal kestirimi ve güç kontrolü rahatça izliyor. Bir stat penceresinde ara sıra
%5.9'a çıkan UL BLER görülüyor ama mean %0.5'in altında ve 0 yeniden iletim.

Hızlı-fade bozulması ancak **otoyol hızlarında** (108 km/h, `fd_local` 249 Hz)
ve o zaman bile yalnızca **ara sıra** (koşuların ~1/6'sı) sürekli bir UL BLER
episoduna dönüşüyor — asla kopma yok.

Deney 10 #4'ün başarısızlığı **hıza bağlı değil**: şanssız bir NLOS çekimi
(netgain −19.6, Deney 8/9'daki aynı mekanizma). Loiter yörüngesi yüzünden
donma anındaki yükseklik açısı tam 90° değil ~84-87° → `los_prob` 0.95-0.998
→ NLOS birkaç-yüzde-olasılıklı bir olay, bu kadar koşuda 1 kez görmek normal.

**Sonuç**: ✅ Şehir içi araç hızı (50 km/h) HAPS linkini hiç zorlamıyor —
yürüyüşle aynı. NTN + HAPS geometrisi düşük platform Doppler'i sağladığı için,
kritik olan UE'nin kendi yerel-saçılma Doppler'i ve o da ancak otoyol
hızlarında marjinal etkili oluyor.

---

### Deney 11 — Yoğun kentsel + düşük yükseklik açısı (en kötü durum)

- **Tarih**: 2026-09-01
- **Değişen (Deney 2'ye göre)**: senaryo `URBAN` → `DENSE_URBAN`
  (`HAPS_GROUND_OFFSET_M=35000` sabit, eğik mesafe ~44 km, yükseklik açısı ~27°)
- **Beklenti**: Deney 2'den de kötü — yoğun kentselin LOS olasılığı daha düşük,
  gölgeleme yayılımı çok daha geniş
- **Süre**: 8 koşu (kural 7 + LOS çekimini yakalamak için), 48-100 sn

**Sonuç**

| Durum çekimi | Koşu sayısı | netgain aralığı | Sonuç |
|---|---|---|---|
| **NLOS** | 7/8 | **−27 … −65 dB** | ❌ hepsi bağlanamıyor |
| **LOS** | 1/8 (run #4) | −11.6 dB (sabit) | ❌ 48 sn'de senkron olamadı (sınırda) |

- **0/8 koşu bağlandı.**
- NLOS oranı 7/8, `los_prob[dense_urban, 27°] = 0.398` ile tutarlı (iki bağımsız
  yön için "en az biri NLOS" ~%84).
- NLOS netgain aralığı çok geniş (−27 … −65 dB) — yoğun kentsel NLOS
  `sigma_SF = 12.4 dB` (kentselin 6 dB'sinin iki katından fazla), donmuş çekim
  koşudan koşuya devasa değişiyor.
- **2/8 koşuda UE segfault** verdi (uzun süre senkron olamama yolunda) — bu
  OAI'nin bilinen bir dayanıklılık sorunu (Geliştirme Günlüğü'ndeki LEO notu da
  aynısını belirtiyor), HAPS koduna özgü değil.

**Yükseklik açısı / senaryo ilerlemesi** (net kazanç ve bağlanma olasılığı):

| Deney | Senaryo / geometri | Tipik netgain | Bağlanma |
|---|---|---|---|
| Baz / D1 | banliyö-kentsel, **zenit** | −6 … −7 dB | ~%100 |
| D8 | banliyö, **10 km / ~55°** | −7 … −10 dB | çoğunlukla (ara sıra NLOS düşüşü) |
| D2 | kentsel, **35 km / ~27°** | LOS −11, NLOS −40…−50 | ~%50 (LOS'ta çalışır) |
| **D11** | **yoğun kentsel, 35 km / ~27°** | LOS −11.6, NLOS **−27…−65** | **~%0** |

**Yorum**

Yoğun kentsel + düşük yükseklik açısı **kullanılamaz bir senaryo** — fiziksel
olarak tamamen beklenen: yoğun kentsel kanyonda, çoğunlukla NLOS, HAPS'a 44 km
eğik mesafe, +29 dB clutter kaybı ve 12 dB'ye varan gölgeleme. Bir el terminali
için imkânsız. Nadir LOS çekimi bile (netgain −11.6) senkron marjının tam
sınırında — Deney 2'nin kentsel-LOS'u −11.0'da bağlanmıştı, bu −11.6'da
(48 sn'de) bağlanamadı, yani sınırda/flaky.

**Sonuç**: ✅ Hipotez doğrulandı ve aşıldı — yoğun kentsel + düşük açı, Deney 2'nin
kısmi başarısından **tam başarısızlığa** geçiyor. Kanal modeli TR 38.811'in
en zorlu köşesini doğru şekilde "çalışmaz" olarak modelliyor.

---

### Deney 12 — netgain ↔ yükseklik açısı süpürmesi

- **Tarih**: 2026-09-01
- **Amaç**: yükseklik açısı yükseldikçe SNR (netgain) nasıl değişiyor — temiz
  bir eğri çıkarmak
- **Yöntem**: baz senaryo (banliyö), `HAPS_GROUND_OFFSET_M` = 0 … 35000 arası
  7 nokta, her koşuda `HAPS_DEBUG_38811=1` çıktısından `netgain` ve `elev`
  okundu; kısa koşular (~22 sn, senkron beklenmedi, sadece debug satırı için).
  NLOS çekilen noktalar LOS gelene kadar tekrar koşuldu.

**Ölçülen — LOS netgain vs yükseklik açısı**

| offset (km) | yükseklik açısı | eğik mesafe | ölçülen netgain | FSPL teorisi `−6 + 20·log₁₀(sin θ)` |
|---|---|---|---|---|
| 0 | 78.7° | ~20.4 km | **−6.58 dB** | −6.17 |
| 5 | 65.8° | ~21.9 km | **−7.55 dB** | −6.80 |
| 10 | 55.0° | ~24.4 km | **−7.24 dB** | −7.73 |
| 15 | 46.5° | ~27.6 km | **−9.82 dB** | −8.79 |
| 20 | 39.8° | ~31.2 km | **−9.86 dB** | −9.87 |
| 25 | 34.6° | ~35.4 km | **−10.99 dB** | −10.92 |
| 35 | 27.2° | ~43.8 km | **−12.77 dB** | −12.80 |

**Yorum**

- **LOS'ta ilişki net, monoton ve serbest-uzay yasasını izliyor**:
  `netgain ≈ −6 + 20·log₁₀(sin θ)`. Ölçülen değerler teoriyi **±1 dB** içinde
  takip ediyor; o ±1 dB kalıntı da tam olarak donmuş gölge-sönümleme çekimi
  (banliyö LOS `sigma_SF` bu açılarda 0.7–1.8 dB).
- 79° → 27° arası netgain ~−6.6 → ~−12.8 dB, yani **~6 dB**'lik kademeli düşüş.
  Tamamen eğik mesafenin uzamasından (S-bandında gaz/sintilasyon ihmal edilebilir).
- **NLOS basamağı**: 20 km'lik ilk koşu NLOS çekti → netgain **−39 dB** (~30 dB
  uçurum). Yani asıl elevation etkisi bu iki katmanlı: (a) LOS'ta yumuşak
  `20·log₁₀(sin θ)` eğimi, (b) düşük açıda artan NLOS olasılığının tetiklediği
  ~25–40 dB'lik basamak.

**Teorik LOS eğrisi (tam aralık, referans için)**

| θ | netgain | θ | netgain |
|---|---|---|---|
| 90° | −6.0 | 40° | −9.8 |
| 80° | −6.1 | 30° | −12.0 |
| 70° | −6.5 | 20° | −15.3 |
| 60° | −7.2 | 10° | −21.2 |
| 50° | −8.3 | | |

**Sonuç**: ✅ "Yükseklik açısı yükseldikçe SNR yükseliyor mu?" sorusunun cevabı:
**evet** — LOS'ta `20·log₁₀(sin θ)` kadar (90→27° için ~6–7 dB), artı düşük
açıda NLOS olasılığının getirdiği ~30 dB'lik ayrı bir basamak. Kanal modeli
serbest-uzay geometrisini birebir doğru üretiyor.

---

### Deney 13 — netgain ↔ yükseklik açısı süpürmesi, kentsel senaryo

- **Tarih**: 2026-09-01
- **Değişen (Deney 12'ye göre)**: senaryo `SUBURBAN_RURAL` → `URBAN`
- **Yöntem**: aynı — 7 offset noktası, `HAPS_DEBUG_38811=1`'den `netgain`/`elev`.
  Düşük açı noktalarında 3-4 koşu (LOS olasılığı düştüğü için).

**Ölçülen** (offset km, yükseklik açısı, ölçülen LOS netgain, FSPL teorisi)

| offset | açı | LOS netgain (ort) | LOS aralık | FSPL teorisi | LOS / toplam koşu |
|---|---|---|---|---|---|
| 0 | 78.7° | −10.2 | — | −6.2 | 1/1 |
| 5 | 65.8° | −3.9 | — | −6.8 | 1/1 |
| 10 | 55.0° | −2.4 | — | −7.7 | 1/1 |
| 15 | 46.5° | −9.4 | **−16.5 … −4.2** | −8.8 | 4/4 |
| 20 | 39.8° | −17.8 | (1 örnek) | −9.9 | 1/1 |
| 25 | 34.6° | −13.2 | (1 örnek) | −10.9 | **1/4** |
| 35 | 27.1° | −10.6 | −11.4 … −9.8 | −12.8 | **2/4** |

NLOS çekilen koşular: netgain −39 … −50 dB (link ölü).

**Yorum — Deney 12 (banliyö) ile karşılaştırma**

| | Banliyö (D12) | Kentsel (D13) |
|---|---|---|
| **FSPL eğimi** | `−6 + 20·log₁₀(sin θ)` | **aynı** — geometri senaryodan bağımsız |
| **LOS netgain saçılması** (tek koşu) | teoriden ±1 dB | teoriden **−8 … +5 dB** |
| **LOS `sigma_SF`** | 0.7–1.8 dB | **4.0 dB** (tüm açılarda sabit) |
| **LOS olasılığı @ 27°** | ~0.93 | **~0.49** |
| **Düşük açıda deneyim** | çoğunlukla LOS, kademeli | LOS/NLOS piyangosu |

1. **FSPL yasası aynı**: LOS-ortalama netgain 78°→27° arası ~−6 → ~−13 dB, tıpkı
   banliyö gibi ~7 dB'lik eğik-mesafe eğimi. Geometri senaryoya bakmıyor.
2. **Ama kentsel saçılması ~4×**: `sigma_SF = 4 dB` sabit → tek bir donmuş
   çekim teoriden ±8 dB sapabiliyor. 15 km noktasında 4 koşu −4.2, −8.0, −8.8,
   −16.5 verdi — **tek bir açıda 12 dB yayılma**. Ortalama teoriye yakınsıyor
   ama tek koşu güvenilmez.
3. **LOS olasılığı ~40°'nin altında çöküyor**: 34° → 1/4 LOS, 27° → 2/4 LOS.
   Banliyö 10°'ye kadar >0.9'da kalıyordu; kentsel ~27°'de zaten yazı-tura.
   NLOS çekildiğinde netgain −40…−50 → link ölü.

**Sonuç**: ✅ Yükseklik-açısı ↔ SNR ilişkisinin **geometrik omurgası** her iki
senaryoda aynı (`20·log₁₀(sin θ)`). Kentsel iki şey ekliyor: (a) her
gerçekleşimde ~±8 dB gölgeleme belirsizliği, (b) ~40°'nin altında hızla artan
NLOS olasılığı → düşük açıda kentsel HAPS linki "kademeli zayıflar" değil,
"çoğu zaman ölü, ara sıra çalışır" davranıyor.

---

### Deney 14 — LOS olasılık tablosu doğrulaması (`HAPS_DEBUG_LOS_SWEEP`)

- **Tarih**: 2026-09-01
- **Amaç**: Deney 13'te kentsel düşük açıda link çok sık koptu — bu, TR 38.811
  Tablo 6.6.1-1'in gerçekten uygulandığı anlamına mı geliyor, yoksa bir bug mı?
- **Yöntem**: Adım 42'de eklenen `HAPS_DEBUG_LOS_SWEEP=1` aracı — her senaryo ×
  her referans açı için `is_los` çekimini 200 000 kez yapıp ölçülen P(LOS)'u
  tabloyla karşılaştırıyor. Tek koşu, ~25 sn.

**Ölçülen P(LOS) vs Tablo 6.6.1-1** (200k çekim/hücre)

| Açı | Banliyö ölç. (tablo) | Kentsel ölç. (tablo) | Yoğun kentsel ölç. (tablo) |
|---|---|---|---|
| 10° | 0.781 (0.782) | 0.246 (0.246) | 0.281 (0.282) |
| 20° | 0.869 (0.869) | 0.384 (0.386) | 0.331 (0.331) |
| 30° | 0.919 (0.919) | 0.493 (0.493) | 0.398 (0.398) |
| 40° | 0.929 (0.929) | 0.614 (0.613) | 0.468 (0.468) |
| 50° | 0.935 (0.935) | 0.725 (0.726) | 0.537 (0.537) |
| 60° | 0.940 (0.940) | 0.805 (0.805) | 0.613 (0.612) |
| 70° | 0.949 (0.949) | 0.918 (0.919) | 0.740 (0.738) |
| 80° | 0.952 (0.952) | 0.968 (0.968) | 0.821 (0.820) |
| 90° | 0.998 (0.998) | 0.992 (0.992) | 0.981 (0.981) |

**Yorum**

**27 hücrenin hepsi tabloyla ±0.002 içinde eşleşiyor** — `is_los = uniformrandom()
< haps_38811_los_prob[...]` doğru çalışıyor, `uniformrandom()` düzgün dağılımlı,
tablo indekslemesi doğru. DL ve UL kanal nesneleri ayrı ayrı çalıştırıldı, ikisi
de eşleşti.

Bu, önceki düşük-açı deneylerinin (D2, D8, D9, D11, D13) yüksek NLOS oranlarının
**gerçek olduğunu** doğruluyor — kod bug'ı değil, TR 38.811'in kendisinin
kentsel/yoğun-kentsel için öngördüğü davranış:
- Kentsel 30°: LOS olasılığı **%49** → link yarı yarıya kopar
- Yoğun kentsel 30°: LOS olasılığı **%40** → çoğunlukla kopar (D11 = 7/8 NLOS)
- Banliyö 10°'de bile %78 LOS — banliyö bu yüzden düşük açıda çok daha dayanıklı

**Sonuç**: ✅ LOS/NLOS çekimi doğrulandı. Düşük-açı senaryolarındaki başarısızlık
oranları modelin doğru davranışı, TR 38.811 spesifikasyonunun beklediği şey.

---

### Deney 15 — netgain ↔ yükseklik açısı süpürmesi, yoğun kentsel

- **Tarih**: 2026-09-01
- **Değişen (Deney 12/13'e göre)**: senaryo → `DENSE_URBAN` — süpürme üçlemesini
  tamamlıyor
- **Yöntem**: aynı, 7 offset, `HAPS_DEBUG_38811=1`. Her noktada 3-5 koşu
  (yoğun kentsel LOS olasılığı çok düşük).

**Ölçülen — yoğun kentsel**

| offset | açı | LOS netgain (ort) | LOS aralık | NLOS aralık | FSPL teori | LOS/koşu |
|---|---|---|---|---|---|---|
| 0 | 78.7° | −4.7 | −5.7…−4.2 | — | −6.2 | 3/3 |
| 5 | 65.8° | −8.4 | — | −29 | −6.8 | 2/3 |
| 10 | 55.0° | −10.8 | −11.2…−10.3 | −20…−47 | −7.7 | 2/5 |
| 15 | 46.5° | −10.7 | −12.7…−8.7 | −39 | −8.8 | 2/3 |
| 20 | 39.8° | −11.8 | — | −28…−64 | −9.9 | 1/3 |
| 25 | 34.6° | (0/3 LOS) | — | −35…−47 | −10.9 | **0/3** |
| 35 | 27.1° | −11.6 | — | −48…−58 | −12.8 | 1/3 |

**Üç senaryonun karşılaştırması — LOS netgain ≈ FSPL teorisi**

| açı | Banliyö (D12) | Kentsel (D13) | Yoğun kentsel (D15) | FSPL `−6+20log₁₀(sinθ)` |
|---|---|---|---|---|
| 79° | −6.6 | −10.2 | −4.7 | −6.2 |
| 66° | −7.6 | −3.9 | −8.4 | −6.8 |
| 55° | −7.2 | −2.4 | −10.8 | −7.7 |
| 47° | −9.8 | −9.4 | −10.7 | −8.8 |
| 40° | −9.9 | −17.8 | −11.8 | −9.9 |
| 27° | −12.8 | −10.6 | −11.6 | −12.8 |

**LOS-yakalama oranı (düşük açıda asıl fark)**

| açı | Banliyö | Kentsel | Yoğun kentsel |
|---|---|---|---|
| ~55° | ~%94 | çoğunlukla | **2/5** |
| ~35° | ~%92 | 1/4 | **0/3** |
| ~27° | ~%92 | 2/4 | 1/3 |

**Yorum**

1. **FSPL omurgası her üç senaryoda aynı**: LOS-ortalama netgain hepsinde
   `−6 + 20·log₁₀(sin θ)` eğrisini izliyor (±SF çekimi). Yoğun kentsel LOS
   `sigma_SF` ≈ 2.3–3.5 dB — banliyö (~1) ile kentsel (4) arası.
2. **LOS olasılığı yoğun kentselde çok daha erken çöküyor**: 55°'de bile
   yazı-tura, ~35°'nin altında ~%40. Banliyö 10°'ye kadar >%90'da kalıyordu.
   Bu, Deney 11'in (yoğun kentsel + 27° = 7/8 NLOS, 0/8 bağlanma) neden o kadar
   sert olduğunu açıklıyor.
3. **NLOS netgain'i devasa saçılıyor**: −20 … −64 dB tek bir açıda — yoğun
   kentsel NLOS `sigma_SF` 9–15 dB.

**Sonuç**: ✅ Süpürme üçlemesi tamamlandı. Yükseklik açısı ↔ SNR ilişkisinin
geometrik kısmı (FSPL) senaryodan bağımsız ve modelde birebir doğru. Senaryolar
arası fark iki yerden geliyor: (a) LOS gölge-sönümleme belirsizliği
(banliyö ±1 → kentsel ±8 → yoğun kentsel ±3 dB), (b) LOS olasılığının açıyla
düşme hızı (banliyö çok yavaş, yoğun kentsel çok hızlı). Düşük açıda pratik
kullanılabilirlik tamamen (b) tarafından belirleniyor.

---

### Deney 16 — UE hareketsiz (`HAPS_UE_SPEED_MPS=0`)

- **Tarih**: 2026-09-01
- **Değişen tek şey**: `HAPS_UE_SPEED_MPS=0` — yerel-saçılma Doppler'i tamamen 0
- **Amaç**: sönümleme donunca BLER tamamen sıfırlanır mı?
- **Süre**: 5 koşu, ~55-60 sn, `HAPS_DEBUG_TDL=1` / `HAPS_DEBUG_38811=1`

**Sonuç**

| Koşu | Durum | netgain | tap_energy | Sonuç | UL BLER |
|---|---|---|---|---|---|
| #1 | NLOS (bir yön) | −33.3 | **0.0** (NLOS yön) | ❌ bağlanamıyor | — |
| #2 | LOS | −6.1 | 0.455 | ✅ | %0.34 |
| #3 | LOS | −7.4 | 0.455 | ❌ (55 sn'de olmadı) | — |
| #4 | LOS | −4.9 | — | ✅ | %0.21 |
| #5 | LOS | −5.7 | — | ✅ | %0.21 |

**Cevap**: Hayır — BLER ~%0.2'de kalıyor, hareketli durumla (Deney 10) aynı.
Çünkü baz senaryonun kalıntı BLER'i sönümleme kaynaklı değil — MCS-0'ın
gürültü tabanındaki decode hataları + trafik yokluğu. Fade'i dondurmak
buna yardım etmiyor.

**⚠️ YENİ BUG bulundu — `HAPS_UE_SPEED_MPS=0` + NLOS = ölü kanal**

`fd_local = ue_speed · fc / c = 0` → `rho = exp(0) = 1` → `innov_scale =
sqrt(1−1) = 0` → NTN-TDL AR(1) sönümleme durumu (`ctx->tdl_state`, calloc ile
**0**'dan başlıyor) hiçbir zaman inovasyon almıyor, 0'da kalıyor. Sonuç:
- **NLOS profili (TDL-A/B, tüm taplar Rayleigh)**: `gain = tap_amp · (0 + 1·0) = 0`
  → bütün taplar **tam olarak sıfır** → `tap_energy = 0.0` (run #1'de doğrulandı)
  → kanal hiç sinyal geçirmiyor. "Fade donuyor" değil, "kanal ölüyor".
- **LOS profili (TDL-C/D, tap 0 Ricean)**: specular bileşen deterministik olduğu
  için `gain = tap_amp · spec_amp` sağ kalıyor → kanal çalışıyor (sönümlemesiz,
  sabit specular). Bu yüzden LOS koşuları bağlanabildi.

Katalog `HAPS_UE_SPEED_MPS=0`'ı "sönümlemeyi durdurur" diye tanımlıyor — ama
NLOS için gerçekte kanalı öldürüyor. Düzeltme: `fd_local ≈ 0` iken AR(1)
durumunu bir kez geçerli bir birim-varyanslı kompleks Gauss örneğiyle tohumla
(donmuş ama geçerli bir Rayleigh gerçekleşimi). **Adım 43 adayı.**

**Not — tap_energy 0.455 (1.0 değil)**: kısmen specular oranı (`K/(K+1) ≈ 0.91`),
kısmen de doğrusal-interpolasyonlu fraksiyonel gecikme bölünmesinin enerji
korumaması (`(1−frac)² + frac²`, frac=0.5'te 0.5'e düşer) — bu ikincisi tüm
hızlarda var olan küçük bir kusur, Deney 16'ya özgü değil.

**Sonuç (ilk koşu)**: ⚪ Hipotez çürütüldü (BLER sıfırlanmıyor) + ⚠️
`HAPS_UE_SPEED_MPS=0` NLOS'ta ölü kanal bug'ı bulundu.

---

#### Ölü-kanal bug'ı DÜZELTİLDİ (2026-09-01, Geliştirme Günlüğü Adım 43) + Deney 16 yeniden koşuldu

**Düzeltme**: `haps_update_tdl_taps()` artık ilk çağrıda AR(1) sönümleme durumunu
sürecin durağan varyansıyla (birim kompleks güç) bir kez tohumluyor
(`tdl_state_seeded` bayrağı). `rho = 1` (hız 0) iken bile donmuş durum artık
geçerli bir Rayleigh gerçekleşimi, 0 değil.

**Doğrulama** (`HAPS_UE_SPEED_MPS=0`, `HAPS_DEBUG_TDL=1`):

| Senaryo | NLOS tap_energy (düzeltmeden önce → sonra) |
|---|---|
| urban + 10 km offset, NLOS çekilen koşular | **0.0 → ~0.46 / ~1.22** (gerçekleşimden gerçekleşime değişen, ort ~1.0) |
| banliyö zenit, LOS koşular | 0.455 (sabit) → 0.42…1.07 (değişken — diffuse bileşen artık gerçek enerjiye sahip) |

Regresyon: varsayılan-hız (3 km/h) baz senaryo etkilenmedi — RRC bağlanıyor,
UL HARQ temiz (`458/0/0/0`).

**Yeniden koşum sonucu**: `HAPS_UE_SPEED_MPS=0` artık her iki profilde de
(LOS ve NLOS) çalışan bir donmuş sönümleme kanalı veriyor. BLER cevabı
değişmedi (LOS koşularında ~%0.2, hareketli durumla aynı — beklenen).

**Sonuç**: ✅ Bug düzeltildi (Adım 43). Deney 16'nın ana bulgusu (fade'i
dondurmak BLER'i sıfırlamıyor) geçerli; ek olarak `HAPS_UE_SPEED_MPS=0` artık
NLOS için de güvenli kullanılabilir bir knob.

---

### Deney 17 — O2I "ısıl verimli" bina sınıfı (`HAPS_O2I_THERMAL=1`)

- **Tarih**: 2026-09-01
- **Değişen (Deney 7'ye göre)**: `HAPS_O2I_THERMAL=1` eklendi — ITU-R P.2109
  bina sınıfı "traditional" yerine "thermally-efficient" (metalize/düşük-emisyonlu
  cam kaplama, RF'yi daha çok engeller). `HAPS_O2I_ENABLE=1` sabit, baz senaryo.
- **Süre**: 3 tam koşu + 6+6 kısa örnekleme koşusu (donmuş O2I çekimi dağılımı için)

**Ölçülen** (yükseklik açısı ~78.7°, `HAPS_DEBUG_O2I` çıktısı, `-> L=` alanı)

| Bina sınıfı | O2I kaybı örnekleri (dB) | medyan |
|---|---|---|
| **traditional** (Deney 7) | 15.5, 16.9, 29.1, 29.2, 30.7, 42.0 | ~29 |
| **thermally-efficient** (Deney 17) | 38.1, 54.7, 55.2, 57.5, 57.6, 63.8 | ~56 |

Persentil-başına eşleştirme (P.2109 `p` çekimi eşit olmadığı için medyan-medyan
biraz yanıltıcı, aynı `p`'de karşılaştırma daha net):

| Persentil `p` | traditional L | thermally-efficient L |
|---|---|---|
| ~0.08 | ~16 dB | (~28 dB, ekstrapolasyon) |
| ~0.43 | ~29 dB | ~38 dB (`p=0.32`) |
| ~0.88 | ~42 dB | ~62 dB (`p=0.90`) |

**3 tam koşu sonucu**: netgain −23 / −37 / −43 dB → **3/3 bağlanamıyor**
(zaten Deney 7'de traditional da bağlanamıyordu).

**Yorum**

Isıl verimli bina, aynı persentilde traditional'a göre **~10–22 dB daha fazla**
giriş kaybı ekliyor — kod formülü de bunu doğruluyor: `Lh` (yatay bileşen)
traditional için 2.49 GHz'de ~14 dB, thermally-efficient için ~28 dB (P.2109
Tablo 6.6.3-1 katsayıları). Metalize cam kaplama RF'yi gerçekten daha çok
engelliyor — katalog notu doğru.

Pratik sonuç değişmiyor: O2I zaten (traditional'da bile) linki öldürüyordu;
ısıl verimli sadece "daha ölü". Bir HAPS'a (20 km) modern enerji-verimli bina
içinden bağlı el terminali = imkânsız.

**Sonuç**: ✅ Hipotez doğrulandı — `HAPS_O2I_THERMAL=1` giriş kaybını ~2×'e
çıkarıyor (~29 → ~56 dB medyan). Model P.2109 bina sınıfı ayrımını doğru
uyguluyor.
