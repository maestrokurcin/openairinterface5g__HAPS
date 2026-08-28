# HAPS Kanal Modeli — Mimari Referans

Bu doküman, `HAPS_GELISTIRME_GUNLUGU.md`'nin kronolojik (adım adım, 35 giriş) günlüğünden **farklı** bir amaca hizmet ediyor: "ne zaman, hangi sırayla yapıldı" değil, **"şu an mimari nasıl, bir parça neyi temsil ediyor, hangi referansa dayanıyor"** sorularına hızlı cevap vermek için. Bir sorun çıktığında önce buraya bak; detaylı gerekçe/test geçmişi için günlüğe (`Adım N`) yönlendirme var.

## 1. Genel bakış

HAPS (High Altitude Platform Station, ~20 km irtifada süzülen bir platform — uydudan çok daha alçak, kuleden çok daha yüksek) kanal modeli, OAI'nin `rfsimulator`'una eklenmiş yeni bir kanal tipi ailesi. Referans olarak zaten var olan `SAT_LEO_TRANS` (LEO uydu) modelinin genel iskeleti (gecikme/Doppler/yol kaybı hesaplama düzeni) kullanıldı, ama fizik tamamen HAPS'a özgü: 3GPP TR 38.811 (NTN kanal modeli spesifikasyonu) + birkaç ITU-R tavsiye kararı.

**Sekiz `SCM_t` enum değeri** var (`openair1/SIMULATION/TOOLS/sim.h`):

| Enum | Platform | Fizik |
|---|---|---|
| `HAPS_STATIONARY` | sabit (station-keeping) | artık gerçek TR 38.811 (Adım 27) |
| `HAPS_MOBILE` | hareketli (2 km yörünge) | artık gerçek TR 38.811 (Adım 27) |
| `HAPS_STATIONARY_38811` / `_URBAN` / `_DENSE_URBAN` | sabit | TR 38.811, senaryo seçilebilir |
| `HAPS_MOBILE_38811` / `_URBAN` / `_DENSE_URBAN` | hareketli | TR 38.811, senaryo seçilebilir + küçük-ölçekli sönümleme |

**"_38811" eki artık gerçek fizik/keyfi-değer ayrımını değil, sadece mekan senaryosunu (kırsal/banliyö vs kentsel vs yoğun kentsel) belirtiyor** (Adım 27) — hepsi gerçek TR 38.811 yol kaybı ve kTB+NF gürültü tabanı kullanıyor. Tek fark: küçük-ölçekli (TDL) sönümleme hâlâ sadece `_38811` ailesinde (`enable_small_scale_fading` bayrağı, Adım 27).

## 2. Dosya mimarisi

```
openair1/SIMULATION/TOOLS/          (SIMU kütüphanesi — genel, NR_MAC'a bağımlı değil)
├── sim.h                            struct/enum tanımları: channel_desc_t, haps_channel_ctx_t, SCM_t
├── random_channel.c                 kanal OLUŞTURMA (bağlantı başına 1 kez) - case HAPS_STATIONARY*/HAPS_MOBILE*
├── haps_config.c                    SCM_t enum → haps_channel_ctx_t varsayılanları
├── haps_geometry.c                  saf loiter-dairesi kinematiği (pozisyon/hız, t'nin fonksiyonu)
├── haps_propagation.c               TR 38.811 büyük-ölçekli yol kaybı (FSPL+SF+CL), gas/rain/scint/o2i'yi çağırır
├── haps_gas.c                       ITU-R P.676 atmosferik gaz kaybı
├── haps_rain.c                      ITU-R P.838-3 yağmur kaybı
├── haps_scint.c                     TR 38.811 troposferik sintilasyon (dar Ka-bant yer tutucu)
├── haps_o2i.c                       ITU-R P.2109 bina girişi kaybı (opt-in)
└── haps_tdl.c                       TR 38.811 küçük-ölçekli sönümleme (NTN-TDL-A/B/C/D) + MIMO korelasyonu

radio/rfsimulator/                  (rfsimulator kütüphanesi — NR_MAC_gNB'ye bağımlı, SIB19 için)
├── haps_channel.h/.c                haps_channel_process() - HER TAMPON döngüsünde çalışan orkestratör
└── apply_channelmod.c               dispatcher (HAPS mi değil mi?) + rxAddInput() (TÜM modeller ortak, FIR uygulaması)
```

