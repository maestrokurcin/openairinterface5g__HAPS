# HAPS Kanal Modeli Geliştirme Günlüğü

Bu dosya, OpenAirInterface5G (OAI) `rfsimulator`'a yeni bir **"HAPS"** (High Altitude
Platform Station, ~20 km irtifa) kanal modeli eklenmesi çalışmasının tam kaydıdır.
Referans olarak mevcut `SAT_LEO_TRANS` (LEO uydu) kanal modeli kullanılmıştır.
Bundan sonra yapılacak her adım da (denendi/çalıştı, denendi/hata alındı fark etmeksizin)
kronolojik olarak bu dosyanın sonuna eklenecektir.

Proje kökü: `/home/furkan/openairinterface5g` (ana OAI reposu, henüz commit'lenmemiş
değişiklikler içeriyor) + `/home/furkan/openairinterface5g/haps_test/` (ayrı, kendi git
reposu olan test/konfig klasörü).

---

## 1. Genel Özet

Amaç: `rfsimulator`'ın kanal modelleme alt sistemine (`openair1/SIMULATION/TOOLS/`)
`SAT_LEO_TRANS`/`SAT_LEO_REGEN` ile aynı mimaride yeni bir `HAPS` tipi eklemek, gNB
ile UE arasındaki rfsim bağlantısını bu yeni kanaldan geçirmek ve üzerine gerçekçi
fiziksel etkiler (yol kaybı, gürültü, sabit yayılım gecikmesi) eklemek.

Test senaryosu bilinçli olarak **NTN protokol yığınından (SIB19 / `ntn_Config_r17`)
bağımsız** tutuldu: Band 78 (karasal), 106 PRB, numeroloji 1 kullanılıyor. Amaç önce
saf kanal modelinin (path loss/gürültü/gecikme) doğru çalıştığını kanıtlamak; NTN
protokolüyle (band 254 + SIB19) birleştirme ayrı, sonraki bir adım.

## 2. Dosya Envanteri

### 2.1 Ana repoda (`openairinterface5g/`) değiştirilen **orijinal** dosyalar

| Dosya | Durum |
|---|---|
| `openair1/SIMULATION/TOOLS/sim.h` | Değiştirildi (bu oturumdan önce yapılmış, bu günlükte tespit edilip doğrulandı) |
| `openair1/SIMULATION/TOOLS/random_channel.c` | Değiştirildi (aynı şekilde) |
| `targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.pci0.rfsim.conf` | Değiştirildi (aynı şekilde, `haps_test` ile ilgisiz bir referans dosyası — bkz. Adım 3 notu) |
| `openair1/PHY/NR_UE_TRANSPORT/nr_ntn_l1.c` | Değiştirildi (önceki oturumdan kalma `HAPS_DEBUG_TA`/`HAPS_DEBUG_NO_NTN_TA`; bu oturumda Adım 9'da `HAPS_DEBUG_APPLY_NTN_CONFIG` ve `HAPS_FIX_KOFFSET_PERSIST` eklendi - hepsi varsayılan no-op) |
| `openair2/LAYER2/NR_MAC_UE/nr_ue_procedures.c` | Değiştirildi (aynı şekilde, `HAPS_DEBUG_MSG3`) |
| `openair2/LAYER2/NR_MAC_gNB/gNB_scheduler_ulsch.c` | Değiştirildi (aynı şekilde) |
| `ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn-haps.conf` | Önceki oturumdan kalma (untracked, yeni dosya); bu oturumda Adım 10'da `ta-Common-r17` düzeltildi (kök neden fix'i) |
| `ci-scripts/conf_files/nrue.uicc.ntn-haps.conf` | Önceki oturumdan kalma (untracked, yeni dosya); bu oturumda değiştirilmedi |

### 2.2 `haps_test/` klasöründeki dosyalar

| Dosya | Durum |
|---|---|
| `haps_test/gnb.haps.conf` | Önceden vardı, bu oturumda düzeltildi |
| `haps_test/haps_channel.conf` | Önceden vardı, **kullanılmıyor** (bkz. Adım 2 notu) |
| `haps_test/docker-compose.yaml` | Önceden vardı, **henüz kullanılmadı/düzeltilmedi** (bkz. Adım 2 notu) |
| `haps_test/nrue.haps.conf` | Bu oturumda yeni oluşturuldu (`HAPS_STATIONARY`, çalışıyor) |
| `haps_test/gnb.haps_mobile.conf` | Bu oturumda yeni oluşturuldu (`HAPS_MOBILE`, band78/NTN'siz - Adım 8'in PRACH Ncs penceresi sorunu yüzünden RA/RRC kurulamıyor) |
| `haps_test/nrue.haps_mobile.conf` | Bu oturumda yeni oluşturuldu (aynı şekilde) |
| `haps_test/gnb.haps_mobile_ntn.conf` | Bu oturumda yeni oluşturuldu (`HAPS_MOBILE` + gerçek NTN/SIB19, Adım 13 - **çalışıyor**) |
| `haps_test/nrue.haps_mobile_ntn.conf` | Bu oturumda yeni oluşturuldu (aynı şekilde) |
| `haps_test/HAPS_GELISTIRME_GUNLUGU.md` | Bu dosya, bu oturumda yeni oluşturuldu |

---

## 3. Değişiklik Günlüğü (kronolojik)

### Adım 1 — `HAPS` kanal tipinin C koduna eklenmesi (bu oturumdan önce yapılmıştı, incelenip doğrulandı)

**[Dosya]** `openair1/SIMULATION/TOOLS/sim.h`
```c
// satır ~222-224, enum SCM_t içine:
SAT_LEO_TRANS,
SAT_LEO_REGEN,
HAPS,                       // + Eklendi

// satır ~260-262, CHANNELMOD_MAP_INIT (isim <-> enum eşleşme tablosu) içine:
{"SAT_LEO_TRANS",SAT_LEO_TRANS},
{"SAT_LEO_REGEN",SAT_LEO_REGEN},
{"HAPS",HAPS},              // + Eklendi
```
**# Gerekçe:** Config dosyalarında `type = "HAPS";` yazılabilmesi için önce bir enum
değeri ve isim-eşleşme kaydı gerekiyor — `SAT_LEO_TRANS` ile birebir aynı mekanizma.

**[Dosya]** `openair1/SIMULATION/TOOLS/random_channel.c`
```c
// satır ~1663, new_channel_desc_scm() içindeki switch(channel_model)'a yeni case:
case HAPS:
  nb_taps = 1;
  Td = 0;
  channel_length = 1;
  ricean_factor = 0.0;
  aoa = 0.0;
  maxDoppler = 0;
  chan_desc->sat_height = 20e3;             // 20 km irtifa
  chan_desc->enable_dynamic_delay = false;  // sabit yayılım gecikmesi
  chan_desc->enable_dynamic_Doppler = false;// HAPS sabit/loiter -> Doppler yok
  fill_channel_desc(...);
  printf("HAPS platform height: %f km (Static altitude)\n", chan_desc->sat_height/1000);
  break;

// satır ~1793, identity-matrix kısayolu (AWGN benzeri kanallar için):
if (desc->modelid == AWGN || desc->modelid == SAT_LEO_TRANS
    || desc->modelid == SAT_LEO_REGEN || desc->modelid == HAPS) { ... }

// satır ~2098, debug/telnet çıktısı:
if (cd->modelid == SAT_LEO_TRANS || cd->modelid == SAT_LEO_REGEN || cd->modelid == HAPS)
  prnt("satellite orbit height: %f\n", cd->sat_height);
```
**# Gerekçe:** `enable_dynamic_delay=false` ve `enable_dynamic_Doppler=false` bilinçli
seçim: `radio/rfsimulator/apply_channelmod.c`'deki `update_channel_model()` fonksiyonu
bu iki bayraktan biri `true` olmadıkça Kepler yörünge formülüne hiç girmiyor
(`apply_channelmod.c:13-15`). O formül LEO'nun gerçek orbital hızına göre kurulu ve
20 km irtifada anlamsız sonuç (yine ~7.9 km/s ilk kozmik hız) üretir — bu yüzden HAPS
için o kod yoluna hiç girilmeyecek şekilde tasarlandı. `apply_channelmod.c` dosyasına
bu adımda dokunulmadı.

**Sonuç:** Kod derlendi, çalıştırıldığında `HAPS platform height: 20.000000 km` satırı
log'da görüldü — kanal modeli parse ediliyor ve yükleniyor.

---

### Adım 2 — `haps_test/` konfig dosyalarındaki hataların tespiti ve düzeltilmesi

Mevcut `haps_test/gnb.haps.conf`, `haps_test/haps_channel.conf` ve
`haps_test/docker-compose.yaml` incelendiğinde HAPS kanalının fiilen **hiç devreye
girmediği** görüldü, 3 sorun tespit edildi:

**[Dosya]** `haps_test/gnb.haps.conf`
```
~ Değiştirildi (rfsimulator bloğu):
- options = (); #("saviq"); or/and "chanmod"
+ options = ("chanmod"); #("saviq"); or/and "chanmod"
```
**# Gerekçe:** `options` boş olduğu sürece `rfsimulator` kanal modellemeyi hiç
uygulamıyor (bkz. `radio/rfsimulator/simulator.cpp:553-566`) — yol kaybı/gürültü
tanımlı olsa bile sinyale hiç dokunulmuyordu.

**[Bilgi]** `haps_test/haps_channel.conf` ve `docker-compose.yaml`'daki
`--conf_file /opt/oai-gnb/etc/haps_channel.conf` bayrağı **geçersiz** — `nr-softmodem`
CLI'sinde böyle bir seçenek yok (repo genelinde arandı, bulunamadı). Ayrıca
`docker-compose.yaml`'daki UE komutunda `--rfsimulator.options chan_mod` yazıyor,
doğrusu alt çizgisiz `chanmod` olmalı (`simulator.cpp:566`). Bu Docker akışı **henüz
düzeltilmedi/test edilmedi** — şu ana kadarki tüm testler `nr-softmodem`/`nr-uesoftmodem`
binary'lerini doğrudan çalıştırarak (bare-binary) yapıldı. Docker'a geçilecekse bu iki
hata ayrıca giderilmeli.

