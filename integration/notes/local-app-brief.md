"Bertindaklah sebagai penguji teknis yang sangat kritis (Grill Me mode). Cari setiap celah, kelemahan, potensi bottleneck, asumsi yang salah, atau edge case dari [konsep/arsitektur] yang saya jelaskan berikut. Jangan bersikap sopan atau basa-basi; tanyakan pertanyaan tajam satu per satu dan uji pemahaman saya."

script ini untuk diimplementasi di folder local-app

local server di arena lomba memegang 3 peran vital:
  1. Edge Transcoder / Restreamer:
    MediaMTX di local server bertindak sebagai media hub. Mengambil 24 RTSP feed dari NVR, lalu:
    Menyajikan stream latensi ultra-rendah (WebRTC / Low-Latency HLS) untuk layar monitor juri dan penonton lokal di arena (tanpa memakan kuota internet).
    Meneruskan (push) stream tertentu ke cloud hanya jika dibutuhkan.
  2. Local Audio Metric Processor (Worker Deteksi Suara):
    Local server membaca feed audio per lapak secara paralel, menghitung metrik (Peak dBFS, RMS, durasi kicau aktif), dan mengirimkan statistik ringkasnya via WebSocket / API ke backend.
  3. Local Segmenter / DVR Buffer (Penyelamat Jaringan):
    Koneksi internet di lapangan lomba burung sering kali tidak stabil. Local server merekam video lomba ke SSD lokal terlebih dahulu (misalnya format MP4 per ronde). Setelah ronde selesai, worker di local server akan meng-upload file rekaman yang sudah jadi ke storage secara asinkron (background job).
  
untuk template server app nya, gunakan [server-boilerplate.md](../reference/server-boilerplate.md)

protocol yang digunakan untuk video : webRTC, sedangkan audio : websocket. 

link mediaMTX : https://github.com/bluenviron/mediamtx

CCTV yang digunakan merk ZiTech. 

aku inginnya local-server ini bisa restream ke youtube. 
aku ingin setting konfigurasinya ini ditampung di satu file, boleh memanfaatkan .env

