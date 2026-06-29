# 🎬 Tdarr Akışları

<p>
  <img src="https://img.shields.io/badge/Tdarr-Ak%C4%B1%C5%9F%20Seti-orange?style=flat-square" alt="Tdarr">
  <img src="https://img.shields.io/badge/Kodlay%C4%B1c%C4%B1-HEVC%20NVENC-blue?style=flat-square" alt="HEVC NVENC">
  <img src="https://img.shields.io/badge/Ses-FLAC%2FWAV-9cf?style=flat-square" alt="FLAC/WAV">
  <img src="https://img.shields.io/badge/Lisans-Ki%C5%9Fisel%20Kullan%C4%B1m-lightgrey?style=flat-square" alt="Lisans">
</p>

Kendi medya sunucumda kullandığım, üç parçadan oluşan bir **Tdarr** akış seti.
Amaç basit: kütüphanedeki görüntü ve ses dosyalarını gereksiz yer kaplamadan, kalite kaybını en aza indirerek dönüştürmek; zaten yeterince iyi olan dosyalara hiç dokunmamak.

Bu akışlar kendi kütüphanem için ayarlanmış; dil tercihleri ve Sonarr, Radarr, Bazarr, Jellyfin bağlantıları kendi kurulumuma göre. Mantığı genel olduğu için herkes ihtiyacına göre çatallayıp kendine uyarlayabilir.

---

## 📦 İçindekiler