**[Dosya]** `haps_test/nrue.haps.conf` — **yeni oluşturuldu**
```
+ Eklendi (tüm dosya): uicc0, rfsimulator (options=("chanmod")), channelmod bloğu
  (rfsimu_channel_enB0 + rfsimu_channel_ue0, ikisi de type="HAPS")
```
**# Gerekçe:** `channelmod` bir liste-tipi parametre olduğundan CLI bayrağıyla
(`--channelmod.model_name ...`) verilemiyor, mutlaka bir `-O` config dosyasında tam
olarak tanımlanmalı — `ci-scripts/conf_files/nrue.uicc.ntn-leo.conf` şablonu izlendi.
UE tarafında böyle bir dosya hiç yoktu.

**Sonuç:** Bu değişikliklerle ilk uçtan uca test denendi (bkz. Adım 3).

---

### Adım 3 — İlk uçtan uca test: `min_rxtxtime` hatası

**Komutlar:**
```bash
cd cmake_targets
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-softmodem -O ../haps_test/gnb.haps.conf --rfsim
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-uesoftmodem -O ../haps_test/nrue.haps.conf -r 106 --numerology 1 -C 3619200000 --rfsim
```

**Sonuç: BAŞARISIZ.** PBCH/SIB1/PRACH/RAR/Msg3 hepsi başarıyla geçti (`Found RAR with
the intended RAPID`, `RA-Msg3 transmitted`), ama UE kısa süre sonra assert ile çöktü:
```
Assertion (feedback_ti >= GET_DURATION_RX_TO_TX(&mac->phy_config.config_req.ntn_config, dlsch_pdu->SubcarrierSpacing)) failed!
In nr_ue_process_dci_dl_10() openair2/LAYER2/NR_MAC_UE/nr_ue_procedures.c:1144
PDSCH to HARQ feedback time (2) needs to be higher than DURATION_RX_TO_TX (3).
```
**Kök neden:** `common/utils/nr/nr_common.h:51`'de `NR_UE_CAPABILITY_SLOT_RX_TO_TX (3)`
sabit tanımlı. `gnb.haps.conf`'ta `min_rxtxtime` hiç ayarlanmadığı için gNB, UE'nin
gerçekte destekleyemeyeceği kadar kısa (2 slot) bir DL→UL HARQ geri bildirim zamanlaması
seçmişti. Bu, HAPS kanal modeliyle **ilgisiz**, standart bir OAI TDD/zamanlama
konfigürasyon sorunu (aynı düzeltme daha önce ana repodaki referans
`gnb.sa.band78.fr1.106PRB.pci0.rfsim.conf` dosyasına da uygulanmış olarak bulundu).

**[Dosya]** `haps_test/gnb.haps.conf`
```
~ Değiştirildi:
    gNB_ID    =  0xe00;
    gNB_name  =  "gNB-OAI";
+   min_rxtxtime=6;
```
**# Gerekçe:** `min_rxtxtime`'ı `NR_UE_CAPABILITY_SLOT_RX_TO_TX (3)`'ün üzerine
çekerek gNB'nin geçerli bir HARQ zamanlama seçmesini sağlamak.

---

### Adım 4 — İkinci uçtan uca test: BAŞARILI

**Aynı komutlarla tekrar çalıştırıldı. Sonuç: BAŞARILI.**
```
PUSCH with TC_RNTI 0xfec9 received correctly
Received RRCSetupComplete (RRC_CONNECTED reached)
... sürekli DL/UL HARQ trafiği, SINR = 47 dB, CQI = 15 ...
```
Kanal modeli aktifken (path loss 20 dB, gürültü -110 dBFS) bağlantı sağlıklı ve kararlı.
Bu, `HAPS` kanal tipinin gNB↔UE trafiğinden fiilen geçtiğinin ve bağlantıyı
bozmadığının kanıtı.

**Not:** Bu aşamada `sat_height=20e3` alanı doldurulsa da `enable_dynamic_delay=false`
olduğundan gerçek 20 km'lik fiziksel gecikme/Doppler henüz simüle edilmiyordu — sadece
yol kaybı + gürültü uygulanıyordu.

---

### Adım 5 — Gerçek 20 km sabit yayılım gecikmesinin eklenmesi

`rfsimulator`'ın zaten var olan `prop_delay` parametresi kullanıldı (yeni C kodu
gerekmedi — bu parametre, sabit/istasyon-tutan HAPS için "GEO-style" yaklaşımın
(bkz. proje notları, Faz 1) doğal karşılığı; `radio/rfsimulator/simulator.cpp:73,102`).
Birim ms, ve `chan_offset = ceil(sample_rate * prop_delay_ms / 1000)` ile örneğe
çevrilip **her uç kendi alım tarafında** uyguluyor (`simulator.cpp:1604-1608`) — yani
gNB ve UE'ye aynı tek-yön değeri yazılırsa toplamda gidiş-dönüş gecikmesi 2 katı olur.

Hesap: 20 km / ışık hızı (299 792 458 m/s) = 0.0000667128 s = **0.06671 ms** (tek yön).

**[Dosya]** `haps_test/gnb.haps.conf`
```
~ Değiştirildi (rfsimulator bloğu):
  options = ("chanmod");
+ prop_delay = 0.06671; # 20 km / c tek yön sabit gecikme (ms)
```

**[Dosya]** `haps_test/nrue.haps.conf`
```
~ Değiştirildi (rfsimulator bloğu):
  options = ("chanmod");
+ prop_delay = 0.06671; # 20 km / c tek yön sabit gecikme (ms)
```
**# Gerekçe:** `apply_channelmod.c`'ye dokunmadan (Kepler formülü riskinden kaçınarak,
bkz. Adım 1 gerekçesi), rfsimulator'ın kendi native, kanal-modelinden bağımsız gecikme
mekanizmasıyla gerçek fiziksel mesafeyi modellemek.

**Test komutları:** Adım 3/4 ile aynı.

**Sonuç: BAŞARILI.**
```
[HW] propagation delay 0.066715 ms, 4099 samples   (hem gNB hem UE loglarında)
PUSCH with TC_RNTI 0x9db1 received correctly
Received RRCSetupComplete (RRC_CONNECTED reached)
```
Gecikme her iki uçta da doğru örnek sayısına (4099 örnek ≈ 0.0667 ms @ ~61.44 Msps'lik
efektif örnekleme hızında) çevrilerek uygulandı; RACH/RRC bağlantısı hâlâ sorunsuz
kuruldu (gidiş-dönüş toplamı ~8198 örnek ≈ 0.1334 ms, HARQ zamanlama toleransının
içinde kaldı).

---

### Adım 6 — `ploss_dB` alanının ne anlama geldiğinin incelenmesi, gerçek FSPL hesabı ve denenmesi

Kullanıcı isteği: gerçek FSPL (Free Space Path Loss / serbest uzay yol kaybı) değerini
hesaplayıp `ploss_dB`'ye yazmak. Önce bu alanın motor tarafında **tam olarak nasıl
kullanıldığı** koddan okundu, çünkü isim yanıltıcı çıktı.

**`ploss_dB` / `path_loss_dB` gerçekte ne anlama geliyor (kod okuması):**

