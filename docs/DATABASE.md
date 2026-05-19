# CariMakan â€” Database

## Database yang Digunakan

**Primary:** Firestore (Firebase â€” NoSQL, real-time)  
**Media Storage:** Firebase Storage (foto menu, foto restoran, QR Code)

> Skema koleksi di bawah ini dibuat berdasarkan **ERD implementasi aktual** per 19 Mei 2026, sesuai dengan seeder `setupdb/firestore-setup.js` dan server actions di `carimakan_admin`.

---

## Alasan Memilih Firestore

- **Real-time listener** â€” cocok untuk fitur antrian dan status pesanan yang harus update otomatis
- **Scalable** â€” mendukung concurrent users tanpa konfigurasi server manual
- **Integrasi** â€” native dengan Firebase ecosystem (FCM, Auth, Storage)
- **Managed** â€” tidak perlu maintain server database sendiri

---

## Ringkasan Entitas (14 Koleksi)

| Koleksi | Keterangan |
|---|---|
| `users` | Semua pengguna (customer, owner, admin) |
| `restaurants` | Data restoran mitra |
| `menus` | Menu per restoran |
| `meja` | QR Code meja per restoran (dine in) |
| `orders` | Pesanan customer â€” termasuk platform fee (`app_profit`) |
| `order_items` | Item detail dari setiap pesanan |
| `payments` | Transaksi pembayaran via Midtrans |
| `badges` | Master data badge fasilitas |
| `resto_badges` | Junction: restoran â†” badge |
| `tag_kategori` | Master kategori tag ulasan |
| `review_tags` | Tag ulasan per kategori |
| `order_review_tags` | Junction: order â†” tag ulasan yang dipilih |
| `promo_vouchers` | Promo & voucher diskon |
| `reward_poin` | Histori poin reward customer |

> **Catatan implementasi:** `order_review_tags` kini di-seed bersama orders (Â±70% order mendapat 1â€“3 review tags acak). Koleksi `order_items`, `payments`, dan `reward_poin` diisi oleh transaksi nyata dari aplikasi, bukan seeder.

---

## Skema Detail per Koleksi

---

### 1. `users`

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK â€” di-set manual saat seeding (mis: `admin_001`) |
| `nama` | string | Nama lengkap |
| `email` | string | Email login |
| `password` | string | Password teks (saat ini plain-text, divalidasi di server action) |
| `role` | enum | `customer` / `owner` / `admin` |
| `foto_url` | string | URL foto profil (Firebase Storage) â€” nullable |
| `poin_reward` | int | Total poin aktif yang dimiliki (hanya relevan untuk `customer`) |
| `fcm_token` | string | Token FCM untuk push notification â€” nullable |
| `status` | enum | `aktif` / `suspend` |
| `url_whatsapp` | string | Nomor WhatsApp â€” nullable |
| `created_at` | timestamp | Waktu registrasi |

> **Keamanan:** Login admin divalidasi di server action (`verifyAdminLogin`) â€” hanya `role: admin` dan `status: aktif` yang bisa akses dashboard admin. Password sebaiknya di-hash (bcrypt) sebelum production.

---

### 2. `restaurants`

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `owner_id` | string | FK â†’ `users.id` |
| `nama` | string | Nama restoran |
| `lokasi` | GeoPoint | Koordinat GPS (lat, lng) â€” Firestore GeoPoint |
| `foto_uri` | string | URL foto utama restoran â€” nullable |
| `jam_buka` | string | Jam operasional (mis: `"08:00-22:00"`) |
| `status` | enum | `pending` / `aktif` / `suspend` |
| `url_whatsapp` | string | Kontak WhatsApp restoran |
| `avg_rating` | float | Rata-rata rating â€” **denormalized cache** |
| `total_review` | int | Jumlah total review â€” **denormalized cache** |
| `created_at` | timestamp | Tanggal daftar |

> `avg_rating` dan `total_review` di-update otomatis setiap ada review baru agar tidak perlu agregasi tiap query.

---

### 3. `menus`

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `resto_id` | string | FK â†’ `restaurants.id` |
| `nama` | string | Nama menu |
| `harga` | int | Harga **asli** dalam Rupiah (sebelum platform fee 7%) |
| `foto_url` | string | Foto menu â€” nullable |
| `deskripsi` | string | Deskripsi menu |
| `tersedia` | bool | Status ketersediaan menu |

> **Harga di frontend:** Harga yang ditampilkan ke customer adalah `harga + (harga Ã— 7%)`. Platform fee 7% masuk sebagai `app_profit` di setiap `orders`.

---

### 4. `meja`

Data meja per restoran untuk keperluan Dine In.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `resto_id` | string | FK â†’ `restaurants.id` |
| `nomor_meja` | int | Nomor meja |
| `qr_code_url` | string | URL QR Code statis meja (cetak 1x, permanen) â€” nullable |

