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
3. **Thread-Safe Iterasi & Pembersihan**:
   Method `sessionStore.entrySet().removeIf(entry -> entry.getValue().isExpired(now))` menggunakan iterator yang bersifat *weakly consistent*. Pembersihan berkala setiap 3 menit tidak akan melempar `ConcurrentModificationException` dan tidak memblokir penambahan session baru.

---

## 3. Siklus Hidup `@RequestScope` Bean (`UserSessionContext`)

- **Proxy Mode**: Menggunakan `ScopedProxyMode.TARGET_CLASS` bawaan Spring `@RequestScope`.
- **Mekanisme**: Saat `CustomerController` (yang merupakan Singleton) bergantung pada `UserSessionContext`, Spring menginjeksi sebuah CGLIB proxy.
- **Isolasi Thread**: Setiap kali method pada `UserSessionContext` dipanggil, proxy secara transparan mendelegasikannya ke instance `UserSessionContext` milik thread request HTTP yang sedang aktif di `RequestContextHolder`.
- **Garbage Collection**: Begitu sebuah HTTP request selesai dan koneksi ditutup, instance `UserSessionContext` untuk request tersebut otomatis siap dibersihkan oleh JVM Garbage Collector.
