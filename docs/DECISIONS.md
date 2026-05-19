# CariMakan — Decisions (Keputusan Arsitektur)

Dokumen ini mencatat keputusan-keputusan teknis utama yang diambil dalam pengembangan aplikasi CariMakan beserta alasan di baliknya. Gunakan sebagai referensi ketika ada pertanyaan "kenapa pakai X dan bukan Y?"

---

## ADR-001 — Flutter sebagai Framework Mobile

**Status:** Accepted  
**Konteks:** Butuh satu codebase untuk Android dan iOS.

**Keputusan:** Menggunakan Flutter (dengan Expo Framework sebagai tooling).

**Alasan:**
- Single codebase untuk Android & iOS — efisiensi waktu development
- Performa mendekati native (compiled ke native ARM)
- Komponen UI yang kaya dan konsisten antar platform
- Ekosistem Firestore & FCM terintegrasi baik dengan Flutter
- Tim memilih Flutter/Expo sebagai stack yang familiar

**Alternatif yang ditolak:** React Native — lebih banyak overhead dependency, debugging lebih kompleks untuk tim kecil.

---

## ADR-002 — Next.js untuk Web Admin Panel

**Status:** Accepted  
**Konteks:** Admin membutuhkan panel web terpisah untuk verifikasi restoran, moderasi, dan statistik platform.

**Keputusan:** Menggunakan Next.js + TypeScript.

**Alasan:**
- SSR (Server-Side Rendering) untuk load lebih cepat di halaman dashboard
- TypeScript meningkatkan keamanan tipe data di panel admin yang kompleks
- Mudah di-deploy ke Vercel / GCP
- API Routes Next.js bisa digunakan untuk endpoint admin ringan

---

## ADR-003 — Node.js + Express sebagai Backend

**Status:** Accepted  
**Konteks:** Butuh backend yang cepat untuk API REST dan cocok dengan ekosistem JavaScript/TypeScript.

**Keputusan:** Node.js + Express.js sebagai core backend.

**Alasan:**
- Konsisten dengan ekosistem JavaScript (Flutter + Next.js sudah JS/TS-based)
- Non-blocking I/O cocok untuk banyak concurrent request (real-time antrian)
- Mudah integrasi dengan Firebase SDK
- Ekosistem npm sangat kaya untuk payment gateway, auth, notifikasi

---

## ADR-004 — Firestore sebagai Database Utama

**Status:** Accepted  
**Konteks:** Butuh real-time sync untuk fitur antrian dan status pesanan.

**Keputusan:** Firestore (Firebase NoSQL) sebagai database utama.

**Alasan:**
- **Real-time listener** bawaan — update antrian dan status pesanan langsung terkirim ke client tanpa polling manual
- **Managed & scalable** — tidak perlu setup atau maintain server database
- Integrasi native dengan Firebase Auth, FCM, Storage
- Cocok untuk data yang sering berubah (status pesanan, antrian)

**Trade-off:**
- Query kompleks (JOIN, agregasi) lebih sulit dibanding SQL
- Untuk laporan/analitik, bisa dikombinasikan dengan PostgreSQL atau BigQuery

**Alternatif yang ditolak:** PostgreSQL sebagai primary DB — kurang cocok untuk real-time sync tanpa tambahan infra (WebSocket, Supabase Realtime, dll.).

---

## ADR-005 — JWT untuk Autentikasi

**Status:** Accepted  
**Konteks:** Perlu sistem auth yang stateless dan mendukung multi-role (Customer, Owner, Admin).

**Keputusan:** JWT (JSON Web Token) Authentication.

**Alasan:**
- Stateless — tidak butuh session storage di server
- Payload JWT bisa menyimpan `role` untuk RBAC langsung di token
- Standar industri yang didukung semua framework
- Compatible dengan mobile dan web client

**Implementasi:**
- Token digenerate saat login, dikirim di header `Authorization: Bearer <token>`
- Token expire setelah durasi tertentu, refresh token untuk perpanjangan
- Role (`customer`, `owner`, `admin`) divalidasi di middleware backend

---

## ADR-006 — Midtrans sebagai Payment Gateway

**Status:** Accepted  
**Konteks:** Butuh payment gateway lokal Indonesia yang support e-wallet populer.

**Keputusan:** Midtrans Payment Gateway.

**Alasan:**
- Support semua metode pembayaran lokal: GoPay, OVO, DANA, kartu, transfer bank
- Berstandar PCI-DSS (aman untuk data kartu)
- Banyak digunakan UMKM Indonesia — familier di ekosistem lokal
- SDK tersedia untuk Node.js dan Flutter
- Sistem callback/webhook untuk konfirmasi pembayaran server-side

**Model pembayaran:** Pre-paid — pesanan hanya diproses setelah pembayaran sukses dikonfirmasi Midtrans.

---

## ADR-007 — OneSignal / FCM untuk Push Notification

