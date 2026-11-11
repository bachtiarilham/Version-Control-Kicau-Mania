# API setup local-app untuk frontend terpisah

Dokumen ini menjelaskan backend konfigurasi, tes koneksi, live dan metrik. **Local-app tidak menyertakan UI, halaman setup, dashboard, atau pembuka browser.** Frontend/Insomnia mengonsumsi API di bawah. Tidak ada perubahan cloud-app atau database bisnis.

## 1. Menjalankan backend

Paket portable Windows memakai `.env` dari `.env.portable.example` dan `Start-local-app.cmd`. Tidak perlu Docker atau Go pada mini PC. Untuk development, isi path FFmpeg/ffprobe/MediaMTX lalu jalankan:

```powershell
go run ./cmd/api -env .env.portable.example
```

Mode baru harus diaktifkan eksplisit dengan `ENGINE_ACCESS_MODE=lan`. Konfigurasi lama tanpa pilihan ini tetap mode `cloud`, termasuk validasi tiket playback. Mode LAN dapat mulai dengan `CHANNELS_JSON=[]`; HTTP tetap hidup saat konfigurasi/binary/sumber media belum siap. `GET /healthz` hidup, sedangkan `/readyz` mengembalikan 503 sampai runtime media siap.

Port aplikasi tetap mengikuti `HTTP_ADDR`, default portable `0.0.0.0:8090`. Port yang terpakai menghasilkan error, bukan dipindahkan diam-diam. Pencarian otomatis perangkat memakai ONVIF WS-Discovery, bukan pemindaian semua port atau tebakan URL vendor. MediaMTX internal tetap loopback; izinkan TCP 8090 dan UDP 8189 pada jaringan arena bila viewer berada di perangkat lain. Konfigurasi router/firewall tidak diubah otomatis.

`API_TOKEN` boleh kosong khusus mode LAN; backend membuat credential layanan acak pada proses tersebut. Frontend tidak memerlukan dan tidak menerima credential ini. CLI/integrasi layanan yang membutuhkan Bearer stabil tetap dapat mengisinya. Jangan memakai mode LAN pada listener yang diekspos ke internet.

## 2. Kontrol lokal dan viewer

Setup dan Mulai/Stop tanpa akun hanya tersedia melalui koneksi **loopback langsung** dengan Host `localhost`/IP loopback. Alamat API untuk kontrol pada PC engine misalnya `http://127.0.0.1:8090`. Header forwarded tidak dapat membuat klien LAN menjadi operator lokal.

Frontend operator yang terpisah harus disajikan dari origin loopback yang ada di `ALLOWED_ORIGINS`, misalnya `http://localhost:3000`. Origin cloud/komputer lain tidak memperoleh kontrol lokal walaupun masuk daftar CORS. Viewer LAN tidak menerima token kontrol.

1. GET `/api/v1/setup/bootstrap` dari PC engine. Simpan `data.nonce` **di memori**, bukan URL/localStorage. Response juga berisi interface LAN.
2. Kirim `X-Local-Control: <nonce>` untuk POST/DELETE setup dan POST start/stop sesi.
3. Nonce berubah ketika backend restart. Ambil bootstrap baru jika request ditolak.

Request tanpa Origin tetap bisa dipakai Insomnia/CLI pada loopback. Middleware tetap mendukung Bearer layanan untuk endpoint kontrol media yang lama. Endpoint setup sendiri tetap memerlukan batas loopback dan nonce pada mutasi.

Gunakan HTTP client frontend khusus engine dengan `credentials: "omit"`/`withCredentials: false`. API lokal memakai header nonce/Bearer, bukan cookie login cloud; jangan menyalin konfigurasi cookie atau token akun cloud ke client engine.

Viewer pada subnet terpilih dapat membaca katalog, status, WebSocket audio/metrik, dan membuka WHEP selama sesi aktif. Mereka tidak dapat membaca kredensial/setup atau mengubah sesi. Endpoint rekaman dan kontrol restream tetap memakai akses operator/layanan.

