
---

### 2. Sederhanakan Komputasi Media Lokal (Tetap Wajib Diperbaiki)

Meskipun logika bisnisnya online, hukum pemrosesan hardware lokal tetap berlaku. Tiga optimasi ini tetap harus dilakukan agar edge server stabil:

#### A. Pangkas Subprocess FFmpeg (Gabungkan Audio & DVR)

Jangan jalankan 24 FFmpeg untuk audio ditambah 24 FFmpeg untuk rekaman (total 48 child process).

* Jalankan **1 proses FFmpeg per kamera** dengan dual-output:
* Output 1 $\rightarrow$ Langsung simpan file `.mp4` (stream-copy, 0% CPU).
* Output 2 $\rightarrow$ Pipe PCM audio mentah ke Go untuk menghitung metrik dBFS.




* **Keuntungan:** Menghemat 50% CPU and RAM edge server, serta mengurangi separuh socket TCP loopback ke MediaMTX.



#### B. Gunakan Fragmented MP4 (`fMP4`) untuk Rekaman

Karena internet bisa sewaktu-waktu *error* dan pertandingan harus dihentikan tiba-tiba, kemungkinan sesi rekaman dimatikan secara mendadak sangat tinggi:

* Jangan pakai MP4 standar yang mengandalkan pengiriman huruf `q` di stdin.


* Pasang flag `-movflags +frag_keyframe+empty_moov+default_base_moof`.
* **Efeknya:** Jika internet mati lalu server dimatikan paksa (*force stop*), video rekaman dari ronde yang sempat berjalan tetap utuh dan valid untuk diunggah nanti ke cloud storage.



#### C. Matikan Transcoding pada Restream YouTube

* Untuk restream ke YouTube/sosmed, pastikan parameter ingest kamera sudah H.264 + AAC.
* Gunakan flag `-c copy` tanpa merender ulang video (`libx264`).


* CPU mini PC Anda bisa fokus sepenuhnya untuk routing jaringan dan WebSocket audio/metrik ke cloud.



---

### Alur yang Lebih Ramping untuk Model Full-Online

```text
[ Browser Juri ]                [ Cloud API ]
       │ (1. Minta Tiket)             │
       ├─────────────────────────────>│
       │<─────────────────────────────┤ (2. Tiket Diberikan)
       │                              │
       │ (3. WHEP SDP + Tiket)        │
       ▼                              │
[ Go WHEP Gateway ] ── (Validasi) ────┘ (Jika internet mati -> Tolak)
       │ (Tiket Valid)
       ▼
[ MediaMTX ] ──(WebRTC LAN)──> [ Browser Juri ]
       │
       ▼ (24x Stream Lokal)
[ 24x FFmpeg Pipeline Tunggal ]
       ├── Output File : MP4 Frag (Siap di-upload ke S3 saat sesi selesai)
       └── Output Pipe : PCM Mono 16kHz ──> [ Go Audio Meter ] ──> Sync Metrik ke Cloud

```

Dengan bentuk ini, sistem Anda tetap **100% patuh pada kendali cloud**, tetapi tidak lagi membuang-buang daya komputasi di edge server lokal.