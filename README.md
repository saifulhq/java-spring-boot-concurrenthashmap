# ConcurrentHashMap Session Management Service

Backend service performa tinggi berbasis **Java 21**, **Spring Boot 3+**, **Maven**, dan **Project Lombok** untuk mengelola session di memori (*in-memory*) menggunakan **`ConcurrentHashMap`** dengan isolasi state **`@RequestScope`**.

---

## 📚 Indeks Dokumentasi Teknis

Untuk memastikan siapa pun yang melanjutkan atau memelihara proyek ini memahami seluruh aspek teknis, dokumentasi lengkap telah disusun secara modular di folder [`docs/`](file:///Users/saifulhq/Documents/belajar/antygravity/java/ConcurrentHashMap/docs/):

1. **[Dokumentasi Arsitektur Sistem (`docs/ARCHITECTURE.md`)](file:///Users/saifulhq/Documents/belajar/antygravity/java/ConcurrentHashMap/docs/ARCHITECTURE.md)**
   - Desain komponen, model konkurensi `ConcurrentHashMap`, analisis thread-safety, dan siklus hidup bean Spring `@RequestScope`.
2. **[Spesifikasi API Lengkap (`docs/API_SPEC.md`)](file:///Users/saifulhq/Documents/belajar/antygravity/java/ConcurrentHashMap/docs/API_SPEC.md)**
   - Kontrak endpoint REST, payload request/response, format header `x-session`, dan format error HTTP 400 Bad Request.
3. **[Rencana & Skenario Pengujian (`docs/TEST_PLAN.md`)](file:///Users/saifulhq/Documents/belajar/antygravity/java/ConcurrentHashMap/docs/TEST_PLAN.md)**
   - Detail pengujian otomatis (5 session & 10 concurrent requests), pengujian lazy TTL expiration, dan background scheduler 3 menit.
4. **[Architecture Decision Record (`docs/adr/0001-in-memory-session-and-request-scope.md`)](file:///Users/saifulhq/Documents/belajar/antygravity/java/ConcurrentHashMap/docs/adr/0001-in-memory-session-and-request-scope.md)**
   - Keputusan arsitektur (ADR) di balik pemilihan `ConcurrentHashMap`, SHA-256, HTTP 400 vs 401, dan konfigurasi Azure Key Vault.
5. **[Rencana Arsitektur Menyeluruh (`ARCHITECTURE_PLAN.md`)](file:///Users/saifulhq/Documents/belajar/antygravity/java/ConcurrentHashMap/ARCHITECTURE_PLAN.md)**
   - Master blueprint sistem secara keseluruhan.

---

## 🚀 Fitur Unggulan

- **In-Memory Thread-Safe**: Menggunakan `ConcurrentHashMap` dengan operasi atomik yang mampu menampung ribuan request paralel tanpa *locking* global.
- **SHA-256 Session Key**: Menghasilkan session hash deterministik dari payload `{ clientId, agentId, pov, source }`.
- **TTL 1 Jam & Job Pembersihan 3 Menit**: Session otomatis kedaluwarsa setelah 1 jam dan dibersihkan setiap 180.000 ms via Spring `@Scheduled`.
- **Validasi Header 400 Bad Request**: Menolak request tanpa header `x-session` atau hash expired dengan HTTP 400 (kode 401 dicadangkan untuk JWT).
- **Isolasi `@RequestScope`**: Context request diisolasi per thread request HTTP secara independen.
- **Swagger / OpenAPI 3**: Dokumentasi interaktif siap pakai di `/swagger-ui.html` tanpa memerlukan header `x-session`.
- **Azure Cloud Ready**: Menggunakan `application.properties` yang kompatibel dengan injeksi Azure Key Vault (AKV).

---

## 🛠️ Cara Menjalankan Aplikasi

### Prerequisites
- **Java 21**
- **Maven** (`mvn` atau `./mvnw`)

### Menjalankan Service
```bash
mvn spring-boot:run
```

### Membuka Swagger UI
Akses di browser:
```
http://localhost:8080/swagger-ui.html
```

---

## 🧪 Menjalankan Automated Tests
```bash
mvn test
```
Termasuk pengujian **5 session dan 10 request konkuren** dengan Java 21 Virtual Threads untuk memvalidasi isolasi `@RequestScope`.
