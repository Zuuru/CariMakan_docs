# CariMakan — Database

## Database yang Digunakan

**Primary:** Firestore (Firebase — NoSQL, real-time)  
**Alternative/Relational:** PostgreSQL (opsi untuk query relasional kompleks)  
**Media Storage:** Firebase Storage (foto menu, foto restoran, QR Code)

> Skema koleksi di bawah ini dibuat berdasarkan **ERD revisi** per 08 Mei 2026.

---

## Alasan Memilih Firestore

- **Real-time listener** — cocok untuk fitur antrian dan status pesanan yang harus update otomatis
- **Scalable** — mendukung concurrent users tanpa konfigurasi server manual
- **Integrasi** — native dengan Firebase ecosystem (FCM, Auth, Storage)
- **Managed** — tidak perlu maintain server database sendiri

---

## Ringkasan Entitas (13 Koleksi)

| Koleksi | Keterangan |
|---|---|
| `users` | Semua pengguna (customer, owner, admin) |
| `restaurants` | Data restoran |
| `menus` | Menu per restoran |
| `meja` | QR Code meja per restoran (dine in) |
| `orders` | Pesanan customer (termasuk rating & antrian) |
| `order_items` | Item detail dari setiap pesanan |
| `payments` | Transaksi pembayaran via Midtrans |
| `badges` | Master data badge fasilitas |
| `resto_badges` | Junction: restoran ↔ badge |
| `review_tags` | Master tag ulasan (label review) |
| `order_review_tags` | Junction: order ↔ tag ulasan |
| `promo_vouchers` | Promo & voucher diskon |
| `reward_poin` | Histori poin reward customer |

---

## Skema Detail per Koleksi

---

### 1. `users`

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK — UID dari Firebase Auth |
| `nama` | string | Nama lengkap |
| `email` | string | Email login |
| `role` | enum | `customer` / `owner` / `admin` |
| `foto_url` | string | URL foto profil (Firebase Storage) |
| `poin_reward` | int | Total poin aktif yang dimiliki |
| `fcm_token` | string | Token FCM untuk push notification |
| `status` | enum | `aktif` / `suspend` |
| `url_whatsapp` | string | Nomor WhatsApp (opsional) |
| `created_at` | timestamp | Waktu registrasi |

---

### 2. `restaurants`

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `owner_id` | string | FK → `users.id` |
| `nama` | string | Nama restoran |
| `lokasi` | geopoint | Koordinat GPS (lat, lng) |
| `foto_uri` | string | URL foto utama restoran |
| `jam_buka` | string | Jam operasional (misal: `"08:00-22:00"`) |
| `status` | enum | `pending` / `aktif` / `suspend` |
| `url_whatsapp` | string | Kontak WhatsApp restoran |
| `avg_rating` | float | Rata-rata rating — **denormalized cache** |
| `total_review` | int | Jumlah total review — **denormalized cache** |
| `created_at` | timestamp | Tanggal daftar |

> `avg_rating` dan `total_review` di-update otomatis setiap ada review baru agar tidak perlu agregasi tiap query.

---

### 3. `menus`

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `resto_id` | string | FK → `restaurants.id` |
| `nama` | string | Nama menu |
| `harga` | int | Harga dalam Rupiah |
| `foto_url` | string | Foto menu |
| `deskripsi` | string | Deskripsi menu |
| `tersedia` | bool | Status ketersediaan menu |

---

### 4. `meja`

Data meja per restoran untuk keperluan Dine In.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `resto_id` | string | FK → `restaurants.id` |
| `nomor_meja` | int | Nomor meja |
| `qr_code_url` | string | URL QR Code statis meja (cetak 1x, permanen) |

---

### 5. `orders`

Pesanan utama — mencakup status antrian, payment, dan rating dalam satu dokumen.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `user_id` | string | FK → `users.id` |
| `resto_id` | string | FK → `restaurants.id` |
| `meja_id` | string | FK → `meja.id` — **null jika take away** |
| `promo_id` | string | FK → `promo_vouchers.id` — **null jika tidak pakai promo** |
| `tipe` | enum | `dinein` / `takeaway` |
| `total` | int | Total harga pesanan (Rupiah) |
| `nomor_antrian` | int | Nomor antrian yang di-generate sistem |
| `status` | enum | `pending` / `proses` / `siap` / `selesai` |
| `status_antrian` | enum | `menunggu` / `dipanggil` / `proses` / `selesai` |
| `payment_status` | enum | `unpaid` / `paid` / `failed` |
| `payment_token` | string | Token Midtrans untuk redirect ke halaman bayar |
| `qr_pickup` | string | QR Code dinamis untuk take away — **null jika dine in** |
| `jam_pickup` | timestamp | Waktu pickup yang dipilih — **null jika dine in** |
| `rating_pelayanan` | int | Rating pelayanan 1–5 — **null until reviewed** |
| `rating_makanan` | int | Rating makanan 1–5 — **null until reviewed** |
| `rating_fasilitas` | int | Rating fasilitas 1–5 — **null until reviewed** |
| `rating_total` | float | Rata-rata dari 3 rating — **null until reviewed** |
| `komentar` | string | Komentar ulasan — **null until reviewed** |
| `sudah_direview` | bool | Flag apakah order ini sudah diberi review |
| `created_at` | timestamp | Waktu pesanan dibuat |

---

### 6. `order_items`

