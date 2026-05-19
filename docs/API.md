# CariMakan — API Reference

## Spesifikasi Umum

- **Base URL:** `http://localhost:3000/v1` *(development)* | `https://api.carimakan.app/v1` *(production)*
- **Protocol:** HTTPS only
- **Format:** JSON (`Content-Type: application/json`)
- **Auth:** Bearer JWT Token di header `Authorization`
- **Style:** RESTful API

### Contoh Request Header
```http
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

### Health Check
```
GET /health
```
Response `200`: `{ "status": "UP", "timestamp": "..." }`

---

## Autentikasi (`/v1/auth`)

### POST `/v1/auth/register`
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
> `role` valid: `customer`, `owner`, `admin` (default: `customer`)

**Response `201`:**
```json
{
  "uid": "user_abc123",
  "token": "<jwt_token>",
  "role": "customer"
}
```

---

### POST `/v1/auth/login`
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

## Restoran (`/v1/restaurants`)

### GET `/v1/restaurants`
Ambil daftar semua restoran.

**Query Params:**
| Param | Tipe | Keterangan |
|---|---|---|
| `status` | string | Filter status: `aktif`, `pending`, `suspend`, `all` (default: `aktif`) |
| `limit` | number | Batasi jumlah hasil |

**Response `200`:**
```json
[
  {
    "resto_id": "resto_xyz",
    "name": "Warung Pak Budi",
    "rating_avg": 4.5,
    "total_review": 120,
    "is_queue_open": true,
    "queue_count": 3,
    "badges": ["WiFi", "AC"],
    "photo_url": "https://...",
    "status": "aktif"
  }
]
```

---

### GET `/v1/restaurants/nearby`
Ambil daftar restoran terdekat berdasarkan GPS.

**Query Params:**
| Param | Tipe | Keterangan |
|---|---|---|
| `lat` | float | **Required.** Latitude posisi user |
| `lng` | float | **Required.** Longitude posisi user |
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
    "photo_url": "https://..."
  }
]
```

---

### GET `/v1/restaurants/:resto_id`
Detail informasi restoran.

**Response `200`:**
```json
{
  "resto_id": "resto_xyz",
  "name": "Warung Pak Budi",
  "owner_id": "user_abc",
  "lokasi": { "lat": -6.2, "lng": 106.8 },
  "jam_buka": "08:00-22:00",
  "status": "aktif",
  "url_whatsapp": "https://wa.me/628...",
  "rating_avg": 4.5,
  "total_review": 120,
  "is_queue_open": true,
  "queue_count": 3,
  "badges": [{ "id": "b1", "nama": "WiFi", "icon": "wifi" }],
  "photo_url": "https://...",
  "created_at": "2026-01-01T00:00:00.000Z"
}
```

---

### POST `/v1/restaurants` *(Owner only)*
Daftarkan restoran baru. Status awal: `pending`, menunggu approval Admin.

**Request Body:**
```json
{
  "name": "Warung Pak Budi",
  "lat": -6.2,
  "lng": 106.8,
  "jam_buka": "08:00-22:00",
  "url_whatsapp": "https://wa.me/628...",
  "foto_uri": "https://...",
  "badges": ["badge_id_1", "badge_id_2"]
}
```

**Response `201`:** Data restoran yang baru dibuat.

---

### PUT `/v1/restaurants/:resto_id` *(Owner only)*
Update profil restoran.

**Request Body:** *(semua field opsional)*
```json
{
  "name": "Nama Baru",
  "lat": -6.2,
  "lng": 106.8,
  "jam_buka": "09:00-21:00",
  "url_whatsapp": "https://wa.me/628...",
  "foto_uri": "https://...",
  "is_queue_open": false,
  "badges": ["badge_id_1"]
}
```

**Response `200`:** `{ "success": true, "message": "Restaurant updated successfully" }`

---

## Menu (`/v1/restaurants/:resto_id/menus`)

### GET `/v1/restaurants/:resto_id/menus`
Ambil semua menu restoran.

