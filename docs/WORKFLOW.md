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
Login ke Web Admin Panel
  → Dashboard Statistik
      - Total transaksi hari ini
      - Jumlah user aktif
      - Restoran dengan pesanan terbanyak
  → Manajemen User
      - Suspend / aktifkan akun user
      - Hapus akun yang melanggar aturan
  → Moderasi Ulasan
      - Sembunyikan review yang tidak sesuai
  → Kelola Promo & Voucher
      - Buat kode promo baru
      - Atur nilai diskon & masa berlaku
  → Kelola Badge Database
      - Atur syarat & konfigurasi badge achievement
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
[done]

Dari state manapun:
[cancelled] ← Pembayaran gagal / timeout / owner tolak
```

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