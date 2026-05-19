# CariMakan — Architecture

## Gambaran Arsitektur Sistem

CariMakan menggunakan arsitektur **client-server berbasis cloud** dengan pendekatan mobile-first. Sistem terdiri dari tiga layer utama: Frontend Mobile, Backend API, dan Cloud Infrastructure.

---

## Stack Teknologi

| Layer | Teknologi |
|---|---|
| **Frontend Mobile** | Flutter — Android & iOS (Customer & Owner) |
| **Frontend Web Admin** | Next.js + TypeScript (Admin Dashboard) |
| **Admin Auth** | Server Actions (Next.js) + Firebase Admin SDK |
| **Database** | Firestore (Firebase NoSQL) |
| **Cloud** | Google Cloud Platform + Firebase Storage |
| **Payment** | Midtrans Payment Gateway |
| **Auth** | Validasi server-side via `verifyAdminLogin` (admin), JWT untuk mobile |
| **Push Notification** | Firebase Cloud Messaging (FCM) |
| **Maps** | Google Maps API |

> **Catatan Implementasi:** Admin dashboard (`carimakan_admin`) tidak menggunakan backend Node.js terpisah — semua operasi database dilakukan langsung dari Next.js **Server Actions** menggunakan **Firebase Admin SDK**. Repository Customer dan Owner terpisah dari admin.


---

## Diagram Arsitektur (Overview)

```
┌─────────────────────────────────────────────────────┐
│                   CLIENT LAYER                       │
│                                                     │
│  ┌──────────────────┐    ┌────────────────────────┐ │
│  │  Mobile App      │    │  Web Admin Panel       │ │
│  │  (Flutter/Expo)  │    │  (Next.js + TypeScript)│ │
│  │  Android & iOS   │    │  Browser-based         │ │
│  └────────┬─────────┘    └──────────┬─────────────┘ │
└───────────┼──────────────────────────┼───────────────┘
            │ HTTPS / REST / GraphQL   │
┌───────────┼──────────────────────────┼───────────────┐
│           │      BACKEND LAYER       │               │
│  ┌────────▼──────────────────────────▼────────────┐  │
│  │              Node.js + Express.js               │  │
│  │         (Business Logic / API Gateway)          │  │
│  │  - JWT Auth Middleware                          │  │
│  │  - Role-based Access Control (RBAC)             │  │
│  │  - REST API / GraphQL Endpoints                 │  │
│  └───────────────────────┬─────────────────────────┘  │
└──────────────────────────┼────────────────────────────┘
                           │
┌──────────────────────────┼────────────────────────────┐
│           SERVICES & INTEGRATION LAYER               │
│  ┌──────────────┐ ┌──────┴──────┐ ┌────────────────┐ │
│  │   Midtrans   │ │  Google     │ │  OneSignal /   │ │
│  │   Payment    │ │  Maps API   │ │  FCM           │ │
│  │   Gateway    │ │  (GPS)      │ │  (Notifikasi)  │ │
│  └──────────────┘ └─────────────┘ └────────────────┘ │
└──────────────────────────┬────────────────────────────┘
                           │
┌──────────────────────────┼────────────────────────────┐
│              DATA LAYER  │                            │
│  ┌──────────────────┐  ┌─┴────────────────────────┐  │
│  │    Firestore     │  │  Firebase Storage        │  │
│  │  (Real-time DB)  │  │  (Media / Asset)         │  │
│  └──────────────────┘  └──────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐ │
│  │         Google Cloud Platform (GCP)              │ │
│  └──────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────┘
```

---

## Role & Akses

### Customer (Mobile App)
- Akses fitur pencarian GPS, QR Menu, antrian, order, pembayaran, reward
- Authenticated via JWT

### Owner Resto (Mobile App — Dashboard Mode)
- Akses ke dashboard khusus restoran berdasarkan **role** dari JWT token
- Kelola menu, antrian, pesanan, laporan

### Admin (Web App — `carimakan_admin`)
- Akses penuh via web admin panel (Next.js, browser)
- **Autentikasi:** Login divalidasi server-side via `verifyAdminLogin` — hanya `role: admin` dan `status: aktif` yang bisa masuk
- Verifikasi restoran baru, CRUD user, kelola promo
- Dashboard analytics: profit 7% platform fee, chart per rentang waktu, detail per-restoran (menu, orders, review tags)
- **Tidak melewati backend Node.js** — langsung ke Firestore via Firebase Admin SDK


---

## Komunikasi Antar Komponen

| Protokol | Keterangan |
|---|---|
| **HTTP/HTTPS** | Semua komunikasi API menggunakan HTTPS |
| **Port 80** | HTTP (redirect ke HTTPS) |
| **Port 443** | HTTPS (production) |
| **Format Data** | JSON (standar REST/GraphQL) |
| **Real-time** | Firebase Firestore listener untuk data antrian & status pesanan |
| **Auth Token** | JWT Bearer Token di setiap request header |

---

## Keamanan

- **JWT Authentication** — token-based login untuk semua role
- **End-to-end Encryption** — proteksi data sensitif pengguna
- **RBAC (Role-Based Access Control)** — Customer, Owner, Admin memiliki hak akses berbeda
- **HTTPS Only** — semua komunikasi terenkripsi
- **Midtrans** — payment gateway berstandar PCI-DSS
- **GCP & Firebase** — cloud infrastructure dengan keamanan managed