| Dosya | Görev |
|---|---|
| 🎞️ [`hevc_shrink.json`](#️-hevc_shrinkjson--görüntü-dönüştürme-akışı) | Ana görüntü dönüştürme akışı |
| 🛠️ [`prepare_transcode.json`](#️-prepare_transcodejson--dosya-uyumlandırma) | `hevc_shrink`'in çağırdığı, dosyayı uyumlandıran yardımcı akış |
| 🎵 [`flac_downsample.json`](#-flac_downsamplejson--yüksek-çözünürlüklü-ses-düşürme) | FLAC/WAV yüksek çözünürlüklü ses dosyalarını düşüren akış |

```
                ┌─────────────────────┐
   görüntü ───► │   hevc_shrink.json   │ ───► Sonarr/Radarr ✓
                └──────────┬──────────┘       Bazarr ✓
                           │ hata veya MKV değilse   Jellyfin ✓
                           ▼
                ┌─────────────────────┐
                │ prepare_transcode    │ ── geri döner ──► hevc_shrink
                └─────────────────────┘

   flac/wav ───► ┌─────────────────────┐
                 │ flac_downsample.json │
                 └─────────────────────┘
```

---

## 🎞️ hevc_shrink.json — Görüntü Dönüştürme Akışı

Kütüphanedeki tüm görüntü dosyalarına bakan ana akış.

### ⏭️ Atlama koşulları

İşlem yapmadan geçilen durumlar:
- Dosya zaten işlenmiş, yol bazlı tekrar denetimi bunu yakalıyor
- Genel bit hızı 1.5 Mbps'in altında, zaten yeterince küçük
- Dosya zaten HEVC, AV1 veya VP9 ve çözünürlüğe göre eşiğin altında:

  | Çözünürlük | Eşik |
  |---|---|
  | 1440p ve üstü | 8 Mbps altı |
  | 1080p | 5 Mbps altı |
  | 720p altı | 4 Mbps altı |

### ✈️ Ön hazırlık
- Önemsiz, font veya yan dosya niteliğindeki görsel dosyaları siler
- Altyazı katmanlarını süzer, UTF-8 olanları tutar, görünüm ayarlarını düzeltir
- Girdi dosyasında sağlık denetimi çalıştırır
- Dosya MKV değilse önce `prepare_transcode` akışını çağırır, sonra geri döner

### ⚙️ Kodlama — hevc_nvenc, değişken bit hızı, iki geçişli

| Durum | Ayar |
|---|---|
| 1440p ve üstü | 10 bit HEVC, HDR10/HLG bilgisi korunur |
| 1080p, yüksek bit hızı, 5 Mbps üstü | HEVC CQ 22, p7 ön ayarı |
| 1080p, düşük bit hızı | yeniden kodlama yapılmaz |
| 720p altı ve eski kodek | HEVC CQ 23 |

### ✅ Kodlama sonrası denetimler
- VMAF kalite kapısı, eşik 80, ilk 30 saniyeden örneklenir
- Yazmadan önce disk alanı denetimi
- Dosya boyutu karşılaştırması; çıktı girdiden büyük veya eşitse orijinal korunur
- Orijinali değiştirmeden önce çıktı doğrulaması

### 🔔 Başarı bildirimleri
- **Sonarr / Radarr** — kütüphane veritabanı kimliğine göre yönlendirilir
- **Bazarr** — altyazıları yeniden taraması için bildirilir
- **Jellyfin** — bilgileri yenilemesi için bildirilir

### 🩹 Hata kurtarma
- ffmpeg hata verirse `prepare_transcode` çağrılıp dosya düzenlenir, sonra yeniden denenir
- Sonsuz döngü koruması; düzenleme adımı dosya başına yalnızca bir kez çalışır

---

## 🛠️ prepare_transcode.json — Dosya Uyumlandırma

Tek başına çalışan bir akış değildir; `hevc_shrink` tarafından, bir dosya dönüştürülemediğinde veya MKV olmadığında çağrılır.

1. 🔀 Katmanları yeniden sıralar, görüntü önce gelir
2. 🗑️ Veri katmanlarını ve gömülü görsel katmanlarını (`png`, `mjpeg`, `bmp`) kaldırır
3. 📦 Migz1Remux ile MKV'ye zorla yeniden paketler
4. 🔊 Türkçe, İngilizce, Japonca ve bilinmeyen dışındaki ses parçalarını siler, yorum parçaları da dahil
5. 💬 Türkçe, İngilizce ve bilinmeyen dışındaki altyazı parçalarını siler, yorum ve işitme engelliler için olanlar da dahil
6. 🏷️ MKV bilgilerini `mkvpropedit` ile yeniler
7. ↩️ Denetimi `hevc_shrink`'e, düzenleme sonrası düğümünde geri verir

> ⚠️ **Dikkat:** Bu akış `goToFlow` eklentisiyle `hevc_shrink`'e geri dönüyor. İçe aktardıktan sonra `hevc_shrink`'in kimliği değişeceği için, **Return to Transcoding** düğümündeki `flowId` ve `pluginId` alanlarını kendi kurulumunuza göre güncellemeniz gerekiyor, aksi halde geri dönüş düğümü bulunamaz.

---

## 🎵 flac_downsample.json — Yüksek Çözünürlüklü Ses Düşürme

Yalnızca FLAC ve WAV dosyalarını hedefler. 24 bit, 96 kHz ve üstü yüksek çözünürlüklü sesi, çoğu kullanım için duyulabilir kalite kaybı yaşatmadan 16 bit 48 kHz FLAC'a çevirir.

### ⏭️ Atlama koşulları
- Uzantı `.flac` veya `.wav` değil
- Dosya zaten işlenmiş
- Genel bit hızı 1 Mbps'in altında, zaten kabul edilebilir aralıkta

### ⚙️ İşlem
```bash
ffmpeg -c:a flac -sample_fmt s16 -ar 48000
```
- Varsa görüntü katmanlarını siler, etiket bilgilerini korur
- Çıktı girdiden büyük veya eşitse çalışma dosyası silinir, orijinal korunur
- Çıktı girdiden küçükse orijinalin yerine geçer

---

## 🧰 Gereksinimler

- [Tdarr](https://docs.tdarr.io/) sunucu ve düğüm
- `ffmpeg` ve `mkvpropedit`, düğüm üzerinde kurulu olmalı
- Topluluk eklentileri:
  - `Tdarr_Plugin_075a_Transcode_Customisable`
  - `Tdarr_Plugin_MC93_Migz1Remux`
  - `Tdarr_Plugin_MC93_Migz3CleanAudio`
  - `Tdarr_Plugin_MC93_Migz4CleanSubs`
- NVENC destekli bir ekran kartı, kodlama adımları `hevc_nvenc` kullanıyor

## 🚀 Kurulum

1. Tdarr arayüzünde Akışlar menüsünden İçe Aktar seçeneğiyle üç dosyayı da yükleyin.
2. `prepare_transcode.json` içindeki Return to Transcoding düğümünün `flowId` ve `pluginId` değerlerini, kendi `hevc_shrink` akışınızın gerçek kimliğiyle güncelleyin, yukarıdaki uyarıyı unutmayın.
3. Sonarr, Radarr, Bazarr ve Jellyfin bildirim düğümlerindeki bağlantı bilgilerini, adres, anahtar ve kütüphane kimliği eşlemelerini kendi kurulumunuza göre düzenleyin.
4. Dil süzgeçlerini kendi tercihlerinize göre değiştirin.
5. Akışları ilgili kütüphanelere atayın.

---

## 📄 Lisans

Kişisel kullanım için paylaşılmıştır, dilediğiniz gibi çatallayıp değiştirebilirsiniz.