**Neden `haps_channel.c` diğerlerinden ayrı bir klasörde?** Çünkü `nr_update_sib19()`'u çağırıyor (SIB19/NTN yayını için) — bu, düşük seviyeli `SIMU` kütüphanesinin alamayacağı bir bağımlılık (`NR_MAC_gNB`'ye bağlı). Diğer tüm `haps_*.c` dosyaları saf fizik/matematik, hiçbir NR-özel bağımlılığı yok.

**Paylaşılan `channel_desc_t` struct'ı nasıl etkilendi?** SCM_A/EPA/TDL_A-E/AWGN/SAT_LEO_TRANS/REGEN gibi HAPS-dışı TÜM modeller de bu struct'ı kullanıyor. Değişiklik minimal tutuldu: sadece **tek bir yeni alan** eklendi: `haps_channel_ctx_t *haps_ctx` (HAPS-dışı modeller için hep `NULL`, sıfır etki). Eski 9 ayrı HAPS-özel alan (`haps_loiter_radius` vb.) bu yeni struct'ın içine taşındı.

## 3. Veri akışı

### 3.1. Bağlantı kurulumu (bir kez)

```
.conf dosyası (channelmod bloğu)
        │
        ▼
random_channel.c: case HAPS_STATIONARY*/HAPS_MOBILE*
        │  - sat_height, loiter_radius/platform_speed ayarlanır
        │  - haps_config_new() çağrılır → haps_ctx oluşturulur
        │  - channel_length: 1 (STATIONARY) / 2 (MOBILE, TDL yok) / 16 (MOBILE, TDL var)
        │  - HAPS_STATIONARY* için: path_loss_dB VE noise_power_dB burada TEK SEFERLİK hesaplanır
        │    (çünkü STATIONARY hiçbir zaman "her tampon döngüsü" adımına girmiyor - aşağıya bak)
        ▼
channel_desc_t (+ haps_ctx) hazır
```

### 3.2. Her tampon döngüsü (sadece dinamik modeller)

```
apply_channelmod.c: update_channel_model()
        │
        │  KAPI: sat_height>0 VE (enable_dynamic_delay VEYA enable_dynamic_Doppler)
        │  ⚠️  HAPS_STATIONARY*'de İKİSİ DE false → bu yol HİÇ ÇALIŞMAZ (bilinçli, Adım 7:
        │      RA/Msg3 kararsızlaşıyordu, sabit gecikmeye geri dönüldü)
        ▼
haps_channel.c: haps_channel_process()
        │
        ├─ haps_geometry.c: pozisyon/hız (t anındaki loiter-dairesi konumu)
        ├─ gecikme/Doppler hesabı (gerçek geometriden, uplink/downlink ayrı)
        ├─ haps_update_frac_delay_taps()  (channel_length==2 ise: sadece fraksiyonel gecikme)
        ├─ haps_tdl.c: haps_update_tdl_taps()  (enable_small_scale_fading ise: NTN-TDL sönümleme)
        ├─ haps_propagation.c: haps_38811_path_loss_dB()  (use_38811_pathloss ise)
        │     ├─ haps_gas.c, haps_rain.c, haps_scint.c, haps_o2i.c çağrılır
        └─ nr_update_sib19()  (uplink dalında, 10ms'de bir - Earth-centered çerçeve dönüşümüyle)
        ▼
channel_desc_t.ch[] (FIR filtresi) + path_loss_dB + noise_power_dB güncel
        ▼
apply_channelmod.c: rxAddInput()  (TÜM modeller ortak - HAPS'a özel değil)
        │  ch[] ile gerçek IQ örneklerini konvolüe eder, path_loss_dB'yi uygular, gürültü ekler
        ▼
UE/gNB'nin aldığı gerçek sinyal
```

