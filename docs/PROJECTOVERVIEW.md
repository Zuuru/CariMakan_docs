# CariMakan — Project Overview

## Ringkasan Proyek

**CariMakan** adalah aplikasi mobile berbasis lokasi (*location-based service*) yang membantu pengguna menemukan tempat makan terdekat, melihat menu digital, memesan makanan (take away & dine in), memantau antrian secara real-time, dan melakukan pembayaran digital.

- **Versi SRS:** 1.10
- **Tanggal Dokumen:** 14 Maret 2026
- **Kelas:** TI-2C
- **Tim:**
  - Adila Dimaz Buwana (4.33.24.2.02)
  - M. Oksa Setyarso (4.33.24.2.12)
  - Paulus Ale Kristiawan (4.33.24.2.17)
  - Zulfikri Arya Putra Ismail (4.33.24.2.25)

---

## Tujuan Aplikasi

Menjadi platform kuliner digital yang:
- Membantu pengguna menemukan restoran terdekat berbasis GPS
- Mendigitalisasi UMKM kuliner lokal
- Menyelesaikan masalah antrian dan pembayaran yang tidak efisien
- Memberikan pengalaman pemesanan yang cepat dan transparan

---

## Target Pengguna

| Pengguna | Deskripsi |
|---|---|
| **Customer** | Mahasiswa, pekerja, wisatawan yang ingin menemukan dan memesan makanan |
| **Owner Resto** | Pemilik UMKM kuliner yang ingin mendigitalisasi operasional restoran |
| **Admin** | Pengelola platform yang memverifikasi restoran dan memoderasi konten |

---

## Fitur Utama (10 Fitur)

1. **Sistem Autentikasi & Manajemen Akun** — Login/register dengan JWT untuk semua role
2. **Menu QR Code** — Scan QR untuk akses menu digital yang selalu diperbarui
3. **Monitoring Antrian Real-Time** — Pantau jumlah antrian dan estimasi waktu tunggu
4. **Pembayaran Digital Terintegrasi** — e-wallet (GoPay, OVO, DANA), kartu, transfer bank via Midtrans
5. **Manajemen Pemesanan** — Take away & dine in dengan status pesanan real-time
6. **Pencarian Berbasis GPS** — Integrasi Google Maps API untuk temukan resto terdekat
7. **Dashboard Pengelolaan Restoran** — Upload menu, kelola stok, analitik penjualan
8. **Notifikasi Otomatis** — Push notification status pesanan, promo, antrian
9. **Badge & Reward** — Sistem loyalitas poin yang dapat ditukar voucher diskon
10. **Interaksi Data Dua Arah** — Sinkronisasi real-time antara customer ↔ owner

---

## Ruang Lingkup (In Scope)

- Pencarian restoran berbasis GPS
- Menu digital via QR Code
- Sistem antrian real-time
- Pemesanan take away & dine in
- Pembayaran digital (cashless)
- Notifikasi dan reward pengguna

## Di Luar Ruang Lingkup (Out of Scope)

- Layanan delivery dengan mitra eksternal (GoFood, GrabFood, dll.)
- Reservasi tempat duduk
- Integrasi sistem eksternal secara default

---

## Platform Target

- **Mobile:** Android & iOS
- **Web (Admin Panel):** Browser modern (Chrome, Edge, Firefox)