**Response `200`:**
```json
[
  {
    "menu_id": "menu_001",
    "name": "Nasi Goreng Spesial",
    "price": 25000,
    "description": "Nasi goreng dengan topping spesial",
    "is_available": true,
    "photo_url": "https://...",
    "category": "makanan"
  }
]
```
> `category` valid: `makanan`, `minuman`

---

### POST `/v1/restaurants/:resto_id/menus` *(Owner only)*
Tambah menu baru.

**Request Body:**
```json
{
  "name": "Nasi Goreng Spesial",
  "price": 25000,
  "description": "Nasi goreng dengan topping spesial",
  "is_available": true,
  "photo_url": "https://...",
  "category": "makanan"
}
```

**Response `201`:** Data menu yang baru dibuat beserta `menu_id`.

---

### PUT `/v1/restaurants/:resto_id/menus/:menu_id` *(Owner only)*
Edit menu (harga, ketersediaan, foto, dll).

**Request Body:** *(semua field opsional)*
```json
{
  "name": "Nasi Goreng Super",
  "price": 28000,
  "description": "...",
  "is_available": false,
  "photo_url": "https://...",
  "category": "makanan"
}
```

**Response `200`:** `{ "success": true, "message": "Menu updated successfully" }`

---

### DELETE `/v1/restaurants/:resto_id/menus/:menu_id` *(Owner only)*
Hapus menu.

**Response `200`:** `{ "success": true, "message": "Menu deleted successfully" }`

---

## Antrian (`/v1/queues`)

### GET `/v1/queues/:resto_id/status`
Cek status antrian restoran secara real-time.

**Response `200`:**
```json
{
  "is_queue_open": true,
  "queue_count": 5,
  "estimated_wait_minutes": 25
}
```
> Estimasi: 5 menit per antrian.

---

### POST `/v1/queues/:resto_id/take` *(Auth required)*
Customer ambil nomor antrian.

**Request Body:**
```json
{
  "type": "dine_in"
}
```
> `type` valid: `dine_in`, `take_away`

**Response `201`:**
```json
{
  "queue_id": "q_abc",
  "queue_number": 6,
  "type": "dine_in"
}
```

---

### PUT `/v1/queues/:resto_id/call` *(Owner only)*
Owner panggil antrian berikutnya (FIFO).

**Response `200`:**
```json
{
  "message": "Queue called successfully",
  "queue_id": "q_abc",
  "queue_number": 6,
  "type": "dine_in"
}
```

---

### PUT `/v1/queues/:resto_id/toggle` *(Owner only)*
Toggle buka / tutup antrian.

**Response `200`:**
```json
{
  "success": true,
  "is_queue_open": false
}
```

---

## Pesanan (`/v1/orders`)

### POST `/v1/orders` *(Auth required)*
Buat pesanan baru. Pembayaran bersifat **pre-paid** via Midtrans.

**Request Body:**
```json
{
  "resto_id": "resto_xyz",
  "queue_id": "q_abc",
  "type": "take_away",
  "items": [
    { "menu_id": "menu_001", "qty": 2, "catatan": "Tidak pedas" },
    { "menu_id": "menu_003", "qty": 1 }
  ],
  "pickup_time": "2026-03-14T12:30:00Z",
  "promo_code": "DISKON10"
}
```
> `type` valid: `dine_in`, `take_away`. `queue_id`, `pickup_time`, `promo_code` bersifat opsional.

**Response `201`:**
```json
{
  "order_id": "order_xyz",
  "total_price": 62500,
  "status": "pending",
  "payment_url": "https://checkout.sandbox.midtrans.com/..."
}
```
> Platform fee: **7%** dari harga setelah diskon.

---

### GET `/v1/orders/history` *(Auth required)*
Riwayat semua transaksi customer yang sedang login.

**Response `200`:**
```json
[
  {
    "order_id": "order_xyz",
    "resto_id": "resto_xyz",
    "resto_name": "Warung Pak Budi",
    "tipe_pesanan": "take_away",
    "status": "completed",
    "total_price": 62500,
    "created_at": "2026-03-14T10:00:00.000Z"
  }
]
```

---

