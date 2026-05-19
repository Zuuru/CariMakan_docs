# CariMakan — Workflow

Dokumen ini menjelaskan alur kerja (user flow) utama dalam aplikasi CariMakan untuk setiap aktor, berdasarkan SRS dan Activity Diagram.

---

## Alur 1 — Customer: Dine In

> Customer datang ke restoran, scan QR di meja, pesan, bayar, tunggu.

```
Buka aplikasi
  → Lihat daftar restoran (GPS)
  → Pilih restoran
  → Lihat info antrian & menu
  → [Antrian dibuka?]
      - Tidak → Tampil pesan "Antrian ditutup"
      - Ya → [Antrian penuh?]
                - Ya → Tampil pesan "Antrian penuh"
                - Tidak → Ambil nomor antrian
  → Dapat nomor antrian
  → Menunggu giliran dipanggil (push notif)
  → Datang ke restoran → Scan QR Code di meja
  → Pilih menu & buat order
  → Melakukan pembayaran (pre-paid via Midtrans)
  → Menunggu pesanan diproses
  → Terima push notif "Pesanan siap"
  → Terima pesanan
  → [Kasih review?]
      - Ya → Isi rating & ulasan
      - Tidak → Selesai
```

---

## Alur 2 — Customer: Take Away

> Customer pesan dari luar restoran, bayar, datang ambil pada waktu yang dipilih.

```
Buka aplikasi
  → Pilih restoran
  → Lihat menu
  → Pilih menu & tentukan waktu pickup
  → Buat order (type: take_away)
  → Melakukan pembayaran (pre-paid via Midtrans)
  → Dapat QR Code dinamis sebagai bukti pengambilan
  → Menunggu push notif "Pesanan siap diambil"
  → Datang ke restoran → Tunjukkan QR Code dinamis
  → Ambil pesanan
  → [Kasih review?]
      - Ya → Isi rating & ulasan
      - Tidak → Selesai
```

---

## Alur 3 — Owner Restoran: Kelola Pesanan Harian

```
Login ke aplikasi (masuk ke halaman dashboard owner)
  → Atur status antrian (Buka / Tutup)
  → Lihat dashboard antrian real-time
      - Kolom antrian Dine In
      - Kolom antrian Take Away
  → [Ada pesanan masuk?]
      - Ya → Lihat detail pesanan
           → Proses pesanan (masak)
           → Tandai pesanan "Siap"
           → Sistem kirim push notif otomatis ke customer
  → Panggil antrian berikutnya
  → Repeat hingga tutup
  → Lihat laporan harian (summary transaksi)
```

---

## Alur 4 — Owner Restoran: Setup Awal Restoran

```
Register akun (role: owner)
  → Isi data restoran (nama, alamat, foto, jam, GPS)
  → Pilih badge fasilitas (WiFi, AC, dll.)
  → Submit → Status: Menunggu verifikasi Admin
  → [Admin approve?]
      - Approve → Restoran aktif, bisa mulai terima pesanan
      - Reject → Notifikasi ditolak, perbaiki data & submit ulang
  → Upload menu (nama, harga, foto, kategori)
  → Generate QR Code untuk setiap meja (download & cetak)
  → Siap beroperasi
```

---

## Alur 5 — Admin: Verifikasi Restoran Baru

```
Login ke Web Admin Panel
  → Buka menu "Restoran Pending"
  → Pilih restoran yang menunggu review
  → Cek kelengkapan data (nama, alamat, foto, GPS)
  → [Keputusan?]
      - Approve → Status restoran jadi "active"
               → Owner dapat notifikasi persetujuan
      - Reject → Owner dapat notifikasi penolakan + alasan
  → Lanjut ke restoran berikutnya
```

---

## Alur 6 — Admin: Kelola Platform

