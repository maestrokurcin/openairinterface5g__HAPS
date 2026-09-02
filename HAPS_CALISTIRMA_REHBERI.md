# HAPS Kanal Modeli — Çalıştırma Rehberi

Bu doküman, `haps_test/` klasöründeki her test senaryosunu **terminalden nasıl çalıştıracağını** anlatan pratik bir rehber. Mimarinin nasıl çalıştığı için `HAPS_MIMARI.md`'ye, "neden böyle yapıldı"nın tam geçmişi için `HAPS_GELISTIRME_GUNLUGU.md`'ye bakın.

## 0. Ön koşullar

**Her zaman ikisini birlikte derle** (`librfsimulator.so` çalışma zamanında `dlopen()` ile yükleniyor, ninja'nın bağımlılık grafiğinde `nr-softmodem`'e bağlı değil — sadece `nr-softmodem` derlersen sessizce eski kalır):

```
cd /home/furkan/openairinterface5g/cmake_targets
ninja -C ran_build/build rfsimulator nr-softmodem nr-uesoftmodem
```

**Her çalıştırmada `MALLOC_ARENA_MAX=1` şart** — onsuz bu makinede `nr-softmodem` çöküyor/takılıyor.

**Sıra**: Önce gNB'yi başlat, ~6 saniye bekle (RU/PHY ayağa kalksın), sonra UE'yi başlat. Ayrı iki terminal kullan.

**Nereden çalıştırılır**: Komutlar `cmake_targets/` dizininden, `../haps_test/...` göreli yoluyla yazılmıştır — hepsi bu varsayımla.

**Başarı belirtisi**: UE logunda `[PHY] UE synchronized!` ve `[NR_RRC] ... Generating RRCSetupComplete` görürsen bağlantı kurulmuş demektir.

---

## 1. Senaryo tablosu