Detail item di dalam setiap pesanan.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `order_id` | string | FK → `orders.id` |
| `menu_id` | string | FK → `menus.id` |
| `qty` | int | Jumlah yang dipesan |
| `harga_saat_order` | int | Snapshot harga saat transaksi |
| `catatan` | string | Catatan khusus (misal: "tanpa sambal") |

> `harga_saat_order` adalah snapshot — perubahan harga menu di masa depan tidak merusak histori transaksi.

---

### 7. `payments`

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `order_id` | string | FK → `orders.id` |
| `gateway_token` | string | Token/ID transaksi dari Midtrans |
| `method` | string | `gopay` / `ovo` / `qris` / `bank` / dll. |
| `amount` | int | Nominal pembayaran (Rupiah) |
| `status` | enum | `pending` / `success` / `failed` |
| `paid_at` | timestamp | Waktu pembayaran berhasil dikonfirmasi |

---

### 8. `badges`

Master data badge/fasilitas restoran.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `nama` | string | Nama badge (misal: "WiFi", "AC", "Area Parkir") |
| `icon` | string | URL atau nama icon |

---

### 9. `resto_badges`

Junction table antara restoran dan badge yang dimilikinya.

| Field | Tipe | Keterangan |
|---|---|---|
| `resto_id` | string | FK → `restaurants.id` |
| `badge_id` | string | FK → `badges.id` |

---

### 10. `review_tags`

Master tag label untuk ulasan (sistem kategori review).

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `label` | string | Label tag (misal: "WiFi cepat", "Makanan enak") |
| `icon` | string | Icon tag |
| `kategori` | enum | `pelayanan` / `makanan` / `fasilitas` |

---

### 11. `order_review_tags`

Junction table antara order dan tag ulasan yang dipilih customer.

| Field | Tipe | Keterangan |
|---|---|---|
| `order_id` | string | FK → `orders.id` |
| `tag_id` | string | FK → `review_tags.id` |

---

### 12. `promo_vouchers`

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `created_by` | string | FK → `users.id` — Admin atau Owner pembuat |
| `resto_id` | string | FK → `restaurants.id` — **null jika promo global (Admin)** |
| `user_id` | string | FK → `users.id` — **null jika publik; isi jika voucher spesifik user** |
| `kode` | string | Kode promo unik |
| `nama` | string | Nama promo |
| `deskripsi` | string | Deskripsi promo |
| `nilai_diskon` | int | Nilai diskon (nominal atau persen) |
| `is_percent` | bool | `true` jika persen, `false` jika nominal tetap |
| `mulai` | timestamp | Tanggal mulai berlaku |
| `berakhir` | timestamp | Tanggal kadaluarsa |
| `is_active` | bool | Status aktif promo |
| `is_used` | bool | Status sudah digunakan — **hanya berlaku jika `user_id` tidak null** |

**Aturan scope promo:**

| `resto_id` | `user_id` | Scope |
|---|---|---|
| null | null | Promo global — dibuat Admin, berlaku semua restoran |
| isi | null | Promo restoran — dibuat Owner, khusus restorannya |
| isi / null | isi | Voucher personal — sekali pakai untuk 1 user spesifik |

---

### 13. `reward_poin`

Histori transaksi poin reward customer.

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | PK |
| `user_id` | string | FK → `users.id` |
| `order_id` | string | FK → `orders.id` |
| `jumlah_poin` | int | Jumlah poin (positif = earn, negatif = redeem) |
| `created_at` | timestamp | Waktu transaksi poin |

> Total poin aktif tersimpan di `users.poin_reward` sebagai cache. Histori detail ada di koleksi ini.

---

## Relasi Antar Koleksi

```
users ──(owner_id)──────────► restaurants ──(resto_id)──► menus
  │                                │                        
  │                                ├──(resto_id)──► meja   
  │                                └──(resto_id)──► resto_badges ◄── badges
  │
  └──(user_id)──► orders
                    │
                    ├──(meja_id)──────────► meja
                    ├──(promo_id)─────────► promo_vouchers
                    ├──► order_items ──(menu_id)──► menus
                    ├──► payments
                    ├──► order_review_tags ◄── review_tags
                    └──► reward_poin
```

---

## Catatan Desain Penting

| Keputusan | Penjelasan |
|---|---|
| **Rating disimpan di `orders`** | Tidak ada koleksi `reviews` terpisah — rating pelayanan/makanan/fasilitas dan komentar langsung di dokumen order untuk menyederhanakan query |
| **`avg_rating` & `total_review` di Resto** | Denormalized cache agar tidak perlu agregasi setiap tampilkan daftar restoran |
| **`harga_saat_order` di `order_items`** | Snapshot harga saat transaksi — perubahan harga menu di masa depan tidak merusak histori |
| **`payment_token` di `orders`** | Token Midtrans di-cache di order untuk redirect bayar tanpa query ke koleksi payments |
| **`qr_pickup` di `orders`** | QR dinamis untuk take away — berbeda dari `qr_code_url` di meja yang statis dan permanen |
| **`sudah_direview` di `orders`** | Flag boolean untuk mencegah customer review lebih dari sekali per transaksi |
| **`is_used` di `promo_vouchers`** | Hanya relevan jika `user_id` tidak null (voucher personal sekali pakai) |
| **`fcm_token` di `users`** | Disimpan agar backend bisa kirim push notification langsung ke device spesifik tanpa roundtrip ke FCM registry |