**Status:** Accepted  
**Konteks:** Butuh push notification real-time untuk status pesanan, antrian dipanggil, dan promo.

**Keputusan:** Firebase Cloud Messaging (FCM) sebagai primary, OneSignal sebagai alternatif/layer abstraksi.

**Alasan:**
- FCM gratis dan terintegrasi langsung dengan Firebase ekosistem
- OneSignal memberikan dashboard yang lebih user-friendly untuk kelola notifikasi
- Keduanya support Android & iOS
- Trigger notifikasi dari backend saat owner update status pesanan

---

## ADR-008 — Google Maps API untuk Fitur GPS

**Status:** Accepted  
**Konteks:** Butuh fitur pencarian restoran berbasis lokasi real-time.

**Keputusan:** Google Maps API (Places + Geolocation).

**Alasan:**
- Standar industri untuk location-based service
- Data lokasi paling akurat dan coverage luas di Indonesia
- SDK tersedia untuk Flutter (google_maps_flutter)
- Integrasi dengan GeoPoint Firestore untuk query restoran terdekat

---

## ADR-009 — Sistem Pre-paid (Bayar Dulu, Antre Kemudian)

**Status:** Accepted  
**Konteks:** Perlu memastikan pesanan yang masuk ke dapur adalah pesanan yang sudah dibayar untuk menghindari pesanan fiktif.

**Keputusan:** Pembayaran dilakukan **sebelum** pesanan masuk ke antrian dapur.

**Alasan:**
- Mencegah pesanan yang tidak jadi dibayar (ghost order)
- Meningkatkan kepastian bagi pemilik restoran
- Mengurangi kerugian bahan baku akibat pesanan batal
- Alur: Pilih menu → Bayar → Masuk antrian → Pesanan diproses

---

## ADR-010 — Dua Tipe Pemesanan: Dine In & Take Away

**Status:** Accepted  
**Konteks:** SRS mendefinisikan dua skenario pemesanan dengan alur berbeda.

**Keputusan:** Membedakan alur `dine_in` dan `take_away` dalam sistem.

**Perbedaan:**
| Aspek | Dine In | Take Away |
|---|---|---|
| **QR Code** | QR statis di meja (generate sekali, cetak permanen) | QR dinamis sebagai bukti pengambilan |
| **Identifikasi** | Nomor meja dari scan QR | Waktu pickup yang dipilih customer |
| **Antrian** | Antrian dine in terpisah di dashboard owner | Antrian take away terpisah di dashboard owner |
| **Alur** | Scan QR meja → Pilih menu → Bayar → Tunggu | Pilih menu → Pilih waktu pickup → Bayar → Ambil |

---

## ADR-011 — Next.js Server Actions untuk Admin (Tanpa Backend Terpisah)

**Status:** Accepted  
**Konteks:** Admin dashboard perlu akses database yang aman tanpa mengekspos kredensial ke client.

**Keputusan:** Gunakan Next.js **Server Actions** + Firebase **Admin SDK** langsung di `carimakan_admin`, tanpa backend Node.js/Express terpisah untuk admin.

**Alasan:**
- Server Actions berjalan di server (tidak pernah terekspos ke browser) — aman untuk menyimpan service account key
- Menghilangkan satu layer infra (tidak perlu maintain backend Express terpisah untuk admin)
- Cocok untuk admin-only operations yang tidak membutuhkan real-time listener
- Deployment lebih sederhana (Next.js + Vercel/GCP Cloud Run)

**Trade-off:**
- Tidak bisa digunakan untuk real-time Firestore listeners (butuh solusi lain jika diperlukan)
- Semua query dijalankan synchronous saat request — bukan streaming

---

## ADR-012 — Platform Fee 7% via Field `app_profit` di `orders`

**Status:** Accepted  
**Konteks:** CariMakan perlu model monetisasi dari setiap transaksi yang bisa ditrack dan diaudit.

**Keputusan:** Platform mengambil **7% dari harga asli** setiap transaksi sebagai platform fee. Nilai ini disimpan langsung di field `app_profit` pada koleksi `orders` saat transaksi dibuat.

**Alasan:**
- Pre-computed — tidak perlu agregasi real-time yang mahal
- Mudah di-query di admin dashboard: `SUM(app_profit) WHERE status = 'completed'`
- Audit trail yang jelas — setiap order punya catatan berapa yang masuk ke platform
- Scalable untuk penambahan tier fee berbeda di masa depan (misal: fee berbeda per kategori resto)

**Implementasi:**
```
harga_asli = harga menu asli (Rupiah)
app_profit = floor(harga_asli × 0.07)
total_price = harga_asli + app_profit   ← yang dibayar customer
```

**Tampilan di Admin Dashboard:**
- Card Profit: akumulasi `SUM(app_profit)` semua orders `completed`
- Chart Harian: bucketing `app_profit` per rentang waktu (1h, 7h, 30h, 3b, 1th)
- Detail Resto: `SUM(app_profit)` GROUP BY `resto_id`