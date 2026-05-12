# CariMakan — API Reference

## Spesifikasi Umum

- **Base URL:** `https://api.carimakan.app/v1`
- **Protocol:** HTTPS only
- **Format:** JSON (`Content-Type: application/json`)
- **Auth:** Bearer JWT Token di header `Authorization`
- **Style:** RESTful API (GraphQL sebagai alternatif)

### Contoh Request Header
```http
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

---

## Autentikasi (`/auth`)

### POST `/auth/register`
Daftar akun baru (Customer atau Owner Resto).

**Request Body:**
```json
{
  "name": "Budi Santoso",
  "email": "budi@email.com",
  "phone": "081234567890",
  "password": "password123",
  "role": "customer"
}
```

**Response `201`:**
```json
{
  "uid": "user_abc123",
  "token": "<jwt_token>",
  "role": "customer"
}
```

---

### POST `/auth/login`
Login dan dapatkan JWT token.

**Request Body:**
```json
{
  "email": "budi@email.com",
  "password": "password123"
}
```

**Response `200`:**
```json
{
  "token": "<jwt_token>",
  "uid": "user_abc123",
  "role": "customer"
}
```

---

## Restoran (`/restaurants`)

### GET `/restaurants/nearby`
Ambil daftar restoran terdekat berdasarkan GPS.

**Query Params:**
| Param | Tipe | Keterangan |
|---|---|---|
| `lat` | float | Latitude posisi user |
| `lng` | float | Longitude posisi user |
| `radius` | number | Radius pencarian (meter, default: 2000) |

**Response `200`:**
```json
[
  {
    "resto_id": "resto_xyz",
    "name": "Warung Pak Budi",
    "distance_m": 350,
    "rating_avg": 4.5,
    "is_queue_open": true,
    "queue_count": 3,
    "badges": ["WiFi", "AC"],
    "photo_url": "https://storage.firebase.../foto.jpg"
  }
]
```

---

### GET `/restaurants/:resto_id`
Detail informasi restoran.

### POST `/restaurants` *(Owner only)*
Daftarkan restoran baru (status awal: `pending`, menunggu approval Admin).

### PUT `/restaurants/:resto_id` *(Owner only)*
Update profil restoran (nama, jam, badges, GPS, foto).

---

## Menu (`/restaurants/:resto_id/menus`)

### GET `/restaurants/:resto_id/menus`
Ambil semua menu restoran (diakses customer via QR Code).

**Response `200`:**
```json
[
  {
    "menu_id": "menu_001",
    "name": "Nasi Goreng Spesial",
    "price": 25000,
    "is_available": true,
    "photo_url": "...",
    "category": "makanan"
  }
]
```

### POST `/restaurants/:resto_id/menus` *(Owner only)*
Tambah menu baru.

### PUT `/restaurants/:resto_id/menus/:menu_id` *(Owner only)*
Edit menu (harga, ketersediaan, foto).

### DELETE `/restaurants/:resto_id/menus/:menu_id` *(Owner only)*
Hapus menu.

---

## Antrian (`/queues`)

### GET `/queues/:resto_id/status`
Cek status antrian restoran secara real-time.

**Response `200`:**
```json
{
  "is_queue_open": true,
  "queue_count": 5,
  "estimated_wait_minutes": 15
}
```

### POST `/queues/:resto_id/take`
Customer ambil nomor antrian.

**Request Body:**
```json
{
  "type": "dine_in"
}
```

**Response `201`:**
```json
{
  "queue_id": "q_abc",
  "queue_number": 6,
  "type": "dine_in"
}
```

### PUT `/queues/:resto_id/call` *(Owner only)*
Owner panggil antrian berikutnya.

### PUT `/queues/:resto_id/toggle` *(Owner only)*
Buka / tutup antrian.

---

## Pesanan (`/orders`)

### POST `/orders`
Buat pesanan baru. Pembayaran bersifat **pre-paid** — pesanan hanya masuk antrian setelah pembayaran berhasil.

**Request Body:**
```json
{
  "resto_id": "resto_xyz",
  "queue_id": "q_abc",
  "type": "take_away",
  "items": [
    { "menu_id": "menu_001", "qty": 2 },
    { "menu_id": "menu_003", "qty": 1 }
  ],
  "pickup_time": "2026-03-14T12:30:00Z",
  "promo_code": "DISKON10"
}
```

**Response `201`:**
```json
{
  "order_id": "order_xyz",
  "total_price": 62500,
  "status": "pending",
  "payment_url": "https://midtrans.com/pay/..."
}
```

### GET `/orders/:order_id`
Cek detail dan status pesanan.

### PUT `/orders/:order_id/status` *(Owner only)*
Update status pesanan.

**Request Body:**
```json
{
  "status": "ready"
}
```
> Memicu push notification otomatis ke customer.

### GET `/orders/history` *(Customer)*
Riwayat semua transaksi customer.

---

## Pembayaran (`/payments`)

### POST `/payments/webhook`
Endpoint untuk menerima callback dari Midtrans (server-to-server).

> Endpoint ini dipanggil oleh Midtrans, bukan oleh client langsung.

### GET `/payments/:order_id`
Cek status pembayaran pesanan tertentu.

---

## Review (`/reviews`)

### POST `/reviews`
Customer berikan rating dan ulasan setelah pesanan selesai.

**Request Body:**
```json
{
  "order_id": "order_xyz",
  "rating": 5,
  "comment": "Makanannya enak banget!"
}
```

### GET `/reviews/:resto_id`
Ambil semua review publik restoran tertentu.

---

## Reward & Poin (`/rewards`)

### GET `/rewards/me`
Cek total poin dan histori reward customer yang sedang login.

### POST `/rewards/redeem`
Tukar poin dengan voucher diskon.

**Request Body:**
```json
{
  "poin": 100
}
```

---

## Admin (`/admin`)

> Semua endpoint `/admin` hanya bisa diakses dengan role `admin`.

### GET `/admin/restaurants/pending`
Daftar restoran yang menunggu verifikasi.

### PUT `/admin/restaurants/:resto_id/verify`
Approve atau reject pendaftaran restoran.

**Request Body:**
```json
{
  "action": "approve"
}
```

### GET `/admin/users`
Daftar semua pengguna platform.

### PUT `/admin/users/:uid/suspend`
Suspend atau aktifkan kembali akun pengguna.

### GET `/admin/stats`
Statistik platform (total transaksi, user aktif, resto terlaris).

### POST `/admin/promos`
Buat promo/voucher baru.

---

## QR Code Meja (`/table-qr`)

### POST `/table-qr/generate` *(Owner only)*
Generate QR Code statis untuk nomor meja tertentu.

**Request Body:**
```json
{
  "resto_id": "resto_xyz",
  "table_number": "A1"
}
```

**Response `201`:**
```json
{
  "qr_id": "qr_001",
  "table_number": "A1",
  "qr_url": "https://storage.firebase.../qr_A1.png"
}
```

---

## HTTP Status Code

| Code | Arti |
|---|---|
| `200` | OK — request berhasil |
| `201` | Created — data berhasil dibuat |
| `400` | Bad Request — input tidak valid |
| `401` | Unauthorized — token tidak ada / expired |
| `403` | Forbidden — tidak punya akses |
| `404` | Not Found — data tidak ditemukan |
| `409` | Conflict — data sudah ada (misal: email duplikat) |
| `500` | Internal Server Error |