`radio/rfsimulator/apply_channelmod.c:216-260` (`rxAddInput()`), sinyale kanal
etkisini uygulayan asıl yer. İçindeki yorum satırı açık:
```c
// channelDesc->path_loss_dB should contain the total path gain
// so, in actual RF: tx gain + path loss + rx gain (+antenna gain, ...)
const double pathLossLinear = pow(10, channelDesc->path_loss_dB / 20.0);
...
out_ptr->r += rx_tmp.r * pathLossLinear + noise_per_sample * gaussZiggurat(0.0, 1.0);
```
Yani `path_loss_dB` (config'teki adıyla `ploss_dB`) **saf bir kayıp değeri değil**,
sinyale doğrudan çarpan olarak uygulanan **net bağlantı kazancı**dır:
`toplam_kazanç_dB = tx_kazancı - gerçek_FSPL + rx_kazancı (+anten kazançları...)`.
Formülde işaret pozitif (`+path_loss_dB/20`), yani config'e **pozitif** bir sayı
yazarsanız sinyal güçlenir, **negatif** yazarsanız zayıflar. Bu yüzden önceki
adımlardaki `ploss_dB = 20.0` değeri aslında "20 dB'lik net kazanç" demekti — gerçek bir
20 km bağlantının kaybı değil, keyfi/iyimser bir varsayılan test değeriydi. rfsim'in bu
basit yolu, gerçek dBm cinsinden kalibre edilmiş bir TX gücü/RX kazancı modellemiyor
(kod içindeki `// Fixme: not sure when it is "volts" so dB is 20*log10(...)` yorumu da
bunu doğruluyor) — normalize edilmiş, "ortalama genlik 256" varsayımına dayalı, göreceli
bir birim.

**Gerçek FSPL hesabı (20 km, taşıyıcı frekans 3619.2 MHz = testte kullanılan `-C`
değeri):**
```
FSPL(dB) = 20*log10(d_km) + 20*log10(f_MHz) + 32.44
         = 20*log10(20) + 20*log10(3619.2) + 32.44
         = 26.02 + 71.17 + 32.44
         = 129.64 dB
```
(Doğrulama: `20*log10(4*pi*d_m*f_Hz/c)` formülüyle de aynı sonuç, 129.64 dB.)

**[Dosya]** `haps_test/gnb.haps.conf` ve `haps_test/nrue.haps.conf`
```
~ Değiştirildi (her iki model, her iki dosya):
- ploss_dB = 20.0;
+ ploss_dB = -129.64; # gercek FSPL, saf kayip olarak (kazanc terimi eklenmeden)
```
**# Gerekçe:** Yukarıdaki işaret kuralına göre saf bir fiziksel kaybı temsil etmek için
FSPL negatif işaretle yazıldı (`-129.64`), üzerine herhangi bir tx/rx anten kazancı
eklenmedi (gerçek bir HAPS bağlantısında olması gereken onlarca dB'lik anten
kazancı bilinçli olarak dahil edilmedi, çünkü kullanıcı isteği sadece "gerçek FSPL"
değeriydi).

**Test edildi. Sonuç: BAŞARISIZ — bağlantı tamamen koptu.**
```
[PHY] [UE thread Synch] Running Initial Synch
[PHY] synch Failed:
[PHY] [UE thread Synch] Running Initial Synch
[PHY] synch Failed:
... (30 saniye boyunca sürekli tekrar, hiç PSS/SSB/PBCH tespiti yok) ...
```
UE, PSS/SSB'yi bir kere bile tespit edemedi — yani `pathLossLinear = 10^(-129.64/20)
≈ 3.3×10⁻⁷` çarpanı, sinyali gürültü tabanının çok altına düşürdü. Bu beklenen bir
sonuç: gerçek bir 20 km RF bağlantısının kapanabilmesi için (gerçek anten kazançları,
TX gücü olmadan) 130 dB'e yakın saf bir kaybı hiçbir link telafi edemez; gerçek
sistemlerde bu kayıp onlarca dB'lik yönlü anten kazancıyla (VSAT/phased-array tipi
antenler) dengelenir, ama rfsim'in bu basit `chanmod` yolunda böyle bir kazanç terimi
ayrı ayrı modellenmiyor — hepsi tek bir `ploss_dB` sayısına sıkıştırılmış durumda.

**[Dosya]** `haps_test/gnb.haps.conf` ve `haps_test/nrue.haps.conf`
```
~ Değiştirildi (geri alındı, her iki model, her iki dosya):
- ploss_dB = -129.64;
+ ploss_dB = 20.0; # NOT: gercek FSPL (-129.64 dB) denendi, baglanti koptu
```
**# Gerekçe:** Test ortamını çalışır durumda bırakmak için önceki bilinen-iyi değere
geri dönüldü. Geri dönüş sonrası tekrar test edildi, sonuç: **BAŞARILI**
(`PUSCH ... received correctly`, `RRCSetupComplete`, `RRC_CONNECTED`).

**Açık konu / sonraki adım için not:** Saf FSPL değerini kullanmak isteniyorsa, buna
gerçekçi bir tx+rx anten/kazanç bütçesi eklenmesi gerekiyor
(`ploss_dB = tx_kazancı_dB - FSPL_dB + rx_kazancı_dB`), yoksa bağlantı hiç kurulamıyor.
Bu tx/rx kazanç değerleri henüz belirlenmedi/eklenmedi — kullanıcı onayı bekleniyor.

---

### Adım 7 — "Case" ayrımı: `HAPS_STATIONARY` (sabit) ve `HAPS_MOBILE` (hareketli) olarak ikiye bölme

Kullanıcı isteği: tx/rx anten kazancı konusunu (Adım 6'nın açık konusu) şimdilik bir kenara
bırakıp, "daha gerçekçi simülasyon" için bir sonraki aşamaya geçmek — spesifik olarak:
tek bir `HAPS` tipi yerine **sabit (station-keeping)** ve **hareketli (loiter)** HAPS'ı iki
ayrı "case" olarak modellemek, ve gerçek NTN protokolüyle (SIB19) birleştirmeyi
kademeli/sonraya bırakmak.

**Tasarım:** `SAT_LEO_TRANS`/`SAT_LEO_REGEN`'in iki ayrı enum/tip olması gibi, tek `HAPS`
tipi ikiye bölündü: `HAPS_STATIONARY` ve `HAPS_MOBILE`. İkisi de aynı yeni geometrik
modeli kullanıyor, tek fark hız/yarıçap parametreleri:
- Yer istasyonu (gNB ve UE, `apply_channelmod.c`'nin LEO modelinde de zaten yaptığı gibi
  tek noktaya indirgenmiş) yerel düzlemsel (flat-earth) koordinatlarda orijinde (0,0,0).
  20 km'lik irtifada Dünya eğriliği HAPS'ın loiter yarıçapına (birkaç km) göre ihmal
  edilebilir, bu yüzden LEO'nun küresel Dünya modeli yerine düz yerel model kullanıldı.
- Platform, merkezi yer istasyonundan tam olarak kendi yarıçapı kadar ofsetli bir yatay
  daire üzerinde uçuyor — böylece bir turda tam tepe noktasından (zenith) geçiyor, bu da
  gerçekçi, zamanla değişen bir eğik mesafe (slant range) ve Doppler üretiyor (yer
  istasyonu tam dairenin merkezinin altında olsaydı mesafe hep sabit kalır, Doppler hep 0
  olurdu — dejenere bir durum).
- `HAPS_STATIONARY`: loiter yarıçapı = 0 → platform sabit şekilde tam tepede, konum/hız
  zamandan bağımsız, Doppler her zaman tam 0 (station-keeping).
- `HAPS_MOBILE`: loiter yarıçapı = 2000 m, hız = 27.78 m/s (~100 km/h, ITU/3GPP'nin HAPS
  için öngördüğü 0-200 km/h aralığında temsili bir değer) → zamanla değişen mesafe/Doppler.

**[Dosya]** `openair1/SIMULATION/TOOLS/sim.h`
```
~ Değiştirildi (enum SCM_t ve CHANNELMOD_MAP_INIT):
- HAPS,
+ HAPS_STATIONARY,
+ HAPS_MOBILE,

+ Eklendi (channel_desc_t struct'a):
  double haps_loiter_radius;   // HAPS_MOBILE: loiter dairesi yarıçapı (m)
  double haps_platform_speed;  // HAPS_MOBILE: platform hızı (m/s)
```
**# Gerekçe:** Tek bir `HAPS` yerine iki enum, config dosyasında `type = "HAPS_STATIONARY"`
veya `type = "HAPS_MOBILE"` yazılarak seçilebilsin diye — `SAT_LEO_TRANS`/`SAT_LEO_REGEN`
ile birebir aynı desen.

**[Dosya]** `openair1/SIMULATION/TOOLS/random_channel.c`
```
~ Değiştirildi: eski tek `case HAPS:` bloğu, iki ayrı case ile değiştirildi
  (`case HAPS_STATIONARY:` ve `case HAPS_MOBILE:`), her biri kendi
  haps_loiter_radius/haps_platform_speed/enable_dynamic_delay/enable_dynamic_Doppler
  değerlerini set ediyor (bkz. yukarıdaki tasarım notu ve aşağıdaki test sonuçları için
  hangi bayrakların sonunda hangi değerde kaldığı).
~ Değiştirildi: identity-matrix kısayolu (~satır 1829) ve debug/telnet çıktısı
  (~satır 2134) artık `HAPS_STATIONARY`/`HAPS_MOBILE` ikisini de kontrol ediyor.
```

**[Dosya]** `radio/rfsimulator/apply_channelmod.c`
```
~ Değiştirildi: update_channel_model() fonksiyonu, konum/hız hesaplamasını modelid'e göre
  iki dala ayıracak şekilde yeniden yapılandırıldı:
  - if (modelid == HAPS_STATIONARY || modelid == HAPS_MOBILE): yukarıda tarif edilen
    düz-yerel loiter-dairesi modeli (yeni kod).
  - else: mevcut LEO Kepler yörünge modeli (değişmeden korundu, sadece aynı ortak
    pos_sat_*/vel_sat_*/pos_ue_* değişkenlerine yazacak şekilde taşındı).
  is_uplink alt bloğundaki `if (modelid == SAT_LEO_TRANS)` ile korunan ikinci-sekme
  (satellite->gNB) mesafe hesabı dokunulmadan bırakıldı - zaten sadece SAT_LEO_TRANS'ta
  çalışıyor, HAPS_STATIONARY/HAPS_MOBILE bu ikinci sekmeyi hiç görmüyor (HAPS tek-sekme:
  gNB'nin platform üzerinde olduğu SAT_LEO_REGEN'e benzer bir yapı - tasarım gereği).
~ Değiştirildi: SIB19/`ntn_Config_r17` güncelleme çağrısı (`nr_update_sib19`) artık
  sadece `modelid == SAT_LEO_TRANS || modelid == SAT_LEO_REGEN` için çalışıyor, HAPS
  modelleri için atlanıyor (kullanıcının "NTN protokolüne kademeli entegre edelim"
  isteğine uygun - bu adım SIB19'a hiç dokunmuyor, o ayrı, sonraki bir adım).
```
**# Gerekçe:** `radius_earth`, `radius_sat`, `w_sat` gibi LEO'ya özgü değişkenler fonksiyon
genelinde (LEO dalının dışında da) kullanıldığı için üst kapsama taşındı (yoksa derleme
hatası: "undeclared identifier" - bu hata gerçekten alındı, aşağıda not edildi).

**Derleme:** İlk denemede `ninja rfsimulator nr-softmodem nr-uesoftmodem` çalıştırıldığında
**derleme hatası alındı**:
```
apply_channelmod.c:112:35: error: 'radius_sat' undeclared (first use in this function)
apply_channelmod.c:112:52: error: 'w_sat' undeclared (first use in this function)
```
Sebep: `radius_sat`/`w_sat`, `is_uplink` bloğunun içindeki `if (modelid==SAT_LEO_TRANS)`
alt bloğunda (5s/10s ileri projeksiyon hesabı için) tekrar kullanılıyor, ama onları
LEO dalının içinde `const` tanımlamıştım - o dal bitince kapsam dışına çıkıyorlardı. Düzeltme:
`radius_sat`/`w_sat`'ı da `radius_earth` gibi fonksiyonun en üstünde (0.0 varsayılanla)
tanımlayıp LEO dalında sadece atama yapacak şekilde değiştirdim. **Bu düzeltmeden sonra
derleme başarılı oldu.**

**Test 1 — `HAPS_STATIONARY`, `enable_dynamic_delay=true` (ilk deneme):** Config'ler
`type = "HAPS_STATIONARY"` olarak güncellendi, elle girilen `rfsimulator.prop_delay` config
satırları kaldırıldı (artık kanal modelinin kendisi hesaplıyor varsayımıyla). Test edildi:
```
[HW] Downlink delay 0.066713 ms, Doppler shift SAT->UE 0.000000 kHz   (beklenen değerle uyumlu)
```
Gecikme değeri doğru hesaplandı (Adım 5'in elle girilen 0.06671 ms'ine neredeyse birebir
eşit) ve Doppler doğru şekilde tam 0. **Ama RA/bağlantı sonucu: BAŞARISIZ** — 89 kez
`RA failed at state WAIT_Msg3`, hiç `RRCSetupComplete` yok. UE tarafında RAPID eşleşmesi
büyük ölçüde çalışıyordu (24/25) ve Msg3 fiilen gönderiliyordu, ama gNB hiçbir zaman
başarıyla decode edemedi (Adım 4/5'te tamamen sorunsuz çalışan aynı senaryo, tek fark
gecikmenin statik config yerine dinamik/çalışma-zamanında hesaplanması).

**Kök neden (netleştirilmedi, sadece gözlem):** Statik `rfsimulator.prop_delay` config'te
verildiğinde bağlantı kurulmadan önce, sabit bir değer olarak devrededir. Dinamik modda
ise `channelDesc->channel_offset` başlangıçta 0'dır ve ilk gerçek trafik aktığında (RA
sürecinin tam ortasında) `update_channel_model()` ilk kez çağrılıp ~4099 örneğe atlar —
yani bağlantının en kritik, en hassas anında (PRACH/RAR/Msg3 zamanlamasının henüz
oturmadığı an) ani bir gecikme sıçraması oluyor olabilir. Bu, projenin önceki LEO/HAPS NTN
araştırmasında (`oai-leo-to-haps-adaptation` hafıza notu, Round 8) bulunan "tek seferlik,
karmaşık, sıralamaya bağımlı geçiş anında küçük ama kalıcı bir zamanlama kayması oluşuyor"
deseniyle örtüşüyor, ama bu sefer NTN/SIB19 katmanında değil, doğrudan rfsimulator'ın
kendi `channel_offset` mekanizmasında. Ayrıca aynı hafıza notunda, LEO'nun kendi
`enable_dynamic_delay=true` yolunun bu ortamda hiçbir zaman güvenilir çalıştığı
doğrulanamamıştı (Round 3: LEO senkronize bile olamadan segfault etmişti) — yani bu genel
mekanizma muhtemelen HAPS'a özgü değil, rfsimulator'ın dinamik-gecikme yolunda önceden
var olan, bu ortamda daha önce hiç tam teşhis edilmemiş bir kırılganlık. **Kök neden
teşhisi yapılmadı** (kapsamlı bir PHY-seviyesi debug gerektirir, önceki Msg3 araştırma
maratonuyla aynı büyüklükte bir efor olabilir) — kullanıcı onayı olmadan bu araştırmaya
girilmedi.

**Karar:** `HAPS_STATIONARY` için pratik ve kanıtlanmış çözüme geri dönüldü — sabit bir
platform için mesafe zaten değişmediğinden dinamik hesaplamanın hiçbir gerçek kazancı
yok. `enable_dynamic_delay`/`enable_dynamic_Doppler` tekrar `false` yapıldı, config
dosyalarına statik `prop_delay = 0.06671` geri eklendi.

**[Dosya]** `openair1/SIMULATION/TOOLS/random_channel.c`
```
~ Değiştirildi (case HAPS_STATIONARY):
- chan_desc->enable_dynamic_delay = true;
+ chan_desc->enable_dynamic_delay = false;  # geri alindi, asagidaki test sonucuna bkz.
```
**[Dosya]** `haps_test/gnb.haps.conf`, `haps_test/nrue.haps.conf`
```
+ Eklendi (geri eklendi): rfsimulator.prop_delay = 0.06671;
```
**Test 2 — `HAPS_STATIONARY`, geri alınmış hâliyle tekrar test edildi. Sonuç: BAŞARILI**
(0 `RA failed`, `PUSCH ... received correctly`, `RRCSetupComplete`, `RRC_CONNECTED`) —
Adım 4/5'teki kanıtlanmış duruma tam olarak geri dönüldü.

**Test 3 — `HAPS_MOBILE` (yeni, ayrı test config'leriyle):** `haps_test/gnb.haps_mobile.conf`
ve `haps_test/nrue.haps_mobile.conf` **yeni oluşturuldu** (gnb.haps.conf/nrue.haps.conf'un
birer kopyası, `type = "HAPS_MOBILE"`, `prop_delay` yok - kanal modeli hesaplıyor). Test
edildi:
```
[HW] Downlink delay 0.068034 ms, Doppler shift SAT->UE 0.000301 kHz   (t=~0)
[HW] Downlink delay 0.068033 ms, Doppler shift SAT->UE 0.002127 kHz   (birkaç saniye sonra)
```
Fizik doğru: gecikme ~0.068 ms civarında (20 km + 2 km loiter yarıçapından beklenen
mertebe), Doppler zamanla büyüyor (27.78 m/s hız için teorik maksimum Doppler ≈ v/c × f_c
≈ 333 Hz - gözlenen küçük, artan değerler bu üst sınırın altında, tutarlı). **Ama bağlantı
sonucu: HAPS_STATIONARY'nin ilk denemesiyle aynı şekilde tamamen BAŞARISIZ** — 114 kez
`RA failed`, 0 `received correctly`, `RRC_CONNECTED` hiç yok. Bu, sorunun harekete/Doppler'e
değil, doğrudan `enable_dynamic_delay=true`'nun kendisine (yukarıdaki Test 1'de tespit
edilen aynı mekanizmaya) bağlı olduğunu doğruluyor — `HAPS_MOBILE` bu bayrağı gerçek
zamanla-değişen mesafe yüzünden `false` yapamıyor (statik `prop_delay` bir hareketli
platform için fiziksel olarak yanlış olur), yani bu regresyon `HAPS_MOBILE` için şu an
**kaçınılmaz ve çözülmemiş** durumda.

**Güncel durum özeti:**
- `HAPS_STATIONARY` (`gnb.haps.conf` + `nrue.haps.conf`): **çalışıyor**, kanıtlanmış,
  statik `prop_delay` kullanıyor.
- `HAPS_MOBILE` (`gnb.haps_mobile.conf` + `nrue.haps_mobile.conf`, **yeni dosyalar**):
  kod tamamlandı ve derleniyor, kinematik hesap (mesafe/Doppler) doğru çalıştığı
  loglardan doğrulandı, **ama RA/RRC bağlantısı hiç kurulamıyor** - bilinen, henüz kök
  nedeni teşhis edilmemiş açık bir hata. Kullanıcı onayıyla devam edilirse bir sonraki
  adım bu spesifik hatanın (dinamik `channel_offset`'in bağlantı kurulurken neden
  kararsızlığa yol açtığının) teşhisi olmalı.

---

### Adım 8 — Kök neden teşhisi: sorun dinamik mekanizmada değil, gecikmenin BÜYÜKLÜĞÜNDE; PRACH'ın Ncs-sınırlı arama penceresi tespit edildi

Kullanıcı isteği: Adım 7'nin bıraktığı açık soruyu (neden `enable_dynamic_delay=true`
RA/Msg3'ü bozuyor) teşhis etmeye devam etmek. Tüm testler `/tmp/haps_diag/` altındaki
geçici kopyalarla yapıldı, `haps_test/` içindeki çalışan dosyalara dokunulmadı.

**Belirleyici test — hipotez: "dinamik" mi yoksa "gecikme büyüklüğü" mü sorumlu?**
`apply_channelmod.c`'ye hiç dokunmadan (yani `enable_dynamic_delay=false` kalırken),
sadece kanal modelinin **statik** `offset` alanını (config'teki `channelmod` bloğundaki
`offset = 0;` satırı, `sim.h`'deki `CHANNELMOD_MODEL_CO_PNAME`) elle `4099`'a çekip
`HAPS_STATIONARY`'nin çalışan (statik `prop_delay` kullanan) config'iyle test edildi.
**Sonuç: aynı şekilde tamamen BAŞARISIZ** (100 `RA failed`, 0 `received correctly`) —
`apply_channelmod.c`'nin dinamik kodu hiç devrede değilken bile aynı bozulma oluyor.
**Bu, Adım 7'nin "dinamik mekanizma bozuyor" teşhisinin YANLIŞ olduğunu kanıtlıyor** —
gerçek sorumlu, gecikmenin nasıl ayarlandığı değil, doğrudan **büyüklüğü**.

Ayrıca bu test, `rfsimulator.prop_delay` config alanının (Adım 5'ten beri "çalışıyor"
sandığımız statik gecikme) kanal modeli aktifken (`options=("chanmod")`) aslında **hiç
uygulanmadığını** da ortaya çıkardı: `radio/rfsimulator/simulator.cpp:1307-1321`'de asıl
örnek seçimi her zaman `ptr->channel_model->channel_offset`'i kullanıyor,
`prop_delay`'den türeyen cihaz-seviyesi `t->chan_offset` sadece kanal modeli YOKKEN
(satır 1329) veya arabellek saklama boyutu hesabında (satır 1518, sadece üst sınır
olarak) kullanılıyor. Yani Adım 4/5/7'nin "başarılı" testlerinde fiilen uygulanan
gecikme hep **0** idi — `prop_delay`'in kendisi gerçek gecikmeyi hiç etkilemiyordu,
sadece başlangıç log satırında görünüyordu.

**Eşik değerini bulmak için ikili arama** (aynı statik `offset` alanı, `HAPS_STATIONARY`
config'i, `zeroCorrelationZoneConfig=13`, `prach_ConfigurationIndex=98`):

| `offset` (örnek, 61.44 Msps'de) | Tek yön gecikme | Sonuç |
|---|---|---|
| 0, 50, 100, 200 | 0 - 3.26 µs | ✅ 0 RA failed, bağlantı kuruluyor |
| 300, 350, 400, 450 | 4.88 - 7.32 µs | ❌ tamamen başarısız (58-82 RA failed) |
| 500, 1000, 2000, 4099 | 8.14 µs - 66.7 µs | ❌ tamamen başarısız (76-131 RA failed) |

**Keskin bir eşik var, kademeli bir bozulma değil** — 200 örnekte kusursuz, 300 örnekte
tamamen kopuk. Bu, bir SNR/kalite sorunundan çok bir **sabit pencere/sınır** aşılmasına
işaret ediyor.

**Kod izi — kök neden adayı: PRACH korelatörünün `zeroCorrelationZoneConfig`'den türeyen
gecikme arama penceresi.** `openair1/PHY/NR_TRANSPORT/nr_prach.c:602` içindeki
`for (int i = 0; i < NCS2; i++)` döngüsü, gNB'nin PRACH'ı hangi gecikme aralığında
arayacağını `NCS2` ile sınırlıyor. `NCS2`, `zeroCorrelationZoneConfig`'ten
(`openair2/LAYER2/NR_MAC_COMMON/nr_mac_common.c:548` `get_NCS()`) türüyor. Bizim
`prach_ConfigurationIndex=98` **kısa preamble formatı** kullanıyor (format>3), bu da
`NCS_unrestricted_delta_f_RA_15[]` tablosunu kullanıyor
(`nr_mac_common.c:81`: `{0,2,4,6,8,10,12,13,15,17,19,23,27,34,46,69}`) — `zeroCorrelationZoneConfig=13`
için `Ncs=34`. `nr_prach.c:602`'deki `NCS2 = (Ncs<<8)/139 ≈ 62` bin.

Her bin'in fiziksel süresi `nr_prach.c:626-628`'deki dönüşüm formülünden çıkarılabilir
(format>3, `numerology_index=1`): `bin_süresi = (2048/2^mu)/256 örnek @ 30.72 Msps =
8/2^mu = 4 örnek @ 30.72 Msps = 130.2 ns`; bizim gerçek örnekleme hızımızda (61.44 Msps,
30.72'nin 2 katı) bu **8 örnek/bin**'e karşılık geliyor. Yani toplam arama penceresi:
`62 bin × 8 örnek/bin ≈ 496 örnek` — **yukarıdaki tabloda bulduğumuz 200-300 örnekli
ampirik eşikle aynı büyüklük mertebesinde ve tutarlı** (pencere muhtemelen simetrik/iki
yönlü kullanıldığından ölçülen eşik ham NCS2 değerinin yaklaşık yarısı civarında çıkıyor).

**`zeroCorrelationZoneConfig=15` (maksimum yasal değer, `Ncs=69`) ile de tekrar test
edildi** (`offset=4099` sabit kalarak): `NCS2 = (69<<8)/139 ≈ 127` bin `× 8 örnek ≈ 1016`
örneklik pencere - **hâlâ 4099 örnekten çok küçük**, bu yüzden **hiç iyileşme
gözlenmedi** (126 `RA failed`, 0 başarı) - kısa formatta ZCZ'yi maksimuma çekmek bile
20 km'lik bir mesafeyi karşılamaya yetmiyor.

**Sonuç / kök neden:** Bu OAI/rfsim kurulumunda (kısa PRACH formatı, `numerology=1`),
PRACH korelatörünün doğru şekilde çözebildiği azami yayılım gecikmesi kabaca birkaç
yüz örnek (~yüzlerce ns - birkaç µs, ~yüzlerce metre - ~1-1.5 km eşdeğeri) ile sınırlı -
bu sınır **`zeroCorrelationZoneConfig`'in yasal maksimum değerinde bile** 20 km'lik bir
HAPS bağlantısının gerçek gecikmesini (4099 örnek/66.7 µs) karşılamaya **yetmiyor**. Eşik
aşıldığında, PRACH'ın tespit ettiği gecikme (`out.max_preamble_delay`) yanlış/sarmalanmış
oluyor, bundan türeyen RAR TA komutu da yanlış oluyor, ve bunun sonucunda Msg3 gNB'nin
dinlediği pencerenin tamamen dışına düşüyor ("no signal" - toplam sinyal yokluğu, kısmi
bir SNR/kalite sorunu değil, gNB tarafında `no signal` sayısının UE'nin gerçekten Msg3
gönderdiği durum sayısıyla birebir eşleştiği doğrulandı).

**Bunun anlamı:** Bu, sadece bir config ayarıyla düzeltilebilecek bir hata değil - kısa
PRACH formatının Ncs penceresi **tasarım gereği** küçük hücreler için optimize edilmiş.
3GPP'nin NTN çözümü tam olarak bunun için var: SIB19'un `ta-Common-r17` alanı, UE'ye
PRACH'ı göndermeden ÖNCE toplam yayılım gecikmesini **açık-çevrim (open-loop)** olarak
önceden telafi etmesini söylüyor - böylece PRACH gNB'ye fiilen ulaştığında geriye kalan
(rezidüel) gecikme normal Ncs penceresinin içinde kalıyor. Bu da bu projenin başından
beri (`oai-leo-to-haps-adaptation` hafıza notu) her NTN/HAPS test denemesinde
`ntn_Config_r17`/SIB19'un neden hep birlikte kullanıldığını açıklıyor - isteğe bağlı bir
ek değil, **20 km mertebesindeki gerçek gecikmelerle PRACH'ın çalışabilmesi için zorunlu
bir bağımlılık**. Yani bu adımın başında "kademeli, NTN'ye sonra entegre edelim" diye
ayrı tutulan iki konu (gerçekçi mesafe/gecikme simülasyonu ile NTN protokolü) aslında
birbirinden bağımsız değilmiş - gerçekçi mesafe testi yapabilmek için SIB19'un açık-çevrim
ön-telafisine ihtiyaç var.

**Denenen ama sonuçsuz kalan bir doğrulama:** Kısa formattan uzun formata
(`prach_ConfigurationIndex=0`, `prach_RootSequenceIndex_PR=1`, 839-uzunluklu dizi, çok
daha büyük Ncs tablosu) geçilirse pencerenin yeterince büyüyüp büyümeyeceğini görmek için
denendi. **Sonuç: kullanılamaz/sonuçsuz** - bu değişiklikle UE `offset=0`'da (gecikme
hiç yokken) bile hiç senkronize olamadı (32 kez `synch Failed`), yani uzun format için
sadece bu iki alanı değiştirmek yetmiyor - muhtemelen `msg1_SubcarrierSpacing` veya PRACH
occasion/slot eşleşmesiyle ilgili başka alanların da uyumlu şekilde değiştirilmesi
gerekiyor. Bu, ayrı ve doğru şekilde ele alınması gereken bir iş - bu oturumda daha fazla
denenmedi.

**Genel değerlendirme:** Kök neden artık net ve kod-seviyesinde doğrulanmış: **vanilla
(NTN'siz) PRACH, HAPS ölçeğindeki (20 km) gerçek gecikmeleri yapısal olarak
çözemiyor.** `HAPS_MOBILE`'ın (ve gerçek mesafeli herhangi bir `HAPS_STATIONARY`
testinin) çalışabilmesi için iki yoldan biri gerekiyor:
1. NTN protokolünü (SIB19/`ta-Common-r17`/`ntn_Config_r17`) devreye almak - açık-çevrim
   ön-telafi PRACH'ın gördüğü rezidüel gecikmeyi Ncs penceresi içine çeker. Bu, projenin
   "kademeli NTN entegrasyonu" planındaki bir sonraki adımla zaten örtüşüyor.
2. Uzun PRACH formatına (839-uzunluklu dizi, çok daha büyük Ncs tablosu) doğru şekilde
   geçmek - denendi ama bu oturumda tamamlanamadı (yukarıdaki not).

**Güncel durum:** `HAPS_MOBILE`/gerçek-mesafeli `HAPS_STATIONARY` kod tarafı tamamen
hazır ve doğru çalışıyor (fizik doğru hesaplanıyor); bağlanamama nedeni artık net bir
şekilde teşhis edildi (PRACH Ncs penceresi), ama düzeltme bu oturumun kapsamı dışına
taşan iki ayrı büyük iş kalemi (NTN entegrasyonu veya uzun PRACH formatı) gerektiriyor -
kullanıcı onayıyla ikisinden biri seçilip bir sonraki adımda uygulanabilir.

---

### Adım 9 — NTN/SIB19 protokolü devreye alındı: mevcut durumun teyidi ve derinlemesine kök neden avı

Kullanıcı isteği: teşhise devam edip NTN/SIB19 protokolünü fiilen devreye almak. Bu adımda
`haps_test/`'teki band78 test düzeneği değil, ana repodaki mevcut (önceki oturumdan kalma,
henüz commit'lenmemiş) **band 254 + gerçek `ntn_Config_r17`/SIB19** config'leri kullanıldı:
`ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn-haps.conf` +
`ci-scripts/conf_files/nrue.uicc.ntn-haps.conf` (20 km sabit HAPS, `ta-Common-r17=32771`
[0.13 ms], `cellSpecificKoffset_r17=1`, `zeroCorrelationZoneConfig=15` — hepsi önceki
oturumun `oai-leo-to-haps-adaptation` hafıza notunda "Round 1-8" olarak belgelenen uzun
araştırmadan kalma değerler).

**Referans/teyit testi (değişiklik yapmadan, mevcut build ile):**
```
RA failed: 32   received correctly: 0   RAPID match: 3   RAPID mismatch: 29   no signal: 0
```
Bu, önceki oturumun bıraktığı durumu **birebir yeniden üretiyor** — bu oturumdaki diğer
değişikliklerin (yeni `HAPS_STATIONARY`/`HAPS_MOBILE` kanal tipleri, `sim.h`/
`random_channel.c`/`apply_channelmod.c` düzenlemeleri) bu NTN/band254 senaryosunu
etkilemediği doğrulandı (beklenen - hepsi farklı `modelid` dallarında).

**Round 7'nin yeniden doğrulanması:** `HAPS_DEBUG_NO_NTN_TA=1` (mevcut, önceki oturumdan
kalma debug bayrağı, `UE->timing_advance_ntn`'i hesaplandıktan hemen sonra sıfırlıyor)
ile tekrar test edildi:
```
RA failed: 30   received correctly: 1   RAPID match: 31   RAPID mismatch: 0
```
**Önceki oturumun bulgusu birebir doğrulandı**: NTN açık-çevrim TA ön-telafisi
KAPATILDIĞINDA RAPID eşleşmesi 3/32'den 31/31'e (yaklaşık %9'dan %100'e) çıkıyor. Bu,
aktif ön-telafinin düzeltme yapmak yerine **durumu kötüleştirdiğini** kanıtlıyor - rastgele
gürültü değil, sistematik bir işaret/büyüklük hatası olduğuna işaret ediyor (mismatch
deseni burada da tutarlı şekilde ±1: `preamble (0) vs (1)`, `(57) vs (56)`, vb. — önceki
oturumun bulgusuyla aynı).

**Yeni enstrümantasyon — Round 8'in daha önce hiç yapılmamış önerisi uygulandı:**
`apply_ntn_config()`'e (`nr_ntn_l1.c:134-183`) `HAPS_DEBUG_APPLY_NTN_CONFIG` env
değişkeniyle korunan bir log eklendi; `koffset`, `duration_rx_to_tx` ve `timing_advance`
alanlarının geçiş anındaki (önce/sonra) değerlerini yazdırıyor. Çalıştırılınca:
```
HAPS_DEBUG_APPLY_NTN_CONFIG: koffset=1(was 0) duration_rx_to_tx=4(was 3) timing_advance=7680(was 0) ...
HAPS_DEBUG_TA: ... timing_advance_ntn=2054 samples
```
`executables/nr-ue.c:897-911` kodu okunarak izlendi: `apply_ntn_config()` döndükten hemen
sonra `UE->timing_advance=0;` satırı çalışıyor (yeniden senkronizasyon nedeniyle kapalı
çevrim TA'yı sıfırlıyor - kasıtlı). Ardışık kararlı-durum TX döngüsünde
(`nr-ue.c:1012`), `new_timing_advance = UE->timing_advance + UE->timing_advance_ntn`
hesaplanıyor ve yerel `timing_advance` değişkeni bu yeni değere **geçiş anındaki 7680'lik
bookkeeping düzeltmesini hiç görmeden** üzerine yazılıyor - yani `apply_ntn_config()`'in
`koffset` geçişi için hesapladığı 7680 örneklik düzeltme, bir sonraki döngü adımında
sessizce **kayboluyor**.

**Hipotez ve deneysel test:** Bu bookkeeping düzeltmesinin (7680 örnek, `koffset<<mu`
kadar slot'a denk gelen süre) kaybolmasının kalıcı bir zamanlama kaymasına yol açtığı
düşünüldü. Test etmek için `nr_ntn_l1.c`'ye `HAPS_FIX_KOFFSET_PERSIST` env değişkenli bir
deneysel düzeltme eklendi: bu düzeltmeyi (yerel `*timing_advance` yerine/yanında) kalıcı
`UE->timing_advance_ntn` alanına da yazıyor, böylece kararlı-durum döngüsündeki
hesaplamadan kaybolmuyor.

**[Dosya]** `openair1/PHY/NR_UE_TRANSPORT/nr_ntn_l1.c`
```
~ Değiştirildi (apply_ntn_config): koffset_bookkeeping_delta ayrı bir değişkende
  tutulacak şekilde küçük bir refactor + HAPS_DEBUG_APPLY_NTN_CONFIG log satırı
+ Eklendi: apply_ntn_timing_advance_and_doppler() çağrısından sonra,
  `if (getenv("HAPS_FIX_KOFFSET_PERSIST")) UE->timing_advance_ntn += koffset_bookkeeping_delta;`
```

**Test sonucu: İYİLEŞME YOK.** `HAPS_FIX_KOFFSET_PERSIST=1` ile: `RA failed: 32,
received correctly: 0, RAPID match: 2, RAPID mismatch: 30` — referans testle (3/32)
istatistiksel olarak aynı (fark gürültü seviyesinde). **Bu deneysel düzeltme hipotezi
çürüttü** - `duration_rx_to_tx` geçişinin bookkeeping düzeltmesinin kaybolması, RAPID
mismatch'in gerçek nedeni **değil** (ya teorik analiz yanlıştı - bu kaybolma aslında
kasıtlı/zararsız bir tek-seferlik yumuşatma olabilir - ya da düzeltme yanlış yerde/yanlış
işaretle uygulandı). Kod, no-op varsayılan davranışla (env değişkeni set edilmedikçe hiç
çalışmıyor) ağaçta bırakıldı, ileride tekrar denenebilir.

**Genel durum:** Kök neden hâlâ tam olarak tek bir satıra/mekanizmaya sabitlenemedi.
Kesin olarak bilinenler:
1. Aktif NTN açık-çevrim TA ön-telafisi, PRACH RAPID tespitini **kötüleştiriyor** (sıfırlamak
   düzeltiyor) - işaret/büyüklük hatası kesin, ama tam olarak hangi terimde olduğu
   bulunamadı (`duration_rx_to_tx` geçiş bookkeeping'i test edildi, elendi).
2. Bu oturumun Adım 8 bulgusu (PRACH'ın Ncs-sınırlı arama penceresi) hâlâ geçerli ve
   muhtemelen etkiyi büyütüyor: küçük bir rezidüel hata bile (aktif ön-telafi varken) bu
   dar pencereyi aşmaya yetiyor, bu yüzden sonuç ya tam eşleşme ya da yakın-komşu
   sarmalanmış eşleşme (±1) oluyor, ara durum (kısmi bozulma) yok.
3. `HAPS_DEBUG_TA`, `HAPS_DEBUG_NO_NTN_TA`, `HAPS_DEBUG_MSG3` (önceki oturumdan) ve
   `HAPS_DEBUG_APPLY_NTN_CONFIG`, `HAPS_FIX_KOFFSET_PERSIST` (bu oturumda eklendi) - hepsi
   varsayılan olarak no-op, ağaçta tekrar kullanılmak üzere duruyor.

**Sonraki adım için değerlendirme:** Bu noktaya kadarki teşhis (bu oturum + önceki
oturumun 8 turu) kapsamlı ama kesin kök nedeni hâlâ bulamadı - `apply_ntn_timing_advance_and_doppler()`'daki
Doppler/TA hesap zincirinin geri kalanının (özellikle `N_common_ta_adj`'nin `ta-Common-r17`
config değerinden nasıl türetildiği, ya da `duration_rx_to_tx`'in kendisinin `writeTimestamp`
formülünde `get_samples_slot_duration()` ile nasıl birleştiği) satır satır elle
doğrulanması gerekiyor - bu, önceki oturumun LLR/kanal-kestirimi seviyesi PHY debug'ına
benzer büyüklükte, ayrı bir derinlemesine oturumu hak eden bir iş.

---

### Adım 10 — BÜYÜK BULUŞ: `ta-Common-r17` yanlış yapılandırılmış, UE'nin kendi hesabını ikiye katlıyormuş — kök neden bulundu ve düzeltildi

Kullanıcı isteği: tüm hesap zincirini (özellikle `N_common_ta_adj`'nin kökeni ve
`duration_rx_to_tx`/`writeTimestamp` etkileşimi) satır satır doğrulamaya devam etmek.

**`N_common_ta_adj`'nin kökeni izlendi** (`openair2/LAYER2/NR_MAC_UE/config_ue.c:371`):
```c
ntn_ta->N_common_ta_adj = ntn_Config_r17->ta_Info_r17->ta_Common_r17 * 4.072e-6; // ms
```
Yani `N_common_ta_adj`, config'teki `ta-Common-r17` alanından **doğrudan, birebir**
türüyor. `nr_ntn_l1.c:106`'da bu değer, UE'nin kendi ephemeris'ten hesapladığı
`N_UE_TA_adj` (`= 2000 * distance / SPEED_OF_LIGHT`, tam round-trip) ile **toplanıyor**:
`timing_advance_ntn = (N_UE_TA_adj + N_common_ta_adj + ...) * samples_per_subframe`.

**3GPP semantiği kontrol edildi** (`radio/rfsimulator/apply_channelmod.c`'deki LEO/transparent
senaryonun SIB19 güncelleme kodu referans alınarak, satır ~187): oradaki `ta-Common`,
`dist_sat_gnb`'den (yani **sadece uydu-gNB besleme/feeder hattı mesafesinden**) hesaplanıyor
— UE-uydu bacağından değil. Yani `ta-Common`'ın anlamı: "tüm UE'lerin ORTAK olarak
paylaştığı, UE'nin kendi hesaplayamayacağı ekstra bileşen" (tipik olarak şeffaf/bent-pipe
mimaride ayrı bir besleme hattı gecikmesi) — UE'nin zaten kendi ephemeris'inden hesapladığı
UE-uydu bacağının **bir kopyası değil**.

**Kritik tespit:** Bizim HAPS senaryomuz **tek-sekmeli** (gNB doğrudan platformun üzerinde,
ayrı bir besleme hattı yok — Adım 1'de `HAPS_STATIONARY`/`HAPS_MOBILE`'ı kasıtlı olarak
`SAT_LEO_REGEN`'e benzer tek-sekmeli tasarladığımız gibi). Bu mimaride ayrı bir "ortak/feeder"
bileşeni **yok** — tüm gecikme zaten `N_UE_TA_adj` içinde. Ama `gnb.sa.band254.u0.25prb.rfsim.ntn-haps.conf`
(önceki oturumdan kalma) `ta-Common-r17 = 32771` (~0.1334 ms) olarak ayarlanmıştı, ve
dosyadaki kendi yorumu da bunu doğruluyordu: `"one-way propagation UE-HAPS-gNB at 20 km
overhead: 2*20000/c = 133.43 us"` — yani **bilerek UE-HAPS round-trip'in bir kopyasını**
`ta-Common`'a yazmışlar; bu da `N_UE_TA_adj` (aynı ~0.134 ms) ile toplanınca, UE'nin
uyguladığı toplam ön-telafiyi **fiziksel olarak gerekenin ~2 katına** çıkarıyordu
(0.134+0.133≈0.267 ms yerine sadece 0.134 ms olmalıydı).

**Bu, Adım 9'un bulgusuyla (aktif ön-telafi düzeltme yerine bozuyor) birebir tutarlı**: fazladan
uygulanan ~0.133 ms'lik (~1027 örnek) hatalı ek telafi, PRACH'ın (Adım 8'de bulunan) dar
Ncs arama penceresini aşıp komşu bir cyclic-shift'e sarmalanmasına yol açıyordu.

**Deneysel doğrulama — `ta-Common-r17 = 0` (`/tmp/haps_ntn_diag/gnb.diag.conf`, geçici kopya):**
```
RA failed: 20   received correctly: 1   RAPID match: 21   RAPID mismatch: 0   no signal: 0
RRC_CONNECTED: ✅ (hem gNB hem UE logunda)
```
**Tekrarlanabilirlik için ikinci kez çalıştırıldı — aynı sonuç** (21/21 RAPID match,
RRC_CONNECTED). Bu, projenin **tüm HAPS/NTN çalışmasında ilk kez** gerçek 20 km gecikmeli,
tam SIB19/`ntn_Config_r17` senaryosunda `RRC_CONNECTED`'a ulaştığı an.

**[Dosya]** `ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn-haps.conf` — **kalıcı
olarak düzeltildi** (geçici kopya değil, gerçek dosya):
```
~ Değiştirildi:
- ta-Common-r17                                                 = 32771; # 0.13 ms
+ ta-Common-r17                                                 = 0;
```
**# Gerekçe:** Yukarıda açıklanan çift-sayım hatasını düzeltmek — tek-sekmeli (gNB platform
üzerinde) bir HAPS mimarisinde `ta-Common`'ın ayrı bir besleme-hattı bileşeni yok, bu yüzden
0 olmalı; UE zaten kendi `N_UE_TA_adj`'i ile tam round-trip'i hesaplıyor. Gerçek dosya
üzerinde üçüncü kez (geçici kopyadan sonra) tekrar test edildi, **aynı sonuç doğrulandı**
(21/21 RAPID match, RRC_CONNECTED).

**Kalan (ayrı, önceden bilinen) sorun:** RAPID artık %100 eşleşiyor, ama RA hâlâ çoğunlukla
(20/21) Msg3'te başarısız oluyor. `HAPS_DEBUG_MSG3=1` ile bakıldığında: **hepsi gerçek
sinyal** (`no signal: 0`, sentinel değerler yok), `ul_cqi` gerçekçi (146-152), `rssi=860`
sabit, `timing_advance` yakınsıyor (16-31 aralığında) — yani PHY gerçekten Msg3'ü
algılıyor, zamanlıyor, ama **CRC'de başarısız oluyor**. Bu, önceki oturumun Round 4-6'da
bulduğu ve "band 254/25RB/numerology-0'a özgü, NTN'den bağımsız, milder bir sağlamlık
sorunu" olarak nitelendirdiği **ikinci, ayrı bug** ile birebir aynı desen (Round 6'da NTN
config tamamen kaldırıldığında da "31 denemeden 30'u başarısız, 1'i başarılı" aynı oranı
bulmuşlardı). **Bu ikinci bug bu oturumda araştırılmadı** — NTN/SIB19 zincirinin
doğrulanması isteniyordu, o tamamlandı ve büyük bir kök neden bulundu; bu ikinci sorun PHY
LLR/kanal-kestirimi seviyesinde ayrı, derin bir inceleme gerektiriyor (Round 5'in kaldığı
yer).

**Özet — bu adımın kazanımı:** Aylarca (birden fazla oturum, ~8+1 tur) çözülemeyen "aktif
NTN TA ön-telafisi RAPID eşleşmesini bozuyor" bulmacasının **kesin kök nedeni bulundu ve
düzeltildi**: config dosyasındaki `ta-Common-r17` değeri, UE'nin kendi hesapladığı round-trip
gecikmenin yanlışlıkla bir kopyasıydı. Tek satırlık bir config düzeltmesiyle RAPID eşleşmesi
%9'dan %100'e çıktı ve proje tarihinde ilk kez gerçek-mesafeli NTN/HAPS senaryosunda
`RRC_CONNECTED`'a ulaşıldı.

---

### Adım 11 — İkinci bug'a (Msg3 PUSCH decode hatası) PHY-seviyesi inceleme

Kullanıcı isteği: Adım 10'da bulunan/düzeltilen birinci bug'dan (RAPID mismatch) sonra
kalan ikinci, ayrı sorunu (RAPID artık doğru eşleşiyor ama Msg3 çoğu zaman CRC'de
başarısız oluyor) detaylı bir PHY incelemesiyle çözmeye çalışmak.

**Ayrıntılı log toplama:** `phy_log_level="debug"` (geçici, `/tmp/haps_phy_diag/gnb.debug.conf`)
+ `HAPS_DEBUG_MSG3=1` ile 1M+ satırlık bir log toplandı, gerçek bir Msg3 denemesi (rnti
`2d98`, 4 HARQ turu) etrafında incelendi.

**Yeni bulgu — `est_delay` çoğu zaman ±20 örneklik telafi sınırını aşıyor:**
`openair1/SCHED_NR/phy_procedures_nr_gNB.c:431-462`'deki `nr_fill_indication()`'ın log'u
incelendi (`pusch->delay.est_delay`, `nr_phy_common.c:344`'teki `nr_est_delay()`'den —
kanal darbe-yanıtının IFFT'sinde pik konumunu bularak hesaplanan ince zamanlama kalıntısı).
Aynı RA denemesinin 4 HARQ turu için:

| Tur | `timing_advance` (MAC, TA komutu) | `est_delay` (örnek) | SNR | Sonuç |
|---|---|---|---|---|
| 0 (835.2) | 28 | -13 | 9.4 dB | NACK |
| 1 (836.6) | 15 | **-63** | 10.4 dB | NACK |
| 2 (838.3) | 31 (nötr) | **0** | 11.8 dB | NACK |
| 3 (840.0) | 31 (nötr) | 0 | 11.4 dB | NACK |

`openair1/PHY/defs_nr_common.h:51`'de `#define MAX_DELAY_COMP 20` — kanal kestirimindeki
DMRS-interpolasyon faz-düzeltmesi (`nr_ul_channel_estimation.c:177`,
`common/utils/nr/nr_common.c:1208 get_delay_idx()`) `est_delay`'i **±20 örnekle
kırpıyor**. Tur 1'deki `-63` örnek, bu sınırın **3 katından fazla** — kırpma nedeniyle
kanal kestirimi düzgün telafi edilemiyor, bu turun kötü decode olması bekleniyordu.

**Ama tur 2 ve 3'te `est_delay=0`** (tam telafi sınırları içinde, hatta sıfır — mükemmel
hizalanmış) **ve SNR 11-12 dB** (QPSK + coderate 0.153 için LDPC'nin normalde kolayca
yakınsaması gereken bir seviye) **olmasına rağmen decode YİNE başarısız oldu.** Bu, önceki
oturumun Round 5'te bulduğu ve çözemediği bilmeceyle **birebir aynı**: iyi koşullarda bile
CRC geçmiyor.

**Deneysel test — `MAX_DELAY_COMP`'u 20'den 80'e çıkarmak** (en azından büyük-`est_delay`'li
turları düzeltir mi diye):

**[Dosya]** `openair1/PHY/defs_nr_common.h`
```
~ Değiştirildi (geçici deney):
- #define MAX_DELAY_COMP 20
+ #define MAX_DELAY_COMP 80
```
Derlendi, test edildi. **Sonuç: DAHA KÖTÜ** — UE bir kez senkronize oldu ama hemen
ardından bağlantı tamamen koptu (`Lost socket`, sonrasında sürekli `synch Failed`), 0 RA
denemesi bile gerçekleşmedi. Muhtemel neden: `delay_table[2*MAX_DELAY_COMP+1][NR_MAX_OFDM_SYMBOL_SIZE]`
boyutundaki önceden-hesaplanmış tablo 4 kat büyüdü, bu da gerçek-zamanlı işlem
bütçesini (CPU/bellek) bozup RFSIM soket senkronizasyonunu kaybettirmiş olabilir. **Değişiklik
geri alındı** (`MAX_DELAY_COMP` tekrar 20'ye döndürüldü, yeniden derlendi, referans testle
(21/21 RAPID match civarı davranış) tekrar doğrulandı).

**Genel değerlendirme:** Bu turda gerçek, somut bir ek bulgu elde edildi (±20 örneklik
telafi sınırı, bazı HARQ turlarında gerçekten aşılıyor) ama bunu büyütmeye çalışan basit
bir müdahale işe yaramadı (aksine yeni bir bağlantı kaybı sorunu getirdi, geri alındı).
Daha da önemlisi, **kırpma sınırı içinde kalan, "temiz" turlar bile hâlâ başarısız oluyor**
— yani `est_delay`/`MAX_DELAY_COMP` kırpması ikinci bug'ın **tek** açıklaması değil, olsa
olsa katkıda bulunan bir faktör. Kök neden, önceki oturumun Round 5'te vardığı sonuçla aynı
yerde kalıyor: **iyi SNR + doğru LDPC parametreleri + (bazı turlarda) sıfır zamanlama
kalıntısına rağmen CRC başarısız** — bu, ham IQ/LLR/kanal-kestirimi seviyesinde (log
satırlarının ötesinde) dump alıp elle inceleme gerektiren, kapsamı bu oturumun log-tabanlı
yöntemini aşan bir sonraki adım.

**Ağaçta kalan durum:** `defs_nr_common.h`'deki `MAX_DELAY_COMP` orijinal değerine (20)
geri döndü — repo'da bu adımdan kalan kalıcı bir kod değişikliği yok, sadece bu günlükteki
bulgular kalıcı.

---

### Adım 12 — İKİNCİ BÜYÜK BULUŞ: `prop_delay` de aynı round-trip/one-way karışıklığından muzdaripmiş — %100 çözüldü

Kullanıcı sorusu: LEO uydusu da (HAPS'tan daha hızlı hareket etmesine rağmen) bu hatayı
almıyor gibi görünüyor, oradaki yaklaşım referans alınabilir mi?

**Netleştirme:** Önceki oturumun `oai-leo-to-haps-adaptation` notuna göre LEO senaryosunun
kendisi bu makinede **hiç doğrulanmış değil** (Round 3: LEO UE hiç senkronize olamadan
segfault etti — muhtemelen LEO'nun CI docker-compose'unun geçtiği ekstra
`--time-sync-I`/`--ntn-initial-time-drift`/`--initial-fo`/`--cont-fo-comp` bayrakları
verilmediği için). Yani "LEO'da bu hata yok" kesin olarak bilinmiyor. Ama kullanıcının
sorusundaki asıl değerli fikir doğruydu: **LEO'nun mimarisiyle bizim NTN-HAPS config'imiz
arasındaki yapısal farkı** kontrol etmeye değerdi.

**Bulunan fark ve kontrol:** LEO, gerçek geometriden **sürekli, dinamik** olarak hesaplanan
bir `channelmod` (`SAT_LEO_TRANS`, `enable_dynamic_delay=true`) kullanıyor — yayılım
gecikmesi SIB19'u besleyen AYNI ephemeris'ten türüyor, tutarlı. Bizim NTN-HAPS config'imiz
(Faz 1, önceki oturumdan) ise "GEO-style" bir kısayol kullanıyor: `options=(); # no
chanmod` + sabit bir `rfsimulator.prop_delay` — bu değer SIB19/ephemeris'ten **tamamen
kopuk**, elle girilmiş bir sayı.

**Bu kopukluğu incelerken `prop_delay` değerinin kendisinde Adım 10'dakiyle birebir aynı
türde bir hata bulundu**: `gnb.sa.band254...ntn-haps.conf:283` ve
`nrue.uicc.ntn-haps.conf:26`'da `prop_delay = 0.13;` ayarlıydı — dosyanın kendi yorumu
"one-way propagation UE-HAPS-gNB at 20 km overhead: 2*20000/c = 133.43 us" diyordu, yani
**`2*mesafe/c` (gidiş-dönüş) formülünü hesaplayıp "tek yön" diye etiketlemişler**.
`radio/rfsimulator/simulator.cpp:1604-1608`'e göre `prop_delay`, **her uç kendi alım
tarafına bağımsız olarak uyguluyor** (Adım 5'te band78 testinde doğru şekilde
`20000/c=0.06671 ms` - gerçek tek-yön değeri - kullanılarak doğrulanmıştı). Round-trip
değerini (0.13ms) her iki uca da tek-yön gibi uygulamak, simülatörün gerçekte simüle
ettiği toplam gecikmeyi ~2 katına çıkarıyordu — UE'nin zaten round-trip'i doğru hesaplayıp
`N_UE_TA_adj` ile ön-telafi ettiği gerçek fiziksel gecikmeyle **tutarsız** hale geliyordu.

**Deneysel doğrulama** (`/tmp/haps_ntn_diag2/`, `prop_delay = 0.06671` — gerçek tek-yön
değeri, Adım 10'un `ta-Common-r17=0` düzeltmesiyle birlikte):
```
RA failed: 0   received correctly: 1   RRC_CONNECTED: ✅ (ilk RA denemesinde!)
```
**3 kez art arda çalıştırıldı, üçünde de aynı sonuç**: 0 RA failed, ilk denemede
`RA-Msg3 transmitted` → `4-Step RA procedure succeeded` → `RRC_CONNECTED`, ardından
kararlı DL/UL HARQ trafiği (band78 testlerindeki sağlıklı desenle aynı). Önceki ~1/21
başarı oranından **tam %100, ilk-denemede başarıya** sıçrama.

**[Dosya]** `ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn-haps.conf` — **kalıcı
olarak düzeltildi**:
```
~ Değiştirildi:
- prop_delay = 0.13;
+ prop_delay = 0.06671;
```
**[Dosya]** `ci-scripts/conf_files/nrue.uicc.ntn-haps.conf` — **kalıcı olarak düzeltildi**:
```
~ Değiştirildi:
- prop_delay = 0.13;
+ prop_delay = 0.06671;
```
**# Gerekçe:** `prop_delay` her uçta bağımsız uygulandığı için tek-yön mesafe/c değeri
olmalı, round-trip değeri değil - Adım 5'in band78 testinde zaten doğru kullanılmıştı, bu
NTN config'i o kuralı takip etmiyordu. Gerçek dosyalar üzerinde tekrar test edildi, **aynı
sonuç doğrulandı** (0 RA failed, ilk denemede RRC_CONNECTED).

**Sonuç:** Kullanıcının "LEO'daki çözümü referans alalım" önerisi doğru bir sezgiydi —
LEO'nun kendisi bu makinede doğrulanmamış olsa da, "gecikme temsilinin tüm bileşenler
arasında (simülatör, SIB19 broadcast, UE'nin kendi hesabı) tutarlı ve round-trip/one-way
karışıklığından arınmış olması gerektiği" ilkesi doğru çıktı — Adım 10 (`ta-Common`) ve
Adım 12 (`prop_delay`) **aynı hata sınıfının iki ayrı örneğiydi**. İkisi birlikte
düzeltilince proje tarihinde **ilk kez**, gerçek 20 km gecikmeli tam NTN/SIB19
senaryosunda **tutarlı, tekrarlanabilir, ilk-denemede `RRC_CONNECTED`** elde edildi.
Önceki "ikinci bug" (Adım 11'in PHY-seviyesi CRC bilmecesi) artık **çözülmüş görünüyor** -
kök neden aslında PHY/LLR seviyesinde değil, bu zamanlama tutarsızlığındaymış.

---

### Adım 13 — `HAPS_MOBILE`'ı NTN/SIB19 ile birleştirme: üçüncü bir çerçeve-tutarlılığı hatası bulundu ve düzeltildi, tam başarı elde edildi

Kullanıcı isteği: `HAPS_MOBILE`'ı (Adım 1/7'nin gerçek loiter-dairesi/Doppler kanal
modeli) gerçek NTN/SIB19 protokolüyle birleştirmek.

**Tasarım:** Mevcut NTN-HAPS senaryosu (Adım 9-12) `options=(); prop_delay=sabit` (Faz 1,
SIB19'dan tamamen kopuk) kullanıyordu. Bunun yerine kendi `HAPS_MOBILE` kanal modelimizi
(`chanmod` aktif) kullanıp, AYNI geometrinin SIB19 yayınını da beslemesini sağladık —
LEO'nun zaten yaptığı gibi.

**[Dosya]** `radio/rfsimulator/apply_channelmod.c`
```
~ Değiştirildi: SIB19 güncelleme koşulu artık HAPS_STATIONARY/HAPS_MOBILE'ı da kapsıyor
  (`nr_update_sib19()` `ntn_Config_r17` yoksa zaten no-op döndüğü için band78/NTN-siz
  testler için güvenli, davranış değişikliği yok).
```
**# Gerekçe:** LEO'nun SIB19 yayınını sürekli güncel tutan aynı mekanizmayı HAPS için de
açmak - `HAPS_MOBILE`'da platform gerçekten hareket ettiği için bu şart.

**[Dosya]** `haps_test/gnb.haps_mobile_ntn.conf`, `haps_test/nrue.haps_mobile_ntn.conf` —
**yeni oluşturuldu**: band254 + `ntn_Config_r17`/SIB19 (Adım 10/12'nin düzeltmeleriyle) +
`channelmod` bloğu (`type="HAPS_MOBILE"`, `chanmod` aktif, statik `prop_delay` kaldırıldı).

**İlk test: BAŞARISIZ, yeni bir hata bulundu.** UE loglarında:
```
timing_advance_ntn: 325749 samples   (beklenenin ~300 katı!)
writeBlockSize is -310389, setting it to 0, changing timing_advance to 15360
writeBlockSize is -302709, ... to 23040
... (sürekli büyüyen, hiç yakınsamayan bir döngü)
```
Ayrıca gNB/UE süreçleri **tamamen kilitlendi** (timeout'a rağmen sonlanmadı, elle
`kill -9` gerekti).

**Kök neden — üçüncü bir çerçeve/koordinat tutarsızlığı:** `apply_channelmod.c`'deki HAPS
dalımız (Adım 7) **yerel düz-dünya** koordinatları kullanıyor (yer istasyonu orijinde
(0,0,0), irtifa Z ekseninde ~20000 m). Ama SIB19'un `positionX/Y/Z-r17` alanları ve UE'nin
`position0` config'i (`z=6377900`) **Dünya-merkezli** bir çerçeve varsayıyor (yer
istasyonu (0,0,dünya_yarıçapı) noktasında, LEO dalının da kullandığı çerçeve). SIB19'a
kendi yerel Z=20000'imizi olduğu gibi yazınca, UE bunu `position0=(0,0,6377900)` ile
birleştirip mesafe hesaplayınca **uydunun neredeyse Dünya'nın merkezinde olduğunu**
sanıyordu (~6358 km'lik tamamen yanlış bir "mesafe"), bu da 42 ms'lik saçma bir TA
değerine (325749 örnek) yol açıyordu.

**[Dosya]** `radio/rfsimulator/apply_channelmod.c`
```
~ Değiştirildi (SIB19 güncelleme bloğu):
+ const bool is_haps = channelDesc->modelid == HAPS_STATIONARY || channelDesc->modelid == HAPS_MOBILE;
+ const double sib19_pos_sat_z = is_haps ? pos_sat_z + radius_earth : pos_sat_z;
  ...
- .position.Z = pos_sat_z / 1.3,
+ .position.Z = sib19_pos_sat_z / 1.3,
```
**# Gerekçe:** SIB19 yayını için sadece Z eksenine `radius_earth` (6377900 m) eklemek,
yerel düz-dünya Z'yi (irtifa) Dünya-merkezli çerçeveye çeviriyor - X/Y zaten küçük yerel
ofsetler olduğu için (LEO dalının da X'i hep 0 tutup sadece Y kullanması gibi)
dokunulmadı. Kanal modelinin **kendi** gecikme/Doppler hesabı (bu SIB19 kodunun hemen
üstünde) hâlâ yerel koordinatları kullanıyor - sadece yayın kodlaması düzeltildi.

**Düzeltmeden sonra test: BAŞARILI.**
```
timing_advance_ntn: 1044 samples   (~0.136 ms - 20.4 km'lik gerçek mesafeyle tutarlı)
RA failed: 0   RAPID match: 1/1   RRC_CONNECTED: ✅ (ilk RA denemesinde)
```
**İki kez art arda çalıştırıldı, ikisinde de aynı sonuç.** Bağlantı kurulduktan sonra
kararlı kaldı (Doppler kayması `DL Doppler shift: 0.001898kHz` gibi küçük, gerçekçi
değerlerde, hareketli platformla tutarlı).

**Sonuç:** `HAPS_MOBILE` artık gerçek NTN/SIB19 protokolüyle **tam çalışır durumda** -
platform gerçekten hareket ediyor (2 km yarıçaplı loiter dairesi, ~100 km/h), Doppler
gerçek zamanlı hesaplanıyor, SIB19 yayını bu hareketle senkronize kalıyor, ve UE ilk RA
denemesinde bağlanabiliyor. Bu, projenin en baştaki hedefiydi (gerçekçi, hareketli bir
HAPS simülasyonu) ve şimdi elde edildi.

---

## 4. Bilinen sınırlamalar / açık konular

- **`HAPS_MOBILE` (band78/karasal, NTN'siz test) RA/RRC bağlantısı hâlâ kurulamıyor** —
  Adım 8'de kök neden teşhis edildi: 20 km'lik gerçek gecikme, PRACH korelatörünün
  `zeroCorrelationZoneConfig`'ten türeyen dar arama penceresini aşıyor. Bu **sadece
  band78/NTN'siz test için** hâlâ geçerli bir sınırlama (`haps_test/gnb.haps_mobile.conf`).
  Asıl hedef olan NTN'li senaryo Adım 13'te **çözüldü** (aşağıya bakın).
- **NTN protokolü + `HAPS_MOBILE` (band 254, gerçek hareket/Doppler) artık tam çalışıyor** —
  Adım 9-13'te devreye alındı; üç ayrı kök neden bulunup düzeltildi: `ta-Common-r17` ve
  `prop_delay`'in ikisi de round-trip/one-way karışıklığından muzdaripti (Adım 10/12), ve
  SIB19 yayınının kendi Dünya-merkezli çerçevesiyle HAPS'ın yerel düz-dünya çerçevesi
  arasında bir koordinat tutarsızlığı vardı (Adım 13). Düzeltildikten sonra hem sabit
  (`ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn-haps.conf` +
  `nrue.uicc.ntn-haps.conf`) hem hareketli (`haps_test/gnb.haps_mobile_ntn.conf` +
  `nrue.haps_mobile_ntn.conf`) senaryolarda ilk-denemede `RRC_CONNECTED` elde ediliyor.
- **`docker-compose.yaml` düzeltilmedi** — Adım 2'de tespit edilen `--conf_file` ve
  `chan_mod`/`chanmod` hataları hâlâ duruyor, Docker akışına geçilirse ele alınmalı.
- **Yol kaybı hâlâ keyfi bir değerde (`ploss_dB=20`)** — Adım 6'da gerçek 20 km FSPL
  (129.64 dB) hesaplanıp denendi, ama saf kayıp olarak uygulanınca bağlantı tamamen
  koptu (rfsim bunu telafi edecek bir tx/rx anten kazancı modellemiyor), bu yüzden geri
  alındı. Gerçekçi bir değer için FSPL'in üzerine gerçekçi bir tx+rx kazanç bütçesi
  eklenmesi gerekiyor — henüz yapılmadı, kullanıcı kararı bekleniyor.
- Gürültü tabanı (`noise_power_dB=-110`) da hâlâ keyfi seçilmiş durumda,
  güncellenmedi.