### GET `/v1/orders/:order_id` *(Auth required)*
Detail lengkap pesanan beserta item-item di dalamnya.

**Response `200`:**
```json
{
  "order_id": "order_xyz",
  "resto_id": "resto_xyz",
  "resto_name": "Warung Pak Budi",
  "user_id": "user_abc",
  "queue_id": "q_abc",
  "tipe_pesanan": "take_away",
  "status": "processing",
  "total_price": 62500,
  "app_profit": 4093,
  "created_at": "2026-03-14T10:00:00.000Z",
  "pickup_time": "2026-03-14T12:30:00.000Z",
  "items": [
    {
      "menu_id": "menu_001",
      "name": "Nasi Goreng Spesial",
      "qty": 2,
      "harga_saat_order": 25000,
      "catatan": "Tidak pedas"
    }
  ]
}
```

> `status` valid: `pending` → `processing` → `ready` → `completed` / `cancelled`

---

### PUT `/v1/orders/:order_id/status` *(Owner only)*
Update status pesanan. Menambah reward poin customer otomatis saat status `completed`.

**Request Body:**
```json
{
  "status": "ready"
}
```

**Response `200`:** `{ "success": true, "message": "Order status updated to ready" }`

> Reward: **1 poin per Rp10.000** yang dibayarkan.

---

## Pembayaran (`/v1/payments`)

### POST `/v1/payments/webhook`
Endpoint untuk menerima callback dari **Midtrans** (server-to-server). Tidak dipanggil langsung oleh client.

**Request Body:** Payload notifikasi Midtrans.

---

### GET `/v1/payments/:order_id` *(Auth required)*
Cek status pembayaran pesanan tertentu.

**Response `200`:**
```json
{
  "payment_id": "pay_abc",
  "order_id": "order_xyz",
  "gateway_token": "tok_order_xyz",
  "method": "qris",
  "amount": 62500,
  "status": "success",
  "paid_at": "2026-03-14T10:05:00.000Z"
}
```
> `status` pembayaran: `pending`, `success`, `failed`, `challenge`

---

## Review (`/v1/reviews`)

### POST `/v1/reviews` *(Auth required — Customer)*
Customer berikan rating dan ulasan setelah pesanan **completed**.

**Request Body:**
```json
{
  "order_id": "order_xyz",
  "rating_pelayanan": 5,
  "rating_makanan": 4,
  "rating_fasilitas": 5,
  "comment": "Makanannya enak banget!",
  "tag_ids": ["tag_001", "tag_005", "tag_009"]
}
```
> `tag_ids` bersifat opsional. Ambil daftar tag via `GET /v1/review-tags`.

**Response `201`:**
```json
{
  "review_id": "review_abc",
  "message": "Review submitted successfully",
  "rating_avg": 4.67
}
```

---

### GET `/v1/reviews/:resto_id`
Ambil semua review publik restoran tertentu (diurutkan terbaru).

**Response `200`:**
```json
[
  {
    "review_id": "review_abc",
    "user_name": "Budi Santoso",
    "rating_pelayanan": 5,
    "rating_makanan": 4,
    "rating_fasilitas": 5,
    "rating_avg": 4.67,
    "comment": "Makanannya enak banget!",
    "created_at": "2026-03-14T11:00:00.000Z"
  }
]
```

---

## Tag Ulasan (`/v1/review-tags`)

### GET `/v1/review-tags/categories`
Ambil semua kategori tag ulasan.

**Response `200`:**
```json
[
  { "kategori_id": "kat_001", "nama": "pelayanan", "icon": "icon_service" },
  { "kategori_id": "kat_002", "nama": "makanan", "icon": "icon_food" },
  { "kategori_id": "kat_003", "nama": "fasilitas", "icon": "icon_facility" }
]
```

---

### GET `/v1/review-tags`
Ambil semua tag ulasan, opsional filter per kategori.

**Query Params:**
| Param | Tipe | Keterangan |
|---|---|---|
| `kategori_id` | string | Filter tag berdasarkan kategori (opsional) |

