Gangguan	Perilaku yang ditargetkan
Internet/cloud mati	Penilaian berjalan normal
Media gagal	Penilaian lanjut, tampilkan peringatan
WiFi perangkat putus	Draft disimpan di browser; belum disebut tersimpan di server
Server restart	Nilai pulih; ronde terinterupsi menunggu operator, sesuai keputusanmu
SSD rusak	Pemulihan dari backup terakhir, sesuai toleransi yang kamu pilih

Prinsip offline-first yang harus menjadi syarat implementasi:
- Operasi lomba selalu memakai server lokal, termasuk saat internet tersedia.
- Login, pembacaan data event, penyimpanan nilai, dan rekap tidak menunggu cloud.
- Nilai disimpan secara transaksional bersama antrean sinkronisasinya.
- Sinkronisasi berjalan di background; kegagalannya tidak menggagalkan penyimpanan lokal.
- Gangguan video, restream, upload, atau cloud tidak mematikan layanan penilaian.
- Frontend, aset, autentikasi, dan resolusi domain tersedia lokal.