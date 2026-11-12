# Panduan Step-by-Step Streaming Lokal (Mode LAN)

Panduan ini menjelaskan langkah demi langkah menjalankan `local-app` secara lokal dalam mode LAN (`ENGINE_ACCESS_MODE=lan`, tanpa `cloud-app`) hingga live streaming WebRTC (WHEP) tampil di browser, termasuk contoh pengujian menggunakan [koleksi Insomnia](insomnia/README.md).

Untuk referensi teknis lanjutan, lihat [Setup API](setup-api.md) dan [Penerimaan media](acceptance.md).

---

## Prasyarat

- Go 1.25+ (untuk menjalankan via `go run`) atau paket portable Windows.
- `ffmpeg`, `ffprobe`, dan `mediamtx` v1.15.3 (tersedia di `local-app/bin/mediamtx-v1.15.3/mediamtx.exe` untuk Windows).
- Sumber RTSP (kamera/NVR di LAN) **atau** script sumber uji [start-webcam-test.ps1](../scripts/start-webcam-test.ps1).
- Insomnia (dengan koleksi [local-app-testing.json](insomnia/local-app-testing.json)) atau `curl`.
- Browser modern (Chrome/Edge) untuk memutar stream WebRTC WHEP.

---

## Tahap 1 — Siapkan Sumber Video Uji (Opsional jika Belum Ada Kamera/NVR)

Buka **Terminal 1** di direktori `local-app`, lalu terbitkan stream RTSP uji ke `rtsp://127.0.0.1:8555/webcam`:

```powershell
# Opsi A: sumber sintetis (color bar + tone audio) selama 600 detik
./scripts/start-webcam-test.ps1 -Synthetic -DurationSeconds 600

# Opsi B: webcam + mikrofon Windows (DirectShow)
./scripts/start-webcam-test.ps1 -ListDevices
./scripts/start-webcam-test.ps1 -VideoDevice "NAMA_WEBCAM" -AudioDevice "NAMA_MIC"
```

Jika sudah memiliki kamera atau NVR RTSP di jaringan LAN, tahap ini dapat dilewati.

---

## Tahap 2 — Konfigurasi `.env` dan Nyalakan `local-app`

Buka **Terminal 2** di direktori `local-app`:

1. Salin konfigurasi contoh Mode LAN dari [.env.portable.example](../.env.portable.example):

   ```powershell
   Copy-Item .env.portable.example .env
   ```

2. Sesuaikan isi `.env`:
   - `ENGINE_ACCESS_MODE=lan`
   - `HTTP_ADDR=0.0.0.0:8090`
   - `FFMPEG_BIN=ffmpeg` *(atau path lengkap ke `ffmpeg.exe` jika tidak ada di `PATH`)*
   - `FFPROBE_BIN=ffprobe` *(atau path lengkap ke `ffprobe.exe`)*
   - `MEDIAMTX_BIN=./bin/mediamtx-v1.15.3/mediamtx.exe` *(sesuaikan dengan lokasi binary MediaMTX v1.15.3)*
   - `ALLOWED_ORIGINS=http://127.0.0.1:8090,http://localhost:3000,http://127.0.0.1:3000`

3. Jalankan server:

   ```powershell
   go run ./cmd/api -env .env
   ```

4. Pastikan proses HTTP sudah hidup:

   ```powershell
   curl.exe http://127.0.0.1:8090/healthz
   ```

   Endpoint `/healthz` mengembalikan HTTP `200` (`"status": "UP"`). Endpoint `/readyz` masih mengembalikan `503` sampai konfigurasi channel diterapkan melalui setup API.

---

## Tahap 3 — Setup Channel via API (Insomnia)

Import [local-app-testing.json](insomnia/local-app-testing.json) ke Insomnia lalu pilih environment **`Mode LAN (Loopback)`**. Semua request setup **wajib** dikirim ke `http://127.0.0.1:8090` (loopback).

1. **Ambil Nonce & Interface LAN**
   - Jalankan `01 — Setup Mode LAN (Loopback)` → **`01 — Bootstrap nonce & interface LAN`** (`GET /api/v1/setup/bootstrap`).
   - After-response script otomatis mengisi `local_control_nonce`, `lan_ip`, dan `lan_cidr`.
   - Jika PC memiliki lebih dari satu adapter jaringan (misalnya VPN atau VirtualBox), periksa `data.networks` pada response lalu sesuaikan `lan_ip` dan `lan_cidr` dengan adapter LAN yang aktif.

2. **Cek Binary & Encoder**
   - Jalankan **`02 — Cek binary & encoder`** (`GET /api/v1/setup/checks`).
   - Pastikan `FFmpeg`, `ffprobe`, `MediaMTX`, `libx264`, `libopus`, dan `aac` berstatus `OK`.

3. *(Opsional jika memakai NVR ONVIF)* **Discovery & Profiles**
   - Jalankan **`04 — Job: Discover perangkat ONVIF`**, lalu poll **`05 — Poll status job setup`** sampai `state: "SUCCEEDED"`.
   - Isi `onvif_username` dan `onvif_password` di environment, jalankan **`06 — Job: Baca profil ONVIF`**, lalu poll **`05`** untuk memperoleh `source` dan `credential_ref`.

4. **Probe Sumber RTSP**
   - Pastikan `channel_id` (mis. `gantangan01`) dan `channel_source` (mis. `rtsp://127.0.0.1:8555/webcam`) sudah sesuai.
   - Jalankan **`07 — Job: Probe stream (RTSP manual)`** (`202`), lalu poll **`05`** hingga `state: "SUCCEEDED"`.
   - Tentukan `live_mode` dari hasil probe:
     - `video_codec: "h264"` dan `b_frames: 0` → gunakan `live_mode: "copy"`.
     - `video_codec: "hevc"` atau `b_frames > 0` → ubah `live_mode` menjadi `"transcode"`.