**Response `200`:**
```json
[
  { "tag_id": "tag_001", "kategori_id": "kat_002", "label": "Makanan enak", "icon": "icon_yummy" },
  { "tag_id": "tag_002", "kategori_id": "kat_001", "label": "Pelayanan ramah", "icon": "icon_smile" }
]
```

---

### POST `/v1/review-tags/categories` *(Admin only)*
Tambah kategori tag baru.

**Request Body:**
```json
{ "nama": "suasana", "icon": "icon_ambience" }
```

---

### POST `/v1/review-tags` *(Admin only)*
Tambah tag ulasan baru ke dalam kategori tertentu.

**Request Body:**
```json
{ "kategori_id": "kat_001", "label": "Antrian cepat", "icon": "icon_fast" }
```

---

### PUT `/v1/review-tags/:tag_id` *(Admin only)*
Edit label, icon, atau kategori tag ulasan.

**Request Body:** `{ "label": "...", "icon": "...", "kategori_id": "..." }` *(semua opsional)*

---

### DELETE `/v1/review-tags/:tag_id` *(Admin only)*
Hapus tag ulasan dan semua referensinya di `order_review_tags`.

---

## Reward & Poin (`/v1/rewards`)

### GET `/v1/rewards/me` *(Auth required)*
Cek total poin dan histori reward customer yang sedang login.

**Response `200`:**
```json
{
  "poin": 250,
  "history": [
    { "id": "rp_001", "order_id": "order_xyz", "jumlah_poin": 62, "created_at": "2026-03-14T11:00:00.000Z" },
    { "id": "rp_002", "order_id": null, "jumlah_poin": -100, "created_at": "2026-03-10T09:00:00.000Z" }
  ]
}
```
> Nilai negatif `jumlah_poin` berarti poin digunakan untuk redeem.

---

### POST `/v1/rewards/redeem` *(Auth required)*
Tukar poin dengan voucher diskon. **1 poin = Rp100 diskon**.

**Request Body:**
```json
{ "poin": 100 }
```

**Response `200`:**
```json
{
  "message": "Points redeemed successfully",
  "redeemed_points": 100,
  "voucher": {
    "kode": "CM-XYZABC",
    "nilai_diskon": 10000,
    "berakhir": "2026-06-14T09:00:00.000Z"
  }
}
```

---

## Admin (`/v1/admin`)

> Semua endpoint `/v1/admin` **hanya bisa diakses dengan role `admin`**.

### GET `/v1/admin/restaurants/pending`
Daftar restoran yang menunggu verifikasi.

**Response `200`:**
```json
[
  {
    "resto_id": "resto_new",
    "owner_id": "user_owner",
    "name": "Restoran Baru",
    "lokasi": { "lat": -6.2, "lng": 106.8 },
    "jam_buka": "08:00-22:00",
    "url_whatsapp": "https://wa.me/628...",
    "created_at": "2026-03-14T08:00:00.000Z"
  }
]
```

---

### PUT `/v1/admin/restaurants/:resto_id/verify`
Approve atau reject pendaftaran restoran.

**Request Body:**
```json
{ "action": "approve" }
```
> `action` valid: `approve` (status → `aktif`), `reject` (status → `suspend`)

---

### GET `/v1/admin/users`
Daftar semua pengguna platform.

**Response `200`:**
```json
[
  {
    "uid": "user_abc",
    "nama": "Budi Santoso",
    "email": "budi@email.com",
    "role": "customer",
    "poin_reward": 250,
    "status": "aktif",
    "url_whatsapp": "https://wa.me/628...",
    "created_at": "2026-01-01T00:00:00.000Z"
  }
]
```

---

### PUT `/v1/admin/users/:uid/suspend`
Suspend atau aktifkan kembali akun pengguna.

**Request Body:**
```json
{ "suspend": true }
```
> `suspend: true` → status `suspend`, `suspend: false` → status `aktif`

---

### GET `/v1/admin/stats`
Statistik platform.

**Response `200`:**
```json
{
  "total_completed_orders": 1500,
  "total_users": 320,
  "active_customers": 295,
  "active_restaurants": 45,
  "total_app_profit": 8750000,
  "best_selling_restaurants": [
    { "resto_id": "resto_xyz", "name": "Warung Pak Budi", "total_sales": 3200000 }
  ]
}
```