## 3. Endpoint dan urutan setup

Semua response REST memakai `success/message/data/errors`; 204 tidak memiliki body. Periksa `X-Request-ID` ketika meminta bantuan. Password, URL berisi autentikasi, command FFmpeg, dan stderr mentah tidak dikirim dalam diagnosis.

| Endpoint | Hasil |
| --- | --- |
| GET `/api/v1/setup/bootstrap` | Nonce operator dan `networks`: name, ip, cidr. |
| GET `/api/v1/setup/checks` | Keberadaan binary/encoder, code, message, advice. |
| GET `/api/v1/setup/configuration` | Konfigurasi tersimpan/import `.env`, tanpa userinfo atau query sensitif sumber. |
| POST `/api/v1/setup/jobs` | Mulai pekerjaan, HTTP 202, `data.id`. |
| GET `/api/v1/setup/jobs/:id` | Progres, hasil, atau error aman. |
| DELETE `/api/v1/setup/jobs/:id` | Minta pembatalan, 204; periksa hasil job untuk mengetahui apakah sudah commit. |
| GET `/api/v1/setup/guidance/:code` | Saran untuk kode diagnosis. |
| GET `/api/v1/local/status` | Kesiapan lokal, channel tanpa URL sumber, sesi terakhir/aktif, target restream, `api_urls`, dan hasil pemeriksaan startup. |
| GET `/api/v1/local/guidance/:code` | Panduan diagnosis yang aman untuk viewer pada CIDR LAN. |

Job mempunyai state `RUNNING`, `SUCCEEDED`, atau `FAILED`. Poll GET job sekitar satu detik. Job gagal berisi `code`, `message`, dan `advice`; ini tetap response GET HTTP 200 yang berhasil membaca status pekerjaan. Batas dua job bersamaan, timeout lima menit, dan hasil/reference kredensial tersimpan sementara sekitar 15 menit. Muat profil kembali bila reference kedaluwarsa.

Job/reference tidak persisten. Setelah backend restart, ambil bootstrap baru lalu baca configuration/status sebelum memutuskan mengulang pekerjaan. Error REST terklasifikasi dapat memiliki `errors.code` dan `errors.request_id`; gangguan channel sesi memakai `error_code`. Penghentian otomatis karena disk/durasi dilaporkan sebagai `stop_reason`.

### Temukan perangkat

POST jobs:

```json
{"action":"discover","ip":"192.168.1.20"}
```

`ip` harus milik interface PC engine yang dipilih dari bootstrap. Discovery mengirim multicast pada interface tersebut; hasil berupa kandidat `endpoint` dan nama/IP. Jika perangkat tidak ditemukan, gunakan alamat ONVIF manual pada langkah berikutnya. Discovery tidak membuktikan bahwa semua NVR mendukung ONVIF atau setiap channel tersedia.

### Baca profil

```json
{
  "action":"profiles",
  "ip":"192.168.1.20",
  "endpoint":"http://192.168.1.10:80/onvif/device_service",
  "username":"AKUN_ONVIF",
  "password":"PASSWORD_ONVIF"
}
```

Adapter hanya membaca layanan Media1/Media2, profil, dan URI stream. Tidak mengubah encoder perangkat. Layanan harus berada di subnet terpilih; redirect ditolak. HTTPS perangkat harus mempunyai sertifikat yang dipercaya. WS-Security PasswordDigest dan HTTP Digest MD5/auth didukung; implementasi ONVIF/profil autentikasi lain yang tidak kompatibel memerlukan RTSP manual.

Hasil profil berisi token, nama, codec/resolusi jika tersedia, source yang telah disanitasi, dan `credential_ref` opaque. Pilih profil secara eksplisit; satu kamera bisa memiliki beberapa profil main/substream. Jangan otomatis menganggap satu profil sama dengan satu gantangan.

### Tes stream

