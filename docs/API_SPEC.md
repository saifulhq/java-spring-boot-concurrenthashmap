# Spesifikasi API: ConcurrentHashMap Session Service

Dokumen ini mendefinisikan seluruh kontrak REST API, header, skema request/response, dan kode error.

---

## Ringkasan Endpoint

| Method | Endpoint | Wajib Header `x-session` | Deskripsi |
|---|---|---|---|
| `POST` | `/session/register` | **Tidak** | Mendaftarkan session baru & menghasilkan SHA-256 hash |
| `GET` | `/session/{hash}` | **Tidak** | Inspeksi data session berdasarkan hash key |
| `GET` | `/session/me` | **Ya** | Melihat data session saat ini |
| `GET` | `/customer/profile` | **Ya** | Mengambil profil pelanggan dari session context |
| `GET` | `/customer/review` | **Ya** | Mengambil review pelanggan dari session context |
| `GET` | `/swagger-ui.html` | **Tidak** | Tampilan Swagger UI interaktif |
| `GET` | `/v3/api-docs` | **Tidak** | Skema OpenAPI JSON |

---

## 1. POST `/session/register`
Mendaftarkan session baru ke dalam `ConcurrentHashMap`.

### Request Header
```
Content-Type: application/json
```

### Request Body
```json
{
  "clientId": "CL-001",
  "agentId": "AG-99",
  "pov": "CUSTOMER",
  "source": "MOBILE_APP"
}
```
*Validasi: Seluruh field wajib diisi (tidak boleh blank/null).*

### Response (201 Created)
```json
{
  "sessionHash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "message": "Session registered successfully",
  "createdAt": "2026-10-16T13:30:00Z",
  "expiresAt": "2026-10-16T14:30:00Z"
}
```

---

## 2. GET `/session/{hash}`
Mengecek status dan melihat data session langsung melalui path parameter.

### Path Variable
- `hash` (string, wajib): SHA-256 session hash

### Response (200 OK)
```json
{
  "sessionHash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "active": true,
  "payload": {
    "clientId": "CL-001",
    "agentId": "AG-99",
    "pov": "CUSTOMER",
    "source": "MOBILE_APP"
  },
  "createdAt": "2026-10-16T13:30:00Z",
  "expiresAt": "2026-10-16T14:30:00Z",
  "remainingMinutes": 59
}
```

### Response (400 Bad Request jika tidak ditemukan atau expired)
```json
{
  "status": 400,
  "error": "BAD_REQUEST",
  "message": "Session hash not found or already expired",
  "timestamp": "2026-10-16T13:35:00Z"
}
```

---

## 3. GET `/customer/profile`
Mengambil profil pelanggan menggunakan konteks `@RequestScope`.

### Request Header
```
x-session: <sha256-hash>
```

### Response (200 OK)
```json
{
  "clientId": "CL-001",
  "agentId": "AG-99",
  "pov": "CUSTOMER",
  "source": "MOBILE_APP",
  "accountTier": "PREMIUM",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "scopeInstanceHashCode": 18273645,
  "threadName": "http-nio-8080-exec-1"
}
```

---

## 4. GET `/customer/review`
Mengambil ulasan pelanggan menggunakan konteks `@RequestScope`.

### Request Header
```
x-session: <sha256-hash>
```

### Response (200 OK)
```json
{
  "clientId": "CL-001",
  "totalReviews": 12,
  "averageRating": 4.8,
  "source": "MOBILE_APP",
  "requestId": "550e8400-e29b-41d4-a716-446655440001",
  "scopeInstanceHashCode": 98765432,
  "threadName": "http-nio-8080-exec-2"
}
```

---

## 5. Format Error Standar (400 Bad Request)
Jika header `x-session` tidak dikirim atau hash sudah kadaluwarsa pada endpoint terproteksi:

```json
{
  "status": 400,
  "error": "BAD_REQUEST",
  "message": "Header 'x-session' is missing, invalid, or expired",
  "timestamp": "2026-10-16T13:35:00Z"
}
```
*(Catatan: Kode 401 dicadangkan khusus untuk validasi token JWT).*
