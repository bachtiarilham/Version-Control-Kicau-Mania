basis template membuat aplikasi ini adalah [server-boilerplate.md](../reference/server-boilerplate.md)
namun, tentunya ada penyesuaian diantaranya : 

domain yang diperlukan : 
  1. asset
  2. competition
  3. iam
  4. judging
  5. media
  6. member

database yang digunakan : 
  1. connection name : PostGres
  2. database name : kicau_mania
  3. port : 5433
  4. username : postgres
  5. password : root

Middleware : 
  1. Urutan Pipeline (Execution Order)
  Recovery → Request ID / Tracing → Logging & Metrics → CORS & Security Headers → Rate Limiting → Authentication → Authorization (RBAC) → Handler/Controller.

  2. Prinsip Kerja Utama
      1. Fail-Fast (Short-Circuit): Hentikan request seawal mungkin jika tidak valid (misal: token expired, rate limit tercapai, body melebihi batas). Jangan biarkan request masuk ke database.
      2. Context Propagation: Ekstrak identitas (user_id, role, tenant_id) di middleware dan simpan ke Request Context. Handler tinggal mengambil data tanpa parse ulang token.
      3. Isolasi Concern: Middleware hanya untuk cross-cutting concerns (keamanan, observabilitas, auth), bukan untuk kalkulasi business logic domain.
      4. Non-Blocking Audit: Jalankan log aktivitas audit secara asinkron (background worker/queue) agar tidak menambah latensi response time.
      5. Factory Pattern: Gunakan closure/parameter dinamis agar satu middleware dapat dipakai ulang dengan konfigurasi berbeda (misal: batas rate limit berbeda per route).

  3. Matriks Pembagian Grup Route
      1.  Level Route : public 
          Middleware yang Terpasang  : Recovery, Request ID, Logging, CORS, Rate Limiter Dasar
      2.  Level Route : Protected (User)
          Middleware yang Terpasang  : Global Stack + Auth Validator (JWT/Session) + Subscription Quota Check
      3.  Level Route : Restricted (Admin)
          Middleware yang Terpasang  : Protected Stack + RBAC Guard + Strict Rate Limiter + Async Audit Logger

Repository : 
  1. aku ingin mengubah cara kerja repository, soalnya saya ingin memanfaatkan sql views sehingga repository di server bersih. 
  2. aku ingin menggunakan materialized_views juga untuk pengambilan data yang berulang ulang serta membutuhkan filter.

