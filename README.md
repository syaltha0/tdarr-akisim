# 🎬 Tdarr Flows

<p>
  <img src="https://img.shields.io/badge/Tdarr-Flow%20Set-orange?style=flat-square" alt="Tdarr">
  <img src="https://img.shields.io/badge/Encoder-HEVC%20NVENC-blue?style=flat-square" alt="HEVC NVENC">
  <img src="https://img.shields.io/badge/Audio-FLAC%2FWAV-9cf?style=flat-square" alt="FLAC/WAV">
  <img src="https://img.shields.io/badge/License-Personal%2FFork%20Friendly-lightgrey?style=flat-square" alt="License">
</p>

Kendi medya sunucumda kullandığım, üç parçadan oluşan bir **Tdarr** flow seti.
Amaç basit: kütüphanedeki video ve ses dosyalarını **gereksiz yer kaplamadan**, **kalite kaybını minimumda tutarak** transcode etmek — ve zaten "yeterince iyi" olan dosyalara hiç dokunmamak.

Bu flow'lar kendi kütüphanem için ayarlanmış (dil tercihleri, Sonarr/Radarr/Bazarr/Jellyfin entegrasyonu vb.) ama mantığı genel olduğu için herkes ihtiyacına göre fork'layıp uyarlayabilir.

---

## 📦 İçindekiler