---

### 5. `orders`

Pesanan utama â€” mencakup status, platform fee, dan tipe pemesanan.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `user_id` | string | FK â†’ `users.id` |
| `resto_id` | string | FK â†’ `restaurants.id` |
| `tipe_pesanan` | enum | `dine_in` / `take_away` |
| `status` | enum | `pending` / `processing` / `ready` / `completed` / `cancelled` |
| `total_price` | int | Total bayar customer = harga asli + 7% platform fee |
| `app_profit` | int | **Platform fee 7%** â€” digunakan untuk kalkulasi profit di admin dashboard |
| `created_at` | timestamp | Waktu pesanan dibuat |

> **Platform Fee:** `app_profit = floor(harga_asli Ã— 0.07)`. Field ini menjadi sumber data utama untuk kalkulasi profit di admin dashboard (chart, per-resto, global).  
> Status `completed` digunakan sebagai filter di semua query analitik dashboard.

---

### 6. `order_items`

Detail item di dalam setiap pesanan.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `order_id` | string | FK â†’ `orders.id` |
| `menu_id` | string | FK â†’ `menus.id` |
| `qty` | int | Jumlah yang dipesan |
| `harga_saat_order` | int | Snapshot harga asli saat transaksi |
| `catatan` | string | Catatan khusus (mis: "tanpa sambal") |

> `harga_saat_order` adalah snapshot â€” perubahan harga menu di masa depan tidak merusak histori transaksi.

---

### 7. `payments`

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `order_id` | string | FK â†’ `orders.id` |
| `gateway_token` | string | Token/ID transaksi dari Midtrans |
| `method` | string | `gopay` / `ovo` / `qris` / `bank` / dll. |
| `amount` | int | Nominal pembayaran (Rupiah) â€” sama dengan `orders.total_price` |
| `status` | enum | `pending` / `success` / `failed` |
| `paid_at` | timestamp | Waktu pembayaran berhasil dikonfirmasi |

---

### 8. `badges`

Master data badge/fasilitas restoran.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `nama` | string | Nama badge (mis: "WiFi", "AC", "Area Parkir") |
| `icon` | string | Nama icon (mis: `wifi`, `ac`, `parking`) |

**Data seed saat ini:** `wifi`, `ac`, `parking`, `toilet`, `child_friendly`, `no_smoking`

---

### 9. `resto_badges`

Junction table antara restoran dan badge yang dimilikinya.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK (mis: `rb_001`) |
| `resto_id` | string | FK â†’ `restaurants.id` |
| `badge_id` | string | FK â†’ `badges.id` |

---

### 10. `tag_kategori`

Master kategori untuk mengelompokkan tag ulasan.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `nama` | string | Nama kategori (`pelayanan`, `makanan`, `fasilitas`, `suasana`) |
| `icon` | string | Icon kategori |

---

### 11. `review_tags`

Tag label spesifik untuk ulasan, dikelompokkan berdasarkan `tag_kategori`.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `kategori_id` | string | FK â†’ `tag_kategori.id` |
| `label` | string | Label tag (mis: "WiFi cepat", "Makanan enak") |
| `icon` | string | Icon tag |

**Tag yang tersedia (11 tag):**

| Tag | Kategori |
|---|---|
| Pelayanan ramah | Pelayanan |
| Antrian cepat | Pelayanan |
| Pesanan tepat waktu | Pelayanan |
| Makanan enak | Makanan |
| Porsi besar | Makanan |
| Harga terjangkau | Makanan |
| WiFi cepat | Fasilitas |
| Tempat bersih | Fasilitas |
| AC sejuk | Fasilitas |
| Suasana nyaman | Suasana |
| Cocok untuk nongkrong | Suasana |

---

### 12. `order_review_tags`

Junction table antara order dan tag ulasan yang dipilih customer.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK (format: `{order_id}_tag_{index}`) |
| `order_id` | string | FK â†’ `orders.id` |
| `tag_id` | string | FK â†’ `review_tags.id` |

> Digunakan di admin dashboard `RestoDetailPage` untuk menampilkan **Top Review Tags** per restoran. Query menggunakan `where('order_id', 'in', [...])` dengan chunking 30 dokumen (batas Firestore `in` query).

---

### 13. `promo_vouchers`

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `created_by` | string | FK â†’ `users.id` â€” Admin atau Owner pembuat |
| `resto_id` | string | FK â†’ `restaurants.id` â€” **null jika promo global (Admin)** |
| `user_id` | string | FK â†’ `users.id` â€” **null jika publik; isi jika voucher spesifik user** |
| `kode` | string | Kode promo unik |
| `nama` | string | Nama promo |
| `deskripsi` | string | Deskripsi promo |
| `nilai_diskon` | int | Nilai diskon (nominal atau persen) |
| `is_percent` | bool | `true` jika persen, `false` jika nominal tetap |
| `mulai` | timestamp | Tanggal mulai berlaku |
| `berakhir` | timestamp | Tanggal kadaluarsa |
| `is_active` | bool | Status aktif promo |
| `is_used` | bool | Status sudah digunakan â€” **hanya berlaku jika `user_id` tidak null** |

