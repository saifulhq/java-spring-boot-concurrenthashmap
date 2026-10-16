# Dokumentasi Arsitektur Sistem: ConcurrentHashMap Session Management

Dokumen ini menjelaskan arsitektur internal, model konkurensi, manajemen memori, dan siklus hidup bean Spring Boot.

---

## 1. Arsitektur Komponen

```mermaid
graph TD
    subgraph Client Layer
        Browser[Browser / Swagger UI]
        MobileApp[Mobile / Web Client]
    end

    subgraph Spring Boot 3+ Web MVC Layer
        Dispatcher[DispatcherServlet]
        Interceptor[SessionAuthInterceptor]
        Dispatcher --> Interceptor
    end

    subgraph Per-Request Scope
        ScopeContext["@RequestScope UserSessionContext\n(Unique per HTTP Request via CGLIB Proxy)"]
    end

    subgraph Application Scope (Singleton)
        SessionRepo["InMemorySessionRepository\n(Singleton Repository)"]
        Store[("ConcurrentHashMap<String, UserSession>\n- Key: SHA-256 Hash\n- Value: UserSession + TTL")]
        SessionRepo --> Store
        
        Scheduler["SessionCleanupJob\n(@Scheduled fixedRate = 180000 ms)"]
        Scheduler -->|removeExpiredSessions| SessionRepo
    end

    subgraph Controllers
        SessionCtrl[SessionController]
        CustomerCtrl[CustomerController]
    end

    Client Layer --> Dispatcher
    Interceptor -->|Validate & Query| SessionRepo
    Interceptor -->|Populate Context| ScopeContext
    Interceptor --> Controllers
    CustomerCtrl -->|Read Current Session| ScopeContext
    SessionCtrl -->|Save & Query| SessionRepo
```

---

## 2. Analisis Keamanan Thread & Konkurensi (`ConcurrentHashMap`)

1. **Struktur Penyimpanan**:
   Menggunakan `ConcurrentMap<String, UserSession> sessionStore = new ConcurrentHashMap<>()`.
2. **Tidak Mengunci Seluruh Map (*Bucket-level striping / CAS*)**:
   `ConcurrentHashMap` pada Java 21 menggunakan *Compare-And-Swap* (CAS) dan *synchronized* hanya pada node bucket pertama saat terjadi *hash collision*. Hal ini memungkinkan ribuan request registrasi dan pembacaan hash berjalan paralel secara simultan tanpa *thread starvation*.
3. **Thread-Safe Iterasi & Pembersihan Berkala dengan Log Detail Metrik Hash**:
   Method `sessionStore.entrySet().removeIf(entry -> entry.getValue().isExpired(now))` menggunakan iterator yang bersifat *weakly consistent*. Pembersihan berkala setiap 3 menit tidak akan melempar `ConcurrentModificationException` dan tidak memblokir penambahan session baru.
   - **Observabilitas Lengkap**: Setiap kali proses cleanup berjalan, sistem mencatat log komprehensif berisi:
     - Jumlah hash awal sebelum proses clear dimulai
     - Jumlah hash/data yang terhapus karena expired (jumlah terclear)
     - Jumlah hash yang tersisa setelah proses clear selesai
     ```log
     [Session Cleanup] Memulai proses pembersihan session expired. Jumlah hash awal: 50
     [Session Cleanup] Pembersihan selesai. Jumlah hash awal: 50, Jumlah data terhapus karena expired (terclear): 12, Jumlah hash setelah clear: 38
     ```

---

## 3. Mekanisme Routing Interceptor & Siklus Hidup `@RequestScope`

### 3.1 Apakah Interceptor Berjalan untuk Semua Request?
**Tidak.** Di Spring Web MVC, `SessionAuthInterceptor` **hanya mencegat request yang pola URL-nya cocok** dengan konfigurasi di `WebMvcConfig`:
- **URL yang Diintercept (`addPathPatterns`)**:
  - `/customer/**` (Endpoint bisnis: `/customer/profile`, `/customer/review`)
  - `/session/me` (Pemeriksaan session aktif)
- **URL yang Dilewati Sepenuhnya / Whitelist (`excludePathPatterns`)**:
  - `/session/register` (Registrasi session)
  - `/session/*` (GET `/session/{hash}`)
  - `/swagger-ui/**`, `/swagger-ui.html`, `/v3/api-docs/**` (Swagger UI)
  - `/error` (Error handler internal Spring)

Ketika request ke `/session/register` atau `/swagger-ui.html` masuk, `DispatcherServlet` mendeteksi bahwa URL tersebut berada di daftar *exclude*, sehingga `SessionAuthInterceptor.preHandle()` **sama sekali tidak dieksekusi (bypassed)**.

- **Proxy Mode**: Menggunakan `ScopedProxyMode.TARGET_CLASS` bawaan Spring `@RequestScope`.
- **Mekanisme**: Saat `CustomerController` (yang merupakan Singleton) bergantung pada `UserSessionContext`, Spring menginjeksi sebuah CGLIB proxy.
- **Isolasi Thread**: Setiap kali method pada `UserSessionContext` dipanggil, proxy secara transparan mendelegasikannya ke instance `UserSessionContext` milik thread request HTTP yang sedang aktif di `RequestContextHolder`.
- **Garbage Collection**: Begitu sebuah HTTP request selesai dan koneksi ditutup, instance `UserSessionContext` untuk request tersebut otomatis siap dibersihkan oleh JVM Garbage Collector.

---

## 4. Mekanisme Pembuatan Key SHA-2 (SHA-256) Unik

Untuk mencegah terjadinya tabrakan (*collision*) atau penimpaan key session jika user melakukan registrasi ulang dengan data payload yang sama, sistem menambahkan *salt* dinamis berbasis timestamp:

### Formula Input String Mentah:
```
{userId}:{leadId}:{source};{salt};{timestamp-create};{timestamp-expired}
```

### Parameter:
- `{userId}`, `{leadId}`, `{source}`: Nilai string dari request body registrasi.
- `{salt}`: Identifier salt penanda internal session (dapat dikonfigurasi di `application.properties` via `session.salt=mysession`, default: `mysession`).
- `{timestamp-create}`: Nilai epoch millis saat request registrasi diproses (`Instant.now().toEpochMilli()`).
- `{timestamp-expired}`: Nilai epoch millis saat session kedaluwarsa (`expiresAt.toEpochMilli()`).

### Contoh Konkret:
```text
USR-001:LEAD-99:MOBILE_APP;mysession;1773643627123;1773647227123
```

### Konfigurasi di `application.properties`:
```properties
# Identifier Salt Session Key (Dapat di-override via Azure Key Vault / App Settings: SESSION_SALT)
session.salt=mysession
```

### Proses Hashing (Java 21 `MessageDigest`):
1. String di atas di-encode ke bytes menggunakan UTF-8: `rawData.getBytes(StandardCharsets.UTF_8)`.
2. Dihitung digest SHA-256: `MessageDigest.getInstance("SHA-256").digest(...)`.
3. Dikonversi ke hex string 64-karakter dengan `HexFormat.of().formatHex(digest)`.

Hasilnya adalah string hex 64 karakter yang unik di setiap milidetik, sehingga registrasi berulang dengan data sama tetap menghasilkan session key yang berbeda dan tidak saling menimpa.
