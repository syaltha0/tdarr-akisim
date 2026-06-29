Tdarr Flows

Kendi medya sunucumda kullandığım, üç parçadan oluşan bir Tdarr flow seti. Amaç: kütüphanedeki video ve ses dosyalarını gereksiz yer kaplamadan, kalite kaybını minimumda tutarak transcode etmek; bunu yaparken zaten "yeterince iyi" olan dosyalara dokunmamak.

Bu flow'lar kişisel kütüphanem için ayarlanmış (dil tercihleri, Sonarr/Radarr/Bazarr/Jellyfin entegrasyonu vb.), ama mantığı genel olduğu için herkes ihtiyacına göre uyarlayabilir.

İçindekiler

DosyaGörevhevc_shrink.jsonAna video transcode flow'uprepare_transcode.jsonhevc_shrink'in çağırdığı, dosyayı "conform" eden yardımcı flowflac_downsample.jsonFLAC/WAV hi-res ses dosyalarını düşürme flow'u


hevc_shrink.json — Video Transcode Pipeline

Kütüphanedeki tüm video dosyalarına bakan ana flow.

İşlem yapmadan atlama koşulları:


Dosya zaten işlenmiş (path bazlı dedup)
Genel bitrate < 1.5 Mbps (zaten yeterince küçük)
Dosya zaten HEVC/AV1/VP9 ve çözünürlüğüne göre bitrate eşiğinin altında:

1440p ve üstü: < 8 Mbps
1080p: < 5 Mbps
720p altı: < 4 Mbps





Ön hazırlık (pre-flight):


Junk/font/sidecar görsel dosyalarını siler
Altyazı stream'lerini filtreler (UTF-8 olanları tutar), disposition'ları düzeltir
Girdi dosyasında health check çalıştırır
Dosya MKV değilse önce prepare_transcode flow'unu çağırır, sonra geri döner


Encode (hevc_nvenc, VBR, 2-pass):


1440p ve üstü: 10-bit HEVC, HDR10/HLG metadata korunur
1080p yüksek bitrate (> 5 Mbps): HEVC CQ 22, preset p7
1080p düşük bitrate: re-encode yapılmaz
720p altı + eski codec: HEVC CQ 23


Encode sonrası kontroller:


VMAF kalite kapısı (eşik: 80, ilk 30 saniyeden örneklenir)
Yazmadan önce disk alanı kontrolü
Dosya boyutu karşılaştırması: çıktı, girdiden büyük/eşitse orijinal korunur
Orijinali değiştirmeden önce çıktı doğrulaması


Başarı durumunda bildirimler:


Sonarr veya Radarr'a bildirir (kütüphane DB ID'sine göre yönlendirir)
Bazarr'a altyazıları yeniden taraması için bildirir
Jellyfin'e metadata yenilemesi için bildirir


Hata kurtarma:


ffmpeg hatası alırsa prepare_transcode'u çağırıp dosyayı normalize/conform eder, sonra tekrar dener
Anti-loop koruması: conform adımı dosya başına sadece bir kez çalışır



prepare_transcode.json — Dosya Conform Etme

Tek başına çalışan bir flow değil; hevc_shrink tarafından, bir dosya transcode edilemediğinde veya MKV olmadığında çağrılır.

Adımlar:


Stream'leri yeniden sıralar (video önce)
Data stream'leri ve gömülü görsel stream'leri (png, mjpeg, bmp) kaldırır
Migz1Remux ile MKV'ye zorla remux eder
Şu dillerde olmayan ses parçalarını siler: tur, eng, jpn, und (commentary track'leri de kaldırılır)
Şu dillerde olmayan altyazı parçalarını siler: tur, eng, und (commentary ve SDH dahil kaldırılır)
MKV metadata'sını yeniler (mkvpropedit)
Kontrolü hevc_shrink'e, post-conform node'unda geri verir



Önemli: Bu flow goToFlow plugin'i ile hevc_shrink'e geri dönüyor. Tdarr'a import ettikten sonra hevc_shrink flow'unun ID'si değişeceği için, prepare_transcode.json içindeki "Return to Transcoding" node'unun flowId / pluginId alanlarını kendi kurulumunuza göre güncellemeniz gerekiyor — aksi halde geri dönüş node'u bulunamaz.




flac_downsample.json — Hi-Res Ses Düşürme

Sadece FLAC ve WAV dosyalarını hedefler. Hi-res ses (24-bit, 96kHz+) dosyalarını, çoğu kullanım için duyulabilir kalite kaybı olmadan 16-bit 48kHz FLAC'a çevirir.

Atlama koşulları:


Uzantı .flac veya .wav değil
Zaten işlenmiş (path bazlı dedup)
Genel bitrate < 1 Mbps (zaten kabul edilebilir aralıkta)


İşlem:

ffmpeg -c:a flac -sample_fmt s16 -ar 48000


Varsa video stream'lerini siler, metadata tag'lerini korur
Çıktı, girdiden büyük/eşitse: working file silinir, orijinal korunur
Çıktı girdiden küçükse: orijinalin yerine geçer



Gereksinimler


Tdarr (Server + Node)
ffmpeg, mkvpropedit (mkvtoolnix) node üzerinde kurulu olmalı
Community plugin'leri:

Tdarr_Plugin_075a_Transcode_Customisable
Tdarr_Plugin_MC93_Migz1Remux
Tdarr_Plugin_MC93_Migz3CleanAudio
Tdarr_Plugin_MC93_Migz4CleanSubs



NVENC destekli bir GPU (HEVC encode adımları hevc_nvenc kullanıyor)


Kurulum


Tdarr arayüzünde Flows → Import ile üç dosyayı da import edin.
prepare_transcode.json'daki Return to Transcoding node'unun flowId/pluginId değerlerini, kendi hevc_shrink flow'unuzun gerçek ID'siyle güncelleyin (yukarıdaki not).
Sonarr/Radarr/Bazarr/Jellyfin bildirim node'larındaki bağlantı bilgilerini (URL, API key, library DB ID eşlemeleri) kendi kurulumunuza göre düzenleyin.
Dil filtrelerini (tur,eng,jpn,und vb.) kendi tercihlerinize göre değiştirin.
Flow'ları ilgili kütüphanelere atayın.


Lisans

Kişisel kullanım için paylaşılmıştır, dilediğiniz gibi çatallayıp (fork) değiştirebilirsiniz.