```json
{
  "action":"probe",
  "channel":{
    "id":"gantangan01",
    "source":"rtsp://192.168.1.10:554/path",
    "credential_ref":"REFERENCE_DARI_PROFIL",
    "live_mode":"copy"
  }
}
```

RTSP manual tidak memerlukan credential_ref. Username/password dapat dikirim terpisah pada objek channel, atau dalam URL yang dimasukkan operator. Reference mengacu URI privat yang dibaca dari profil; hapus reference jika mengganti URL manual.

Tes mengambil sampel paket nyata melalui ffprobe. Hasil mencakup `video_codec`, `width`, `height`, `fps`, `b_frames`, `has_audio`, `audio_codec`, dan `tested_at`. `video_codec=hevc` berarti H.265. `has_audio=false` bukan kegagalan video. Hasil koneksi TCP saja tidak dianggap valid.

### Simpan dan terapkan

```json
{
  "action":"apply",
  "configuration":{
    "version":1,
    "lan_ip":"192.168.1.20",
    "lan_cidr":"192.168.1.0/24",
    "channels":[
      {"id":"gantangan01","source":"rtsp://192.168.1.10:554/path","credential_ref":"REFERENCE_DARI_PROFIL","live_mode":"copy"}
    ]
  }
}
```

Konfigurasi memuat 1–24 ID unik. Mode `copy` memerlukan H.264 tanpa B-frame. Mode `transcode` menghasilkan H.264 baseline 640×360, 15 FPS, menggunakan CPU; tidak mengubah encoder NVR. Recording tetap menyimpan codec video sumber dan AAC bila ada audio.

Apply memeriksa ulang channel dan binary/encoder yang diperlukan. Hanya dilakukan saat tidak ada sesi aktif/finalisasi, kemudian runtime diganti dan konfigurasi di-flush/rename. Jika penerapan gagal, runtime sebelumnya dicoba dipulihkan. Job timeout/cancel sebelum commit dapat dihentikan; cancel setelah commit bukan pembatalan konfigurasi yang sudah tersimpan.

Seluruh konfigurasi disimpan terenkripsi DPAPI pada `DATA_DIR/setup.json`. Jalankan dengan akun Windows yang sama; pindah mesin/akun memerlukan input ulang. File konfigurasi lama dari CHANNELS_JSON hanya menjadi draft awal, tidak dihapus/ditimpa otomatis. Response konfigurasi menghilangkan userinfo dan query URI. Saat mengirim kembali channel dengan ID/host/path sama, credential/query tersimpan dipertahankan jika tidak diganti. Mengganti host/path memerlukan credential/ref baru.

Pada startup berikutnya backend memuat konfigurasi, menyalakan runtime, dan membuat job pemeriksaan sumber otomatis. Status menyediakan `checks_job_id` dan `source_checks`; `tested_at` bukan bukti bahwa sumber selamanya sehat. Job `{"action":"recheck"}` dapat dijalankan manual. Recording tidak dimulai otomatis.

Jika interface tersimpan belum aktif, bootstrap dicoba lagi setiap lima detik dan API tetap hidup. Jika alamat/subnet PC benar-benar berubah, pilih kembali interface lalu terapkan konfigurasi; alamat ICE tidak diganti secara spekulatif.

## 4. Menghubungkan frontend: sesi, video dan grafik

Frontend menekan Mulai dengan POST `/api/v1/media/sessions`:

```json
{"id":"session-001","channels":["gantangan01"],"restreams":[]}
```

Gunakan ID baru setiap sesi. HTTP 202 berarti pekerjaan diterima; poll status sesi sampai channel berjalan. Target restream yang tersedia ada pada local/status dan berasal dari `RESTREAM_TARGETS_JSON`, bukan stream key dari browser. Contoh pilihan: `{"channel_id":"gantangan01","target":"youtube"}`.