5. **Apply Konfigurasi**
   - Jalankan **`08 — Job: Apply konfigurasi channel & LAN`** (`202`), lalu poll **`05`** hingga `state: "SUCCEEDED"`.
   - Jalankan **`00 / 03 — Readiness (/readyz)`**: pastikan HTTP `200` dengan `"ready": true`.

---

## Tahap 4 — Mulai Sesi Media

1. Buka folder **`03 — Media Session (Mode LAN Nonce)`** di Insomnia.
2. Pastikan `session_id` belum pernah dipakai sebelumnya (misalnya `session-lan-001`).
3. Jalankan **`01 — Mulai sesi media (LAN)`** (`POST /api/v1/media/sessions` dengan header `X-Local-Control`):

   ```json
   {
     "id": "session-lan-001",
     "channels": ["gantangan01"],
     "restreams": []
   }
   ```

   Response mengembalikan HTTP `202` dengan `state: "STARTING"`.

4. Poll **`02 — Cek status sesi media (LAN)`** (`GET /api/v1/media/sessions/session-lan-001`) sampai `state` menjadi `"RUNNING"` dan `channel_states[0].state` menjadi `"RUNNING"`.
5. Jalankan **`02 / 03 — Daftar channel & URL playback (Mode LAN)`** (`GET /api/v1/channels`) untuk mendapatkan `whep_url` dan `audio_path` channel aktif.

---

## Tahap 5 — Putar Live Streaming di Browser (WHEP)

Insomnia tidak merender video WebRTC, sehingga playback diuji lewat browser menggunakan endpoint WHEP (`POST /api/v1/playback/{channel}/whep`).

1. Buat file sementara `test-whep.html` di folder luar repo:

   ```html
   <!doctype html>
   <html>
   <body>
     <video id="v" autoplay muted playsinline controls style="width:640px;background:#000"></video>
     <pre id="log"></pre>
     <script>
       const log = (m) => (document.getElementById('log').textContent += m + '\n');
       (async () => {
         const whepUrl = 'http://127.0.0.1:8090/api/v1/playback/gantangan01/whep';
         const pc = new RTCPeerConnection();
         pc.addTransceiver('video', { direction: 'recvonly' });
         pc.addTransceiver('audio', { direction: 'recvonly' });
         pc.ontrack = (e) => {
           document.getElementById('v').srcObject = e.streams[0];
           log('Track diterima: ' + e.track.kind);
         };

         const offer = await pc.createOffer();
         await pc.setLocalDescription(offer);
         await new Promise((resolve) => {
           if (pc.iceGatheringState === 'complete') return resolve();
           pc.onicegatheringstatechange = () => {
             if (pc.iceGatheringState === 'complete') resolve();
           };
         });

         const res = await fetch(whepUrl, {
           method: 'POST',
           headers: { 'Content-Type': 'application/sdp' },
           body: pc.localDescription.sdp,
           credentials: 'omit'
         });
         if (!res.ok) throw new Error('WHEP gagal: HTTP ' + res.status);

         const answerSdp = await res.text();
         await pc.setRemoteDescription({ type: 'answer', sdp: answerSdp });
         log('Streaming aktif! Location: ' + res.headers.get('Location'));
       })().catch((err) => log('Error: ' + err.message));
     </script>
   </body>
   </html>
   ```

2. Sajikan file tersebut pada origin `http://localhost:3000` (sudah diizinkan oleh `ALLOWED_ORIGINS`):

   ```powershell
   python -m http.server 3000
   ```

3. Buka `http://localhost:3000/test-whep.html` di browser. Video live akan tampil di elemen `<video>`.

> **Catatan akses dari perangkat lain di LAN:**
> - Buka **TCP `8090`** (HTTP API, WHEP, WebSocket) dan **UDP `8189`** (WebRTC media) pada firewall PC engine.
> - Ganti `127.0.0.1:8090` pada `whepUrl` dengan `api_urls` dari `GET /api/v1/local/status` (IP LAN PC engine), dan pastikan perangkat viewer berada di subnet `MEDIA_READ_CIDRS` (`lan_cidr`).

---

## Tahap 6 — Uji Grafik/Audio WebSocket (Opsional)

Saat sesi masih `RUNNING`, buka tab **WebSocket Request** baru di Insomnia:

- **Metrik dBFS (JSON text frame):**
  - URL: `ws://127.0.0.1:8090/api/v1/metrics`
  - Header: `Sec-WebSocket-Protocol: edge-v1`
- **PCM Mentah (binary frame 16 kHz mono 16-bit LE):**
  - URL: `ws://127.0.0.1:8090/api/v1/channels/gantangan01/audio`
  - Header: `Sec-WebSocket-Protocol: edge-v1`

---

## Tahap 7 — Stop Sesi & Verifikasi Rekaman

1. Di Insomnia folder **`03 — Media Session (Mode LAN Nonce)`**, jalankan **`03 — Stop sesi media (LAN)`** (`POST /api/v1/media/sessions/session-lan-001/stop`).
2. Poll **`04 — Poll status setelah Stop (LAN)`** hingga `state: "STOPPED"` (melewati `STOPPING` → `FINALIZING` → `STOPPED`). Koneksi player WHEP dan WebSocket otomatis terputus.
3. Buka folder **`05 — Recordings & Restreams`** → jalankan **`01 — Daftar rekaman (Mode LAN Nonce)`** (`GET /api/v1/recordings`) untuk memastikan rekaman MP4 berstatus `state: "READY"`.