**Kritik nokta**: `random_channel()` bağlantı boyunca **sadece bir kez** çalışır (`simulator.cpp`'de `reGenerateChannel` sabit `false`) — yani "zamanla değişen" her şey (gecikme, Doppler, sönümleme, yol kaybı) `haps_channel_process()`'in HER TAMPON döngüsünde yeniden hesaplaması gerekiyor, `random_channel()`'da değil.

## 4. Bileşen → referans tablosu

Her fizik teriminin **hangi spesifikasyona dayandığı** ve **hangi dosyada olduğu**:

| Bileşen | Dosya | Referans | Referansın söylediği / kullanılan tablo |
|---|---|---|---|
| Büyük-ölçekli yol kaybı (FSPL+SF+CL) | `haps_propagation.c` | **3GPP TR 38.811 V15.1.0 §6.6.1/§6.6.2** | Tablo 6.6.1-1 (LOS olasılığı), Tablo 6.6.2-1/2/3 (gölgeleme sönümü + saçaklanma kaybı), S/Ka-bant, LOS/NLOS |
| Atmosferik gaz kaybı | `haps_gas.c` | **ITU-R P.676 Ek 2** | Basitleştirilmiş gaz sönümü formülü (oksijen+su buharı), ITU-Rpy açık kaynak kütüphanesinden port edildi |
| Yağmur kaybı | `haps_rain.c` | **ITU-R P.838-3** | Tam resmi formül, Tablo 1-4 katsayıları (kH/kV/αH/αV), dairesel polarizasyon varsayımı |
| Troposferik sintilasyon | `haps_scint.c` | **3GPP TR 38.811 §6.6.6.2** | Tablo 6.6.6.2.1-1 (Toulouse/20GHz referans) - sadece >6GHz'de (spec'in kendi kuralı) |
| O2I bina girişi kaybı | `haps_o2i.c` | **3GPP TR 38.811 §6.6.3 → ITU-R P.2109** | İki-lognormal-karışımlı model, Tablo 6.6.3-1 (traditional/thermally-efficient) katsayıları |
| Gecikme yayılımı (DS) ölçekleme | `haps_tdl.c` | **3GPP TR 38.811 §6.7.2** | Tablo 6.7.2-1a..8a (S-bant) / -1b..8b (Ka-bant), mean lgDS satırı |
| Küçük-ölçekli sönümleme (multipath) | `haps_tdl.c` | **3GPP TR 38.811 §6.9.2** | Tablo 6.9.2-1..4 (NTN-TDL-A/B/C/D: tap gecikme/güç/K-faktör) |
| MIMO uzamsal korelasyonu | `haps_tdl.c` | *(dış referans yok — OAI'nin kendi `R_sqrt_22_EPA_medium`/`R_sqrt_21_corr`'u yeniden kullanıldı)* | `random_channel.c`'de zaten var olan, EPA modeli için kullanılan matrisler |
| Yerel saçılma Doppler'i (`fd_local`) | `haps_tdl.c` | **Jakes/Clarke formülü** (`f_d=v·fc/c`) + **ITU-R M.1225** (varsayılan hız) | 3 km/h "pedestrian" referans hızı |
| Zaman avansı (timing advance) / SIB19 | `haps_channel.c` | **3GPP NTN `ntn-Config-r17`/SIB19** (zaten OAI'de var olan mekanizma) | HAPS'ın kendi dinamik geometrisiyle senkronize tutuluyor (Earth-centered çerçeve dönüşümü) |
| Fraksiyonel (alt-örnek) gecikme | `haps_channel.c` | *(genel DSP tekniği — spec referansı yok)* | 2-tap doğrusal enterpolasyon, konvolüsyon formülüyle matematiksel olarak kanıtlandı |
| kTB+NF gürültü tabanı | `random_channel.c` | **3GPP TR 38.811 Tablo 4.4-1** (Adım 36) | `-174 + 10·log10(BW) + NF`, NF=9dB — S-bant el cihazı/IoT UE için gerçek spec değeri |

**Önemli**: `HAPS_38811_SIM_CALIBRATION_DB` (114.93 dB) tek kalan sayı — rfsimulator gerçek dBW/dBm birimleriyle çalışmadığı için, gerçek EIRP'yi (TR 38.811 Ek A, 36dBW/MHz) simülatörün iç genlik ölçeğine bağlayan kaçınılmaz bir köprü sabiti (Adım 36). Bu köprü *sabit bir referans bant genişliğinde* (5MHz) hesaplanıyor — test senaryosunun gerçek bant genişliğine göre ölçeklendirmek denendi ama genlik taşmasına (clipping) yol açtığı için geri alındı.

## 5. Ayar anahtarları (ortam değişkenleri)

Hiçbiri kalıcı bir config alanı değil — hepsi bilinçli olarak **opt-in, env-var-only** (varsayılan davranışı bozmamak için):

| Değişken | Ne yapar | Varsayılan |
|---|---|---|
| `HAPS_GROUND_OFFSET_M` | Terminali kapsama alanının merkezinden uzağa koyar (düşük yükseklik açısı testi) | 0 (tam tepede, ~90°) |
| `HAPS_RAIN_RATE_MM_H` | Yağmur oranı | 0 (açık hava) |
| `HAPS_O2I_ENABLE` | O2I bina girişi kaybını açar | kapalı |
| `HAPS_O2I_THERMAL` | "thermally efficient" bina sınıfı (yoksa "traditional") | traditional |
| `HAPS_TDL_USE_ALT_PROFILE` | NTN-TDL-A/C yerine B/D kullan | kapalı (A/C) |
| `HAPS_UE_SPEED_MPS` | `fd_local` için varsayılan UE hızını değiştir | 0.833 (3km/h) |
| `HAPS_DEBUG_38811` | Yol kaybı hesap izini logla | kapalı |
| `HAPS_DEBUG_TDL` | TDL tap/korelasyon izini logla | kapalı |
| `HAPS_DEBUG_O2I` | O2I hesap izini logla | kapalı |
| `HAPS_DEBUG_FRACDELAY` | Fraksiyonel gecikme izini logla | kapalı |

## 6. Sorun giderme — nerede bakmalı

| Belirti | Muhtemel sebep | Bak |
|---|---|---|
| Bağlantı hiç kurulmuyor, yol kaybı/gürültü hiç değişmiyor | `enable_dynamic_delay`/`enable_dynamic_Doppler` ikisi de false, veya `.conf`'ta `options=("chanmod")` eksik | `random_channel.c` (case bloğu), `.conf`'un `rfsimulator` bölümü |
| Yol kaybı değeri saçma/beklenmedik | Yanlış senaryo (`haps_ctx->scenario`), yanlış S/Ka-bant ayrımı, ya da `use_38811_pathloss=false` | `haps_propagation.c`, `haps_config.c` |
| Gecikme/Doppler yanlış | Geometri hesabı (loiter yarıçapı/hız/irtifa) ya da uplink/downlink karışıklığı | `haps_geometry.c`, `haps_channel.c` |
| Sönümleme çok hızlı/yavaş evriliyor | `fd_local`/UE hızı, `HAPS_UE_SPEED_MPS` ile kontrol et | `haps_tdl.c` (`HAPS_UE_SPEED_DEFAULT_MPS`) |
| Sönümleme hiç yok / TDL etkisiz | `enable_small_scale_fading=false` (sadece `_38811` ailesinde true) ya da `channel_length` yanlış | `haps_config.c`, `random_channel.c`'nin `channel_length` ataması |
| MIMO (2x2) tap'ları garip/korelasyonsuz görünüyor | `n_pairs` yanlış hesaplanmış, ya da yanlış korelasyon matrisi seçilmiş | `haps_tdl.c` (`haps_R_sqrt_22_medium`/`haps_R_sqrt_21_corr`, `n_pairs` mantığı) |
| Asimetrik MIMO (2x1/1x2) RRC'ye hiç ulaşmıyor | **HAPS'a özel değil** - `random_channel.c`'nin `load_channellist()`'i uplink/downlink modellerine yön-bazlı olmayan aynı nb_tx/nb_rx veriyor (bkz. Adım 35) | bilinçli olarak açık bırakıldı, düzeltilmedi |
| SIB19/TA bilgisi UE'de tutarsız | Earth-centered çerçeve dönüşümü (`radius_earth` offset) | `haps_channel.c`'nin SIB19 güncelleme bloğu |
| Yeni bir HAPS enum'u beklenmedik davranıyor | `haps_config_new()`'deki varsayılan atamalar | `haps_config.c` |
| `librfsimulator.so` değişikliklerimi yansıtmıyor gibi | Sadece `ninja nr-softmodem` çalıştırılmış, `rfsimulator` unutulmuş | her zaman `ninja rfsimulator nr-softmodem nr-uesoftmodem` birlikte |

## 7. Bilinen sınırlar (özet)

Detaylı gerekçe/test kayıtları için `HAPS_GELISTIRME_GUNLUGU.md`'nin ilgili Adım'ına bakın:

- `fd_local` artık gerçek UE hızından türetiliyor (Adım 33) → ara sıra gerçekçi bağlantı kesilmesi (bilinçli kabul edildi)
- Ka-bant DS tablo seçimi canlı test edilmedi (Adım 32, kanıtlanmış Ka-bant senaryo yok)
- MIMO: 1x1/2x2 tam çalışıyor, 2x1/1x2 kod-seviyesinde doğrulandı ama RRC'ye ulaşmıyor (paylaşılan OAI sınırı, Adım 35), 4x4 hiç yok
- Resmi Docker imaj zinciri ve `oai-amf` hiç test edilmedi (Adım 34, sadece hafif özel bir eşdeğer doğrulandı)
- İyonosferik sintilasyon eklenmedi (Adım 22, orta enlemde ~0 varsayıldı)
- `SAT_LEO_TRANS/REGEN` bu makinede hiç doğrulanmadı (bu proje HAPS'a odaklı)