---

### POST `/v1/admin/promos`
Buat promo/voucher baru (promo global).

**Request Body:**
```json
{
  "kode": "LEBARAN20",
  "nama": "Diskon Lebaran 20%",
  "deskripsi": "Promo spesial Lebaran",
  "nilai_diskon": 20,
  "is_percent": true,
  "mulai": "2026-03-29T00:00:00Z",
  "berakhir": "2026-04-05T23:59:59Z"
}
```
> `is_percent: true` = diskon persen, `false` = diskon nominal (Rupiah).

---

## QR Code Meja (`/v1/table-qr`)

### POST `/v1/table-qr/generate` *(Owner only)*
Generate atau ambil kembali QR Code statis untuk nomor meja tertentu.

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
  "qr_url": "https://api.qrserver.com/v1/create-qr-code/?size=300x300&data=carimakan://resto/resto_xyz/table/A1"
}
```
> QR Code mengarah ke deep link `carimakan://resto/:resto_id/table/:table_number`.

---

## HTTP Status Code

| Code | Arti |
|---|---|
| `200` | OK — request berhasil |
| `201` | Created — data berhasil dibuat |
| `400` | Bad Request — input tidak valid |
| `401` | Unauthorized — token tidak ada / expired |
| `403` | Forbidden — tidak punya akses (role salah) |
| `404` | Not Found — data tidak ditemukan |
| `409` | Conflict — data sudah ada (misal: email duplikat, review sudah ada) |
| `500` | Internal Server Error |

---

## Role & Akses

| Endpoint | Public | Customer | Owner | Admin |
|---|:---:|:---:|:---:|:---:|
| `GET /restaurants` | ✅ | ✅ | ✅ | ✅ |
| `GET /restaurants/nearby` | ✅ | ✅ | ✅ | ✅ |
| `GET /restaurants/:id` | ✅ | ✅ | ✅ | ✅ |
| `POST /restaurants` | ❌ | ❌ | ✅ | ❌ |
| `PUT /restaurants/:id` | ❌ | ❌ | ✅ | ❌ |
| `GET /restaurants/:id/menus` | ✅ | ✅ | ✅ | ✅ |
| `POST/PUT/DELETE .../menus` | ❌ | ❌ | ✅ | ❌ |
| `GET /queues/:id/status` | ✅ | ✅ | ✅ | ✅ |
| `POST /queues/:id/take` | ❌ | ✅ | ❌ | ❌ |
| `PUT /queues/:id/call` | ❌ | ❌ | ✅ | ❌ |
| `PUT /queues/:id/toggle` | ❌ | ❌ | ✅ | ❌ |
| `POST /orders` | ❌ | ✅ | ❌ | ❌ |
| `GET /orders/history` | ❌ | ✅ | ❌ | ❌ |
| `GET /orders/:id` | ❌ | ✅ | ✅ | ✅ |
| `PUT /orders/:id/status` | ❌ | ❌ | ✅ | ❌ |
| `GET /payments/:order_id` | ❌ | ✅ | ✅ | ✅ |
| `POST /payments/webhook` | ✅ | ✅ | ✅ | ✅ |
| `POST /reviews` | ❌ | ✅ | ❌ | ❌ |
| `GET /reviews/:resto_id` | ✅ | ✅ | ✅ | ✅ |
| `GET /review-tags` | ✅ | ✅ | ✅ | ✅ |
| `POST/PUT/DELETE /review-tags` | ❌ | ❌ | ❌ | ✅ |
| `GET /rewards/me` | ❌ | ✅ | ❌ | ❌ |
| `POST /rewards/redeem` | ❌ | ✅ | ❌ | ❌ |
| `GET /admin/*` | ❌ | ❌ | ❌ | ✅ |
| `PUT /admin/*` | ❌ | ❌ | ❌ | ✅ |
| `POST /admin/*` | ❌ | ❌ | ❌ | ✅ |
| `POST /table-qr/generate` | ❌ | ❌ | ✅ | ❌ |