| Dosya | Görev |
|---|---|
| 🎞️ [`hevc_shrink.json`](#️-hevc_shrinkjson--video-transcode-pipeline) | Ana video transcode flow'u |
| 🛠️ [`prepare_transcode.json`](#️-prepare_transcodejson--dosya-conform-etme) | `hevc_shrink`'in çağırdığı, dosyayı "conform" eden yardımcı flow |
| 🎵 [`flac_downsample.json`](#-flac_downsamplejson--hi-res-ses-düşürme) | FLAC/WAV hi-res ses dosyalarını düşürme flow'u |

```
                ┌─────────────────────┐
   video ─────► │   hevc_shrink.json   │ ─────► Sonarr/Radarr ✓
                └──────────┬──────────┘         Bazarr ✓
                           │ (hata / MKV değilse)  Jellyfin ✓
                           ▼
                ┌─────────────────────┐
                │ prepare_transcode    │ ── geri döner ──► hevc_shrink
                └─────────────────────┘

   flac/wav ───► ┌─────────────────────┐
                 │ flac_downsample.json │
                 └─────────────────────┘
```

---

## 🎞️ hevc_shrink.json — Video Transcode Pipeline

Kütüphanedeki tüm video dosyalarına bakan ana flow.

### ⏭️ Atlama koşulları (işlem yapmadan geçer)
- Dosya zaten işlenmiş *(path bazlı dedup)*
- Genel bitrate **< 1.5 Mbps** *(zaten yeterince küçük)*
- Dosya zaten HEVC/AV1/VP9 ve çözünürlüğe göre eşiğin altında:

  | Çözünürlük | Eşik |
  |---|---|
  | 1440p ve üstü | < 8 Mbps |
  | 1080p | < 5 Mbps |
  | 720p altı | < 4 Mbps |

### ✈️ Ön hazırlık (pre-flight)
- Junk / font / sidecar görsel dosyalarını siler
- Altyazı stream'lerini filtreler (UTF-8 olanları tutar), disposition'ları düzeltir
- Girdi dosyasında health check çalıştırır
- Dosya **MKV değilse** önce `prepare_transcode` flow'unu çağırır, sonra geri döner

### ⚙️ Encode (hevc_nvenc, VBR, 2-pass)

| Durum | Ayar |
|---|---|
| 1440p ve üstü | 10-bit HEVC, HDR10/HLG metadata korunur |
| 1080p, yüksek bitrate (> 5 Mbps) | HEVC CQ 22, preset `p7` |
| 1080p, düşük bitrate | re-encode yapılmaz |
| 720p altı + eski codec | HEVC CQ 23 |

### ✅ Encode sonrası kontroller
- VMAF kalite kapısı *(eşik: 80, ilk 30 saniyeden örneklenir)*
- Yazmadan önce disk alanı kontrolü
- Dosya boyutu karşılaştırması — çıktı ≥ girdi ise orijinal korunur
- Orijinali değiştirmeden önce çıktı doğrulaması

### 🔔 Başarı bildirimleri
- **Sonarr / Radarr** → kütüphane DB ID'sine göre yönlendirir
- **Bazarr** → altyazıları yeniden taraması için
- **Jellyfin** → metadata yenilemesi için

### 🩹 Hata kurtarma
- ffmpeg hatası alırsa `prepare_transcode`'u çağırıp dosyayı normalize/conform eder, sonra tekrar dener
- **Anti-loop koruması:** conform adımı dosya başına sadece bir kez çalışır

---

## 🛠️ prepare_transcode.json — Dosya Conform Etme

> Tek başına çalışan bir flow **değil**; `hevc_shrink` tarafından, bir dosya transcode edilemediğinde veya MKV olmadığında çağrılır.

1. 🔀 Stream'leri yeniden sıralar *(video önce)*
2. 🗑️ Data stream'leri ve gömülü görsel stream'leri (`png`, `mjpeg`, `bmp`) kaldırır
3. 📦 Migz1Remux ile MKV'ye zorla remux eder
4. 🔊 Şu dillerde olmayan ses parçalarını siler: `tur` `eng` `jpn` `und` *(commentary dahil)*
5. 💬 Şu dillerde olmayan altyazı parçalarını siler: `tur` `eng` `und` *(commentary + SDH dahil)*
6. 🏷️ MKV metadata'sını yeniler *(`mkvpropedit`)*
7. ↩️ Kontrolü `hevc_shrink`'e, post-conform node'unda geri verir

> ⚠️ **Dikkat:** Bu flow `goToFlow` plugin'i ile `hevc_shrink`'e geri dönüyor. Import ettikten sonra `hevc_shrink`'in ID'si değişeceği için, **"Return to Transcoding"** node'undaki `flowId` / `pluginId` alanlarını kendi kurulumunuza göre güncellemeniz gerekiyor — aksi halde geri dönüş node'u bulunamaz.

---

## 🎵 flac_downsample.json — Hi-Res Ses Düşürme

Sadece **FLAC** ve **WAV** dosyalarını hedefler. Hi-res ses *(24-bit, 96kHz+)* dosyalarını, çoğu kullanım için duyulabilir kalite kaybı olmadan **16-bit 48kHz FLAC**'a çevirir.

### ⏭️ Atlama koşulları
- Uzantı `.flac` veya `.wav` değil
- Zaten işlenmiş *(path bazlı dedup)*
- Genel bitrate **< 1 Mbps** *(zaten kabul edilebilir aralıkta)*

### ⚙️ İşlem
```bash
ffmpeg -c:a flac -sample_fmt s16 -ar 48000
```
- Varsa video stream'lerini siler, metadata tag'lerini korur
- Çıktı ≥ girdi ise: working file silinir, orijinal korunur
- Çıktı < girdi ise: orijinalin yerine geçer

---

## 🧰 Gereksinimler

- [Tdarr](https://docs.tdarr.io/) *(Server + Node)*
- `ffmpeg`, `mkvpropedit` (mkvtoolnix) — node üzerinde kurulu olmalı
- Community plugin'leri:
  - `Tdarr_Plugin_075a_Transcode_Customisable`
  - `Tdarr_Plugin_MC93_Migz1Remux`
  - `Tdarr_Plugin_MC93_Migz3CleanAudio`
  - `Tdarr_Plugin_MC93_Migz4CleanSubs`
- NVENC destekli bir GPU *(HEVC encode adımları `hevc_nvenc` kullanıyor)*

## 🚀 Kurulum

1. Tdarr arayüzünde **Flows → Import** ile üç dosyayı da import edin.
2. `prepare_transcode.json`'daki **Return to Transcoding** node'unun `flowId`/`pluginId` değerlerini, kendi `hevc_shrink` flow'unuzun gerçek ID'siyle güncelleyin *(yukarıdaki uyarı)*.
3. Sonarr/Radarr/Bazarr/Jellyfin bildirim node'larındaki bağlantı bilgilerini (URL, API key, library DB ID eşlemeleri) kendi kurulumunuza göre düzenleyin.
4. Dil filtrelerini (`tur,eng,jpn,und` vb.) kendi tercihlerinize göre değiştirin.
5. Flow'ları ilgili kütüphanelere atayın.

---

## 📄 Lisans

Kişisel kullanım için paylaşılmıştır, dilediğiniz gibi fork'layıp değiştirebilirsiniz.
