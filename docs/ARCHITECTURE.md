# CariMakan — Architecture

## Gambaran Arsitektur Sistem

CariMakan menggunakan arsitektur **client-server berbasis cloud** dengan pendekatan mobile-first. Sistem terdiri dari tiga layer utama: Frontend Mobile, Backend API, dan Cloud Infrastructure.

---

## Stack Teknologi

| Layer | Teknologi |
|---|---|
| **Frontend Mobile** | Flutter (Expo Framework) — Android & iOS |
| **Frontend Web Admin** | Next.js + TypeScript |
| **Backend** | Node.js + Express.js |
| **API Style** | RESTful API / GraphQL |
| **Database** | Firestore (Firebase) / PostgreSQL |
| **Cloud** | Google Cloud Platform + Firebase Storage |
| **Payment** | Midtrans Payment Gateway |
| **Auth** | JWT Authentication |
| **Push Notification** | OneSignal / Firebase Cloud Messaging (FCM) |
| **Maps** | Google Maps API |

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

### Admin (Web App)
- Akses penuh via web admin panel
- Verifikasi restoran baru, moderasi user, kelola promo & statistik

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