**Ada alur yang belum saya jelaskan lengkap. Local-app sudah layak untuk uji integrasi terbatas, tetapi belum bisa dinyatakan siap operasional 24 channel.** Kendala terbesar saat ini adalah belum ada bukti playback browser dengan NVR dan jaringan arena yang sebenarnya.

Beberapa alur penting yang sebelumnya belum dijelaskan:

| Alur | Perilaku yang perlu dipahami frontend |
|---|---|
| **Startup dan konfigurasi tersimpan** | API bisa hidup sebelum media siap. Konfigurasi lama dimuat kembali, tetapi sesi recording/live tidak otomatis dimulai setelah restart. |
| **Pekerjaan asinkron** | Discovery/probe/apply dan Mulai/Stop tidak selesai dalam satu request. HTTP `202` berarti diterima; frontend harus membaca hasil job atau status sesi. |
| **Kesiapan playback** | API berhasil → pipeline `RUNNING` → browser benar-benar menampilkan video adalah tiga pemeriksaan berbeda. SDP answer berhasil belum membuktikan koneksi media berhasil. |
| **ICE dan koneksi browser** | Setelah pertukaran SDP, browser masih memeriksa jalur jaringan dan membangun koneksi WebRTC. IP kandidat atau firewall UDP yang salah bisa membuat layar kosong meskipun REST normal. |
| **Refresh atau tutup halaman** | Menutup player tidak sama dengan Stop sesi engine. Frontend harus menutup koneksi WebRTC dan menghapus resource WHEP; sesi recording tetap berjalan sampai dihentikan. |
| **Sumber terputus** | Engine mencoba ulang channel yang gagal. Frontend perlu menampilkan gangguan dan menangani koneksi player yang terputus; pemulihan sumber belum menjamin player otomatis tersambung kembali. |
| **Stop dan finalisasi** | Stop menutup media, lalu menggabungkan dan memvalidasi rekaman. Frontend harus menunggu status final, bukan langsung menganggap selesai setelah menerima `202`. |
| **Restart/crash** | Sesi sebelumnya menjadi `INTERRUPTED`; engine mencoba memulihkan rekaman yang tersedia, tanpa otomatis melanjutkan media. |
| **Upload** | Recording selesai berbeda dengan upload selesai. Upload berjalan di background dan dicoba ulang jika gagal. |

Untuk kesiapan sekarang, bukti yang ada adalah:

| Bagian | Penilaian berdasarkan pemeriksaan yang sudah dilakukan |
|---|---|
| API, Swagger, build dan vet | **Sudah diperiksa.** |
| Pipeline recording, audio, video-only dan konversi H.265 | **Sudah dicoba dengan sumber sintetis.** |
| Start/Stop, pemulihan sumber dan restart | **Sudah dicoba pada lingkungan terisolasi.** |
| WHEP | **Sudah sampai SDP answer dan pengelolaan resource**, belum membuktikan playback browser nyata. |
| ONVIF NVR arena | **Belum diuji dengan perangkat aslinya.** Simulator Media1 bukan bukti kompatibilitas semua NVR. |
| Frontend → engine → video/audio tampil | **Belum terbukti end-to-end pada jaringan arena.** |
| 24 channel dan beberapa penonton | **Belum diuji.** Spesifikasi mini PC juga belum diketahui. |
| Restream eksternal dan S3 nyata | **Belum diterima pada layanan sebenarnya.** |
| Integrasi kontrol/tiket cloud | **Belum lengkap:** endpoint pendamping cloud belum aktif. |

Bukti dan batas pemeriksaannya tercatat di [dokumen penerimaan](D:/Workdir/Go/kicau-mania/local-app/docs/acceptance.md). Suite tes Go tidak dijalankan sesuai permintaanmu sebelumnya.

**Untuk Sabtu, target yang masuk akal adalah membuktikan satu jalur lengkap terlebih dahulu:**

1. Satu channel NVR berhasil diprobe.
2. Frontend menjalankan Mulai dan menampilkan video/audio nyata.
3. Satu perangkat lain melalui WiFi dapat menonton.
4. Stop benar-benar menghentikan playback dan menghasilkan MP4 yang bisa diputar.
5. Ulangi beberapa kali, lalu tambah channel bertahap sambil mengukur beban.

Sebelum lima langkah itu lolos, status yang jujur adalah **“backend tersedia dan sudah diuji sintetis; integrasi perangkat masih perlu dibuktikan.”** Menambah fitur atau merapikan folder lagi tidak akan menjawab ketidakpastian terbesarnya—sekarang yang dibutuhkan adalah pengujian NVR, frontend, dan mini PC bersama-sama.