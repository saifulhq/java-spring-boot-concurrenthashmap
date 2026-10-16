# ADR 0001: In-Memory Session Management Menggunakan ConcurrentHashMap dan Spring @RequestScope

- **Status**: Disetujui (Accepted)
- **Tanggal**: 2026-10-16
- **Pengambil Keputusan**: Engineering Team & Arsitek

---

## 1. Konteks Masalah
Aplikasi membutuhkan mekanisme pengelolaan session dengan performa tinggi tanpa menggunakan database eksternal (seperti PostgreSQL atau Redis). Sistem harus mampu menampung banyak request registrasi session secara bersamaan, menghasilkan token session berbasis hash SHA-2
dari payload `{ clientId, agentId, pov, source }`, dan menjamin setiap request bisnis berikutnya memiliki isolasi state session yang aman secara thread-safe.

---

## 2. Keputusan Arsitektur

### 2.1 Penggunaan `ConcurrentHashMap` vs Penyimpanan Lain
- **Keputusan**: Menggunakan `ConcurrentHashMap<String, UserSession>` yang dibungkus dalam Spring Singleton Repository (`InMemorySessionRepository`).
- **Alasan**:
  - `ConcurrentHashMap` menyediakan thread-safety tingkat tinggi dengan *bucket-level locking* dan operasi atomik (CAS), sehingga ribuan request paralel dapat membaca dan menulis data tanpa memblokir satu sama lain.
  - Tidak ada overhead jaringan atau ketergantungan database eksternal.
  - Iterasi melalui `entrySet().removeIf(...)` bersifat *weakly consistent*, aman dijalankan bersamaan dengan operasi penambahan session.

### 2.2 Penggunaan SHA-256 sebagai Session Key
- **Keputusan**: Hash SHA-2 dikalkulasikan dari representasi string kanonikal payload registrasi.
- **Alasan**:
  - Menghasilkan string hex 64-karakter dengan entropi seragam, cocok sebagai key hash map dengan distribusi bucket yang merata.

### 2.3 Status Error HTTP 400 Bad Request untuk `x-session-hash`
- **Keputusan**: Jika header `x-session-hash` tidak dikirim, formatnya salah, atau hash tidak ditemukan/expired di memori, kembalikan **HTTP 400 Bad Request**.
- **Alasan**:
  - HTTP 401 Unauthorized secara tegas dicadangkan untuk validasi token otentikasi JWT yang akan diimplementasikan di fase berikutnya. Penggunaan 400 membedakan dengan jelas antara *malformed/missing session context* dengan *unauthorized user identity*.

### 2.4 Isolasi State Menggunakan Spring `@RequestScope`
- **Keputusan**: Data session diinjeksi ke bean `UserSessionContext` beranotasi `@RequestScope`.
- **Alasan**:
  - Menjamin bahwa data session terikat secara eksklusif ke siklus hidup satu HTTP request.
  - Mencegah kebocoran data (*data bleeding*) antar request yang dilayani oleh thread pool Tomcat yang dapat digunakan kembali (*recycled thread*).

### 2.5 Pemilihan `application.properties` untuk Azure Cloud
- **Keputusan**: Menggunakan format `application.properties` (bukan `.yml`).
- **Alasan**:
  - Format flat key-value lebih mudah dibaca dan langsung cocok dengan injeksi Azure Key Vault (AKV) dan Azure App Service App Settings (misal: `SESSION_TTL_DURATION` atau `session--ttl-duration`).

---

## 3. Konsekuensi

### Keuntungan:
- Latensi ultra-rendah (< 1 ms untuk lookup session).
- Sangat sederhana dijalankan tanpa setup database.
- Arsitektur modular, bersih, dan mudah diperluas jika nantinya ingin diganti ke Redis atau database distributed.

### Keterbatasan & Mitigasi:
- *In-memory state* hilang jika aplikasi di-restart. Untuk fase ini sesuai kebutuhan (belum memerlukan database). Jika nanti membutuhkan persistensi lintas restart atau multi-node horizontal scaling, `InMemorySessionRepository` dapat dengan mudah diadaptasi ke Redis tanpa mengubah kode Controller atau `@RequestScope`.