| # | Senaryo | gNB config | UE config | UE'nin ek bayrakları gerekir mi? |
|---|---|---|---|---|
| 1 | `HAPS_STATIONARY`, band78, NTN'siz | `gnb.haps.conf` | `nrue.haps.conf` | **Evet** |
| 2 | `HAPS_MOBILE`, band78, NTN'siz | `gnb.haps_mobile.conf` | `nrue.haps_mobile.conf` | **Evet** |
| 3 | `HAPS_MOBILE` + gerçek NTN/Doppler (_38811 değil) | `gnb.haps_mobile_ntn.conf` | `nrue.haps_mobile_ntn.conf` | Hayır |
| 4 | `HAPS_MOBILE_38811`, kırsal/banliyö (varsayılan) | `gnb.haps_mobile_ntn_38811.conf` | `nrue.haps_mobile_ntn_38811.conf` | Hayır |
| 5 | `HAPS_MOBILE_38811_URBAN` | `gnb.haps_mobile_ntn_38811_urban.conf` | `nrue.haps_mobile_ntn_38811_urban.conf` | Hayır |
| 6 | `HAPS_MOBILE_38811_DENSE_URBAN` | `gnb.haps_mobile_ntn_38811_dense_urban.conf` | `nrue.haps_mobile_ntn_38811_dense_urban.conf` | Hayır |
| 7 | MIMO 2x2 | `gnb.haps_mobile_ntn_38811_2x2.conf` | `nrue.haps_mobile_ntn_38811_2x2.conf` | Hayır |
| 8 | SIMO/MISO 2x1 (⚠️ RRC'ye ulaşmaz, sadece kod testi) | `gnb.haps_mobile_ntn_38811_2x1.conf` | `nrue.haps_mobile_ntn_38811_2x1.conf` | Hayır |

**Neden bazıları ek bayrak istiyor?** Senaryo 1-2 (band78, NTN'siz) UE config'lerinde bir `cells = (...)` bloğu yok — frekans/PRB/numeroloji CLI'dan verilmek zorunda. Senaryo 3-8 (band254, NTN) UE config'lerinde bu bilgi zaten `cells` bloğunda var, CLI'ya hiçbir şey eklemeye gerek yok.

---

## 2. Komutlar

### Senaryo 1 — `HAPS_STATIONARY` (band78, NTN'siz)

En çok kanıtlanmış, en basit senaryo. Sabit platform, gerçek TR 38.811 fiziği (Adım 27).

```
# Terminal 1 (gNB)
cd /home/furkan/openairinterface5g/cmake_targets
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-softmodem -O ../haps_test/gnb.haps.conf --rfsim
```
```
# Terminal 2 (UE, gNB'den ~6sn sonra)
cd /home/furkan/openairinterface5g/cmake_targets
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-uesoftmodem -O ../haps_test/nrue.haps.conf -r 106 --numerology 1 -C 3619200000 --rfsim
```

### Senaryo 2 — `HAPS_MOBILE` (band78, NTN'siz)

Hareketli platform (2 km yörünge), gerçek TR 38.811 fiziği. **Not**: bu senaryo `HAPS_UE_SPEED_MPS` varsayılanıyla (3km/h) ara sıra gerçekçi bağlantı kesilmesi yaşayabilir (Adım 33) — bu bir bug değil.

```
# Terminal 1 (gNB)
cd /home/furkan/openairinterface5g/cmake_targets
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-softmodem -O ../haps_test/gnb.haps_mobile.conf --rfsim
```
```
# Terminal 2 (UE)
cd /home/furkan/openairinterface5g/cmake_targets
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-uesoftmodem -O ../haps_test/nrue.haps_mobile.conf -r 106 --numerology 1 -C 3619200000 --rfsim
```

### Senaryo 3 — `HAPS_MOBILE` + gerçek NTN/SIB19 (band254, `_38811` değil)

```
# Terminal 1 (gNB)
cd /home/furkan/openairinterface5g/cmake_targets
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-softmodem -O ../haps_test/gnb.haps_mobile_ntn.conf --rfsim
```
```
# Terminal 2 (UE)
cd /home/furkan/openairinterface5g/cmake_targets
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-uesoftmodem -O ../haps_test/nrue.haps_mobile_ntn.conf --rfsim
```

### Senaryo 4-6 — `HAPS_MOBILE_38811` ailesi (kırsal/banliyö, kentsel, yoğun kentsel)

Aynı komut deseni, sadece dosya adını değiştir. Kırsal/banliyö (varsayılan) örneği:

```
# Terminal 1 (gNB)
cd /home/furkan/openairinterface5g/cmake_targets
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-softmodem -O ../haps_test/gnb.haps_mobile_ntn_38811.conf --rfsim
```
```
# Terminal 2 (UE)
cd /home/furkan/openairinterface5g/cmake_targets
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-uesoftmodem -O ../haps_test/nrue.haps_mobile_ntn_38811.conf --rfsim
```

Kentsel için `gnb.haps_mobile_ntn_38811_urban.conf`/`nrue.haps_mobile_ntn_38811_urban.conf`, yoğun kentsel için `..._dense_urban.conf` kullan — komut deseni birebir aynı.

### Senaryo 7 — MIMO 2x2

```
# Terminal 1 (gNB)
cd /home/furkan/openairinterface5g/cmake_targets
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-softmodem -O ../haps_test/gnb.haps_mobile_ntn_38811_2x2.conf --rfsim
```
```
# Terminal 2 (UE)
cd /home/furkan/openairinterface5g/cmake_targets
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-uesoftmodem -O ../haps_test/nrue.haps_mobile_ntn_38811_2x2.conf --rfsim
```
gNB logunda `nb_tx_streams 2, nb_rx_streams 2` görmelisin.

### Senaryo 8 — SIMO/MISO 2x1 (⚠️ sadece kod testi)

```
# Terminal 1 (gNB)
cd /home/furkan/openairinterface5g/cmake_targets
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-softmodem -O ../haps_test/gnb.haps_mobile_ntn_38811_2x1.conf --rfsim
```
```
# Terminal 2 (UE)
cd /home/furkan/openairinterface5g/cmake_targets
MALLOC_ARENA_MAX=1 ./ran_build/build/nr-uesoftmodem -O ../haps_test/nrue.haps_mobile_ntn_38811_2x1.conf --rfsim
```
**Bu senaryo RRC'ye ulaşmaz** — UE `synch Failed` ile takılı kalır (Adım 35, HAPS'a özel olmayan bir OAI sınırı). `HAPS_DEBUG_TDL=1` ile `haps_tdl.c`'nin `n_pairs=2` kodunun doğru çalıştığını görebilirsin, ama gerçek bir bağlantı bekleme.

---

## 3. Faydalı ekstra bayraklar

Herhangi bir komutun başına eklenebilir (env var, `MALLOC_ARENA_MAX=1` ile aynı yere):

| Bayrak | Ne işe yarar |
|---|---|
| `HAPS_DEBUG_38811=1` | Yol kaybı hesap izini gNB/UE logunda gösterir |
| `HAPS_DEBUG_TDL=1` | Küçük-ölçekli sönümleme (TDL) tap/korelasyon izini gösterir |
| `HAPS_DEBUG_O2I=1` | O2I bina girişi kaybı hesap izini gösterir (`HAPS_O2I_ENABLE=1` ile birlikte anlamlı) |
| `HAPS_DEBUG_FRACDELAY=1` | Fraksiyonel gecikme izini gösterir |
| `HAPS_GROUND_OFFSET_M=35000` | Terminali kapsama merkezinden uzağa koyar (düşük yükseklik açısı testi) |
| `HAPS_RAIN_RATE_MM_H=25` | Yağmur kaybını devreye sokar (varsayılan: açık hava) |
| `HAPS_O2I_ENABLE=1` | O2I bina girişi kaybını devreye sokar |
| `HAPS_TDL_USE_ALT_PROFILE=1` | NTN-TDL-A/C yerine B/D kullanır |
| `HAPS_UE_SPEED_MPS=0` | Sönümlemeyi tamamen durdurur (varsayılan 3km/h yerine sabit) |
| `HAPS_PLATFORM_SPEED_MPS=0 HAPS_LOITER_RADIUS_M=0` | Platformu zenitte dondurur (loiter Doppler'i + gecikme kayması 0); NTN-TDL sönümleme aktif kalır (Adım 44) — sadece `HAPS_MOBILE*` |

Örnek (Senaryo 4'ü tüm debug izleriyle çalıştırma):
```
HAPS_DEBUG_38811=1 HAPS_DEBUG_TDL=1 MALLOC_ARENA_MAX=1 ./ran_build/build/nr-softmodem -O ../haps_test/gnb.haps_mobile_ntn_38811.conf --rfsim
```

---

## 4. Docker ile çalıştırma (hafif imaj)

Resmi `docker-compose.yaml` bu makinenin disk alanına sığmıyor (Adım 34) — onun yerine, zaten derlenmiş binary'leri kullanan hafif bir eşdeğer var (`Senaryo 1`'i, `HAPS_STATIONARY`'yi çalıştırır):

```
# 1. Çalışma-zamanı dosyalarını stage'e kopyala
mkdir -p /tmp/haps-oai-runtime
cp /home/furkan/openairinterface5g/cmake_targets/ran_build/build/{nr-softmodem,nr-uesoftmodem,*.so} /tmp/haps-oai-runtime/

# 2. İmajı derle (sudo gerekir)
sudo docker build -f /home/furkan/openairinterface5g/haps_test/Dockerfile.lite -t haps-oai:lite /tmp/haps-oai-runtime

# 3. Çalıştır
sudo docker compose -f /home/furkan/openairinterface5g/haps_test/docker-compose.lite.yml up -d

# 4. Logları kontrol et
sudo docker logs haps-ue-lite 2>&1 | grep -iE "synchronized|RRCSetupComplete"

# 5. Kapat
sudo docker compose -f /home/furkan/openairinterface5g/haps_test/docker-compose.lite.yml down
```

---

## 5. Bir şeyler ters giderse

`HAPS_MIMARI.md`'nin **"6. Sorun giderme"** bölümüne bak — belirti → hangi dosyaya bakılacağı tablosu var. En sık karşılaşılanlar:

- **`librfsimulator.so` eski davranıyor** → sadece `nr-softmodem` derlemişsindir, `ninja rfsimulator nr-softmodem nr-uesoftmodem` birlikte çalıştır.
- **gNB anında çöküyor/takılıyor** → `MALLOC_ARENA_MAX=1` unutulmuş.
- **UE "Undefined Frequency Range for frequency 0 Hz" ile çöküyor** → band78 senaryosunda (1-2) `-r 106 --numerology 1 -C 3619200000` bayrakları unutulmuş.
- **UE hiç `synchronized` demiyor** → yanlış config çifti eşleştirilmiş olabilir (örn. `gnb.haps_mobile_ntn_38811_2x2.conf` ile `nrue.haps.conf` gibi) - her senaryonun kendi gNB/UE çiftini kullan.