**Aturan scope promo:**

| `resto_id` | `user_id` | Scope |
|---|---|---|
| null | null | Promo global â€” dibuat Admin, berlaku semua restoran |
| isi | null | Promo restoran â€” dibuat Owner, khusus restorannya |
| isi / null | isi | Voucher personal â€” sekali pakai untuk 1 user spesifik |

---

### 14. `reward_poin`

Histori transaksi poin reward customer.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `user_id` | string | FK â†’ `users.id` |
| `order_id` | string | FK â†’ `orders.id` |
| `jumlah_poin` | int | Jumlah poin (positif = earn, negatif = redeem) |
| `created_at` | timestamp | Waktu transaksi poin |

> Total poin aktif tersimpan di `users.poin_reward` sebagai cache. Histori detail ada di koleksi ini.

---

## Relasi Antar Koleksi

```
users â”€â”€(owner_id)â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–º restaurants â”€â”€(resto_id)â”€â”€â–º menus
  â”‚                                â”‚
  â”‚                                â”œâ”€â”€(resto_id)â”€â”€â–º meja
  â”‚                                â””â”€â”€(resto_id)â”€â”€â–º resto_badges â—„â”€â”€ badges
  â”‚
  â””â”€â”€(user_id)â”€â”€â–º orders â”€â”€(app_profit)â”€â”€â–º [Admin Dashboard Profit Calc]
                    â”‚
                    â”œâ”€â”€(tipe_pesanan: dine_in) â”€â”€â–º meja
                    â”œâ”€â”€(promo_id)â”€â”€â”€â”€â”€â”€â”€â”€â”€â–º promo_vouchers
                    â”œâ”€â”€â–º order_items â”€â”€(menu_id)â”€â”€â–º menus
                    â”œâ”€â”€â–º payments
                    â”œâ”€â”€â–º order_review_tags â—„â”€â”€ review_tags â—„â”€â”€(kategori_id)â”€â”€ tag_kategori
                    â””â”€â”€â–º reward_poin
```

---

## Logika Platform Fee (7%)

```
Harga asli menu:        Rp 100.000
Platform fee (7%):      Rp   7.000
Total bayar customer:   Rp 107.000

Disimpan di orders:
  total_price = 107.000
  app_profit  =   7.000   â† Profit aplikasi CariMakan
```

**Kalkulasi di Admin Dashboard:**
- **Total Profit Global** â†’ `SUM(app_profit)` dari semua `orders` where `status == 'completed'`
- **Profit per Resto** â†’ `SUM(app_profit)` GROUP BY `resto_id`
- **Chart harian** â†’ `app_profit` dibucketkan per rentang waktu (`1h`, `7h`, `30h`, `3b`, `1th`)

---

## Catatan Desain Penting

| Keputusan | Penjelasan |
|---|---|
| **`app_profit` di `orders`** | Menyimpan 7% platform fee langsung di dokumen order agar admin bisa kalkulasi profit tanpa re-compute dari `order_items` |
| **`password` di `users`** | Login admin divalidasi server-side (`verifyAdminLogin`). Password plain-text saat ini untuk development â€” **wajib di-hash sebelum production** |
| **`tipe_pesanan` di `orders`** | Enum `dine_in` / `take_away` menggantikan field `tipe` lama untuk konsistensi naming |
| **`status: completed` sebagai filter** | Semua kalkulasi profit dan analitik hanya mengambil order dengan `status == 'completed'` |
| **`id` di `resto_badges` & `order_review_tags`** | Ditambahkan untuk keperluan batch seeding dan operasi delete yang lebih mudah |
| **`avg_rating` & `total_review` di Resto** | Denormalized cache agar tidak perlu agregasi setiap tampilkan daftar restoran |
| **`harga_saat_order` di `order_items`** | Snapshot harga saat transaksi â€” perubahan harga menu di masa depan tidak merusak histori |
| **`fcm_token` di `users`** | Disimpan agar backend bisa kirim push notification langsung ke device spesifik |
| **`is_used` di `promo_vouchers`** | Hanya relevan jika `user_id` tidak null (voucher personal sekali pakai) |


## Database yang Digunakan

**Primary:** Firestore (Firebase â€” NoSQL, real-time)  
**Alternative/Relational:** PostgreSQL (opsi untuk query relasional kompleks)  
**Media Storage:** Firebase Storage (foto menu, foto restoran, QR Code)

> Skema koleksi di bawah ini dibuat berdasarkan **ERD revisi** per 19 Mei 2026.