Video browser dibuka melalui WHEP `/api/v1/playback/{id}/whep` menggunakan SDP offer. Mode LAN tidak memerlukan tiket cloud, tetapi peer harus berasal dari CIDR arena dan channel harus termasuk sesi aktif. Simpan Location/ETag response untuk request lanjutan/DELETE sesi playback. MediaMTX HTTP/RTSP internal tidak dipublikasikan langsung.

URL playback katalog memakai `LOCAL_PUBLIC_URL` jika disediakan, atau origin HTTP dari IP LAN terpilih dan port API. `api_urls` berisi origin engine, bukan alamat halaman frontend. Dengan demikian perangkat viewer tidak diarahkan ke loopback-nya sendiri.

Metadata channel pada local/status membedakan `video_codec` sumber dari `live_video_codec`/`live_audio_codec`. Width/height berasal dari sumber; mode transcode live tetap 640×360. Jangan menganggap codec rekaman H.265 berarti browser harus mendukung H.265.

Audio live dinormalisasi menjadi Opus bila tersedia. Sumber tanpa audio menghasilkan video-only; tidak dibuat audio senyap palsu. Frontend dapat menjalankan 24 player muted, lalu mengaktifkan suara satu channel melalui tindakan pengguna. Aplikasi ini hanya menyediakan backend; player, grid dan chart dibuat pada frontend terpisah.

Grafik memakai satu WebSocket `/api/v1/metrics` dengan subprotocol `edge-v1` untuk seluruh channel sesi. Pesan text JSON memuat:

```json
{"channel_id":"gantangan01","at":"2026-11-11T03:00:00Z","peak_dbfs":-12.5,"rms_dbfs":-24.1,"active_ms":180,"window_ms":200}
```

PCM opsional tersedia pada `/api/v1/channels/{id}/audio`, binary mono signed 16-bit little-endian 16 kHz. Metrik adalah dBFS, bukan dB SPL terkalibrasi. Periksa has_audio dan timestamp, tampilkan data tertinggal secara jujur, dan batasi history frontend. Tidak ada PCM/metrik untuk sumber tanpa audio.

Stop memakai POST `/api/v1/media/sessions/:id/stop`, HTTP 202. Tutup player/socket pada frontend ketika STOPPING. Engine menutup admission serta playback, menghentikan restream/pipeline, lalu FINALIZING. Jangan mengizinkan Mulai baru sebelum status terminal. Upload tetap berjalan sesudah STOPPED. Setelah crash/restart, sesi sebelumnya ditandai INTERRUPTED dan tidak dilanjutkan otomatis.

## 5. Paket portable dan verifikasi

Developer membuat paket dengan `scripts/package-portable.ps1`, memberikan path binary FFmpeg, ffprobe, MediaMTX **v1.15.3**, serta direktori lisensi/source-notice build FFmpeg. Script membangun executable Windows AMD64, menyalin binary/config/panduan/lisensi dan checksum ke direktori baru. Tidak menyalin `.env` aktif, credential, recording, atau runtime pengguna. Kebutuhan distribusi FFmpeg mengikuti build yang disediakan developer.

End user menjalankan Start-local-app.cmd. Tidak ada UI atau kompilasi/unduhan saat startup. Direktori data dan runtime harus dapat ditulis. Jika port digunakan atau instance lain aktif, launcher mempertahankan pesan kegagalan. Instalasi headless Linux mode cloud yang sudah ada tetap tersedia; penyimpanan konfigurasi portable DPAPI ditargetkan ke Windows.

Pemeriksaan implementasi memakai build/vet dan alur sintetis terisolasi, tanpa suite tes Go. ONVIF NVR asli, firewall LAN, playback browser dari frontend terpisah, S3 asli, dan kemampuan 24 stream perlu penerimaan perangkat nyata. Model mini PC/NVR belum diketahui; konversi H.265 semua channel belum dinyatakan layak operasional hanya karena build berhasil.

## Swagger interaktif

