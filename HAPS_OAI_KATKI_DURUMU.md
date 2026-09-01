# HAPS → OAI Katkı Durumu

Bu doküman, HAPS kanal modelini gerçek upstream OAI'ye ("Duranta OpenAirInterface", `github.com/duranta-project/openairinterface5g`) katkı olarak göndermek için **OAI'nin kendi standartlarının** ne istediğini, **bizim şu ana kadar neyi tamamladığımızı**, ve **neyin hâlâ eksik olduğunu** tek bir yerde takip etmek için var. Araştırma, bu repodaki `CONTRIBUTING.md` ve `doc/code-style-contrib.md` dosyalarının okunmasına dayanıyor.

## 1. OAI'nin katkı standartları

| # | Gereksinim | Açıklama |
|---|---|---|
| 1 | **Doğru hedef** | Kanonik repo artık `github.com/duranta-project/openairinterface5g` (eski `gitlab.eurecom.fr/oai` değil) - fork'lanıp PR `develop`'a açılmalı |
| 2 | **CLA** | Katkı Lisans Anlaşması imzalanmalı (CONTRIBUTING.md'de link var) |
| 3 | **Commit imzası** | Her commit'te `Signed-off-by:` (DCO, `git commit -s`) **ve** kriptografik imza (SSH/GPG) - CI imzasız commit'i reddediyor |
| 4 | **AI kullanım bildirimi** | Her commit'te `Assisted-by: AGENT_NAME:MODEL_VERSION` (örn. `Assisted-by: Claude:claude-sonnet-5`) - AI asla `Co-authored-by` ile eklenmiyor, sadece insan ekliyor |
| 5 | **Doğrusal geçmiş** | Merge commit yok; her commit küçük, kendi başına anlamlı bir "mantıksal değişiklik", ideal olarak tek başına derlenip bir rfsim E2E testi geçmeli. "Fix bug"/"review addressed" tarzı takip commit'leri yasak - geçmiş yeniden yazılmalı |
| 6 | **PR etiketi** | En az bir etiket seçilmeli (`documentation`/`BUILD-ONLY`/`4G-LTE`/`5G-NR`/`nrUE`) yoksa CI çalışmıyor - bizim için en az `5G-NR` |
| 7 | **Kod stili** | `.clang-format` uyumu (2 boşluk girinti, 132 karakter satır, vb.) |
| 8 | **Büyük katkı bildirimi** | Büyük katkılar önceden haftalık geliştirici toplantısında bahsedilmeli |
| 9 | **Emsal** | `SAT_LEO_TRANS`/`SAT_LEO_REGEN` (bu HAPS çalışmasının üzerine kurulduğu model) tek, odaklı bir commit'le eklenmiş (Ağustos 2024, Thomas Schlichter) - başlık kısa, gövde kaynak referansına atıf yapıyor - bizim commit'lerimiz için model bu |

## 2. Tamamladıklarımız

| Gereksinim | Durum | Not |
|---|---|---|
| Kod stili (#7) | ✅ **Tamamlandı** (Adım 37) | `git-clang-format` ile 12 dosya düzeltildi, kalan 4 yeni dosya zaten uyumluydu. Regresyonla doğrulandı. |
| Lisans başlığı | ✅ **Zaten doğru** | `LicenseRef-CSSL-1.0` - projenin güncel lisansıyla tam eşleşiyor, düzeltme gerekmedi |
| Fiziksel/teknik dokümantasyon | ✅ **Hazır** | `HAPS_GELISTIRME_GUNLUGU.md` (kronolojik gerekçe), `HAPS_MIMARI.md` (mimari + 3GPP/ITU-R referansları) - PR açıklaması hazırlarken doğrudan kullanılabilir |

## 3. Eksiklerimiz

| Gereksinim | Durum | Ne yapılmalı |
|---|---|---|
| Açık token (güvenlik) | 🔴 **Çözülmedi** | `github` remote'unun URL'sinde hâlâ gerçek bir GitHub token'ı açık duruyor - iptal edilip URL'den çıkarılmalı |
| Doğru hedef (#1) | ⚠️ **Netleşmedi** | `origin` hâlâ eski `gitlab.eurecom.fr/oai` - PR'ın gerçekten `duranta-project/openairinterface5g`'ye mi gideceği kullanıcıyla netleştirilmeli |
| CLA (#2) | ❌ **İmzalanmadı** | CONTRIBUTING.md'deki bağlantıdan imzalanmalı |
| Commit imzası (#3) | ❌ **Hiçbiri imzalı değil** | 19 commit'in hiçbirinde `Signed-off-by`/kriptografik imza yok |
| AI bildirimi (#4) | ❌ **Hiçbir commit'te yok** | `Assisted-by: Claude:<model>` trailer'ı hiçbir commit'te eklenmedi |
| Doğrusal, temiz geçmiş (#5) | ❌ **Mevcut değil** | 19 commit karışık (Türkçe/İngilizce başlıklı, bazıları kendi önceki commit'lerini yeniden düzenliyor - örn. Adım 26'nın mimari refactor'u) - PR öncesi rebase/squash ile mantıksal, İngilizce, birbirinden bağımsız derlenip test edilebilir commit'lere dönüştürülmeli |
| PR etiketi (#6) | ⏳ **Henüz uygulanamaz** | PR açılmadan seçilemez, ama en az `5G-NR` olması gerektiği biliniyor |
| Büyük katkı bildirimi (#8) | ❌ **Yapılmadı** | Haftalık geliştirici toplantısı henüz bahsedilmedi |
| `doc/git-guide.md` | ❌ **Detaylıca okunmadı** | Commit imzalama adımlarının tam prosedürü için okunmalı |
| PR açıklaması | ❌ **Hazırlanmadı** | Kalibrasyon sabitlerini (Adım 36, `HAPS_38811_SIM_CALIBRATION_DB` gibi 3GPP'den gelmeyen tek kalan sayılar) ve env-var tasarım kararlarını (yağmur/O2I/TDL alt-profil gibi) proaktif açıklayan bir açıklama yazılmalı |
| `load_channellist()` hatası | ❌ **Ayrı bir konu olarak açılmadı** | HAPS'a özel olmayan, projedeki tüm kanal modellerini etkileyen bir OAI hatası (Adım 35) - ayrı bir issue/PR olarak düşünülebilir, bizim katkımıza bağımlı değil |

## 4. Özet - sıradaki en yüksek öncelik

1. **Açık token'ı iptal et** (hızlı, acil, katkı kararından bağımsız)
2. **Hedefi netleştir** (her şeyi etkiliyor)
3. `doc/git-guide.md`'yi oku
4. Commit geçmişini yeniden düzenle (en büyük iş)
5. PR açıklamasını hazırla
6. `5G-NR` etiketiyle PR'ı aç, büyüklüğü nedeniyle haftalık toplantıda bahset