```
Login ke Web Admin Panel (http://localhost:3000)
  → Validasi: email + password dicek via server action ke Firestore
  → Hanya role 'admin' + status 'aktif' yang bisa masuk
  → Dashboard Utama
      - Card: Total Restoran, Total Customer, Total User, Total Profit (7% fee)
      - Profit Chart: tren profit harian dengan filter 1h/7h/30h/3b/1th
      - Tabel profit per restoran (dari aggregasi orders.app_profit)
  → Tab Manajemen User
      - Lihat semua user (customer, owner, admin)
      - CRUD: tambah, edit semua field (nama, email, role, status, WA, foto_url, poin_reward, fcm_token), hapus
  → Tab Restoran
      - Lihat semua restoran dengan status (pending/aktif/suspend)
      - Approve / Reject restoran pending
      - Suspend / Aktifkan restoran
      - Klik "Detail" → buka halaman analitik per-restoran:
          - KPI: total profit, total revenue, total order, rating
          - Tren profit harian (chart 30 hari)
          - Info owner
          - Top review tags
          - Showcase menu (filter tersedia/habis)
          - 10 transaksi terbaru
  → Tab Promo & Voucher
      - Buat/edit/nonaktifkan promo global atau per-restoran
  → Tab Statistik Platform
      - Lihat metrik platform secara keseluruhan
```


---

## Alur 7 — Sistem: Update Status Pesanan (Otomatis)

> Ini adalah alur internal sistem yang terjadi di backend.

```
Owner klik "Tandai Siap" di dashboard
  → Backend update status order: "ready"
  → Backend update Firestore → listener di mobile app customer aktif
  → Backend trigger FCM / OneSignal
  → Push notif terkirim ke HP customer: "Pesanan kamu siap!"
  → Customer buka notif → masuk ke halaman detail pesanan
```

---

## Alur 8 — Sistem: Proses Pembayaran (Midtrans Webhook)

```
Customer klik bayar di app
  → Backend buat transaksi ke Midtrans API
  → Midtrans return payment_url
  → Customer diarahkan ke halaman pembayaran Midtrans
  → Customer selesai bayar (GoPay / OVO / Transfer / dll.)
  → Midtrans kirim callback ke backend: POST /payments/webhook
  → Backend verifikasi signature Midtrans
  → [Status pembayaran?]
      - Success → Update order status: "processing"
               → Simpan data payment ke Firestore
               → Order masuk ke antrian dapur owner
               → Poin reward ditambahkan ke akun customer
      - Failed / Expired → Update order status: "cancelled"
                        → Notifikasi ke customer
```

---

## State Machine: Order Status

```
[pending]
    │
    │ Pembayaran sukses (webhook Midtrans)
    ▼
[processing]
    │
    │ Owner tandai pesanan selesai dimasak
    ▼
[ready]
    │
    │ Customer ambil pesanan / pesanan diserahkan
    ▼
[completed]   ← Digunakan sebagai filter analitik di admin dashboard

Dari state manapun:
[cancelled] ← Pembayaran gagal / timeout / owner tolak
```

> **Admin Dashboard Filter:** Semua kalkulasi profit (`app_profit`) hanya mengambil orders dengan `status == 'completed'`.


---

## State Machine: Queue Status

```
[waiting]     ← Customer ambil nomor antrian
    │
    │ Owner panggil antrian berikutnya
    ▼
[called]      ← Push notif dikirim ke customer
    │
    │ Customer datang & pesanan selesai
    ▼
[done]

[cancelled]   ← Customer batalkan / antrian tutup
```

---

## Notifikasi yang Dikirim Sistem

| Event | Penerima | Isi Notifikasi |
|---|---|---|
| Antrian dipanggil | Customer | "Giliran kamu sudah tiba! Segera ke restoran." |
| Pesanan diproses | Customer | "Pesananmu sedang dimasak 🍳" |
| Pesanan siap | Customer | "Pesananmu sudah siap! Silakan diambil." |
| Pembayaran berhasil | Customer | "Pembayaran berhasil. Nomor antrian: #X" |
| Pembayaran gagal | Customer | "Pembayaran gagal. Silakan coba lagi." |
| Restoran diapprove | Owner | "Selamat! Restoran kamu sudah aktif." |
| Restoran ditolak | Owner | "Pendaftaran restoran ditolak. Alasan: ..." |
| Pesanan baru masuk | Owner | "Ada pesanan baru masuk! Cek dashboard." |
| Promo baru | Customer | "Ada promo spesial hari ini! Gunakan kode: XXXX" |