Buka `http://localhost:8090/api/docs/index.html` setelah menjalankan executable. Swagger tersedia sebelum NVR dikonfigurasi. Jalankan bootstrap, salin `data.nonce`, lalu isi Authorize > LocalControl. Mode LAN menggunakan localhost untuk setup; membuka Swagger melalui IP LAN tidak menghapus pembatasan tersebut. Request di Try it out benar-benar dijalankan. WHEP/WebSocket didokumentasikan tetapi memerlukan client media terpisah. Kontrak JSON tersedia di `/api/docs/doc.json`; aset Swagger sudah tertanam di executable.

## Referensi konfigurasi environment

| Env | Fungsi |
| --- | --- |
| API_TOKEN | Kredensial kontrol engine; jangan dibagikan kepada browser pengguna. |
| ARENA_ID / ENGINE_ID | Identitas engine yang didaftarkan pada cloud. |
| CHANNELS_JSON | Sumber kamera/NVR RTSP dan ID channel. |
| RESTREAM_TARGETS_JSON | Target restream dan kredensial layanan. |
| FRONTEND_ORIGIN | Origin frontend, misalnya https://dev.megaallstararena.com. |
| LOCAL_PUBLIC_URL | HTTPS engine LAN, misalnya https://lan-dev.megaallstararena.com. |
| WEBRTC_HOSTS | IP LAN mini PC untuk kandidat ICE. |
| CLOUD_ENGINE_TOKEN | Kredensial mesin untuk panggilan cloud. |
| CLOUD_COMMANDS_URL | Polling command; hasil ke akhiran /results. |
| CLOUD_HEARTBEAT_URL | Heartbeat, interval default 5 detik. |
| CLOUD_MEDIA_UPDATES_URL | Batch metadata rekaman/upload. |
| CLOUD_PLAYBACK_AUTHORIZE_URL | Validasi tiket playback pengguna. |
| HEARTBEAT_TIMEOUT | Batas heartbeat dianggap segar, default 15 detik. |
| DATA_DIR | Rekaman/manifest, journal command, dan antrean metadata. |
| FFPROBE_BIN | ffprobe untuk validasi segmen/MP4, default `ffprobe`. |
| RUNTIME_DIR | Konfigurasi MediaMTX dan file runtime. |
| S3_* | Bucket, endpoint, region, dan kredensial upload. |

Endpoint cloud baru harus mengikuti kontrak pendamping. Implementasi cloud untuk heartbeat, metadata, penerbitan tiket, dan otorisasi playback belum disertakan dalam perubahan engine ini. Playback menolak sesi baru jika otorisasi cloud belum tersedia.

## Domain dan HTTPS untuk frontend cloud

Browser operator/juri login ke cloud lalu meminta tiket live. Frontend membuka `LOCAL_PUBLIC_URL/api/v1/playback/{channel}/whep` dengan Authorization. Local-app memvalidasi tiket ke cloud dan meneruskan signaling ke MediaMTX loopback. Media WebRTC mengalir melalui LAN.

Perangkat harus mempunyai rute ke LAN arena. Sertifikat HTTPS engine harus dipercaya, origin harus diizinkan, dan browser mungkin meminta izin jaringan lokal. API_TOKEN dan CLOUD_ENGINE_TOKEN tidak dibagikan ke browser.

Gunakan `.env.lan.example`, `scripts/configure-lan.ps1`, lalu `scripts/start-lan.ps1`. Domain, IP, DNS/hosting Hostinger, dan model TP-Link Archer C6 dapat dikustomisasi di env. Parameter router tidak otomatis mengubah pengaturan router.

DNS domain engine harus mengarah ke IP LAN mini PC dari perangkat arena. Jika memakai DNS bawaan, arahkan DNS jaringan ke mini PC melalui router/DHCP. Sertifikat publik membutuhkan konfigurasi DNS/ACME yang benar. Caddy meneruskan API/playback ke Go; HTTP MediaMTX dan RTSP internal tetap loopback. Izinkan HTTPS serta UDP WebRTC pada firewall LAN.
