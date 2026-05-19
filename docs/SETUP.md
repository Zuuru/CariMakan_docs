# CariMakan — Setup Guide

## Prerequisites

Sebelum mulai, pastikan tools berikut sudah terinstall:

| Tool | Versi Minimum | Keterangan |
|---|---|---|
| Node.js | 18.x LTS | Runtime backend |
| npm | 9.x | Package manager |
| Flutter | 3.x | Mobile frontend |
| Dart | 3.x | Bahasa Flutter |
| Git | Latest | Version control |
| Firebase CLI | Latest | Deploy & manage Firebase |

---

## Struktur Monorepo (Rekomendasi)

```
CariMakan/
├── carimakan_admin/          # Next.js admin dashboard (repo ini)
│   ├── src/
│   │   ├── app/
│   │   │   ├── page.tsx          # Dashboard utama
│   │   │   └── actions.ts        # Server Actions (Firestore Admin SDK)
│   │   └── components/       # UI components (UsersTab, RestosTab, dll)
│   ├── setupdb/
│   │   └── firestore-setup.js  # Seeder Firestore (node firestore-setup.js)
│   ├── .env.local            # Kredensial Firebase Admin
│   └── package.json
├── carimakan_mobile/         # Flutter app (Customer & Owner) — repo terpisah
└── CariMakan_docs/           # Dokumentasi proyek
    └── docs/
        ├── DATABASE.md
        ├── ARCHITECTURE.md
        └── ...
```


---

## 1. Setup Firebase

### 1.1 Buat Project Firebase
1. Buka [console.firebase.google.com](https://console.firebase.google.com)
2. Klik **Add Project** → isi nama: `carimakan-prod`
3. Aktifkan **Google Analytics** (opsional)

### 1.2 Aktifkan Services
Di Firebase Console, aktifkan:
- **Authentication** → Sign-in method → Email/Password
- **Firestore Database** → Start in production mode
- **Storage** → Start in production mode
- **Cloud Messaging** (FCM) — otomatis aktif

### 1.3 Download Config
- Untuk Flutter: Download `google-services.json` (Android) dan `GoogleService-Info.plist` (iOS)
- Untuk Backend: Download **Service Account Key** JSON dari Project Settings → Service Accounts

### 1.4 Install Firebase CLI
```bash
npm install -g firebase-tools
firebase login
firebase init
```

---

## 2. Setup Backend (Node.js + Express)

```bash
cd apps/backend
npm install
```

### Environment Variables
Buat file `.env` di `apps/backend/`:

```env
PORT=3000
NODE_ENV=development

# Firebase Admin SDK
FIREBASE_PROJECT_ID=carimakan-prod
FIREBASE_CLIENT_EMAIL=firebase-adminsdk@carimakan-prod.iam.gserviceaccount.com
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"

# JWT
JWT_SECRET=your_super_secret_jwt_key_here
JWT_EXPIRES_IN=7d

# Midtrans
MIDTRANS_SERVER_KEY=SB-Mid-server-xxxxxxxxxxxx
MIDTRANS_CLIENT_KEY=SB-Mid-client-xxxxxxxxxxxx
MIDTRANS_IS_PRODUCTION=false

# Google Maps
GOOGLE_MAPS_API_KEY=AIzaSyXXXXXXXXXXXXXXXXXXXXXXX

# OneSignal (optional)
ONESIGNAL_APP_ID=your_onesignal_app_id
ONESIGNAL_API_KEY=your_onesignal_api_key
```

### Menjalankan Backend
```bash
# Development (dengan hot-reload)
npm run dev

# Production
npm run build
npm start
```

Backend berjalan di: `http://localhost:3000`

---

## 3. Setup Mobile App (Flutter)

```bash
cd apps/mobile
flutter pub get
```

### Konfigurasi Firebase untuk Flutter
1. Copy `google-services.json` ke `android/app/`
2. Copy `GoogleService-Info.plist` ke `ios/Runner/`
3. Jalankan: `flutterfire configure` (jika menggunakan FlutterFire CLI)

### Environment / Config
Buat file `lib/config/env.dart`:

```dart
class Env {
  static const String baseApiUrl = 'http://localhost:3000/v1'; // dev
  static const String googleMapsApiKey = 'AIzaSyXXXXXXXXXX';
  static const String midtransClientKey = 'SB-Mid-client-XXXXXXX';
}
```

### Menjalankan Mobile App
```bash
# Cek device yang terhubung
flutter devices

# Run di emulator/device
flutter run

# Build APK
flutter build apk --release

# Build iOS
flutter build ios --release
```

---

## 4. Setup Web Admin (`carimakan_admin`)

```bash
cd carimakan_admin
npm install
```

### Environment Variables
Buat file `.env.local` di root `carimakan_admin/`:

```env
# Firebase Admin SDK (dari Service Account Key)
FIREBASE_PROJECT_ID=carimakan-prod
FIREBASE_CLIENT_EMAIL=firebase-adminsdk@carimakan-prod.iam.gserviceaccount.com
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
```

> **Catatan:** Admin dashboard tidak menggunakan Firebase Client SDK — hanya Firebase **Admin SDK** via Server Actions.

### Seed Database
Setelah `.env.local` dan `serviceAccountKey.json` siap:

```bash
cd setupdb
node firestore-setup.js
```

Ini akan membuat semua koleksi: `users`, `restaurants`, `menus`, `meja`, `badges`, `resto_badges`, `tag_kategori`, `review_tags`, `promo_vouchers`, `orders`, `order_review_tags`.

### Menjalankan Web Admin
```bash
# Development
npm run dev
# Akses di: http://localhost:3000

# Build production
npm run build
npm start
```


---

## 5. Setup Firestore Security Rules

Di Firebase Console → Firestore → Rules, gunakan rules berikut sebagai starting point:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Users: hanya bisa baca/edit data sendiri
    match /users/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }

    // Restaurants: publik bisa baca, hanya owner yang bisa edit
    match /restaurants/{restoId} {
      allow read: if true;
      allow write: if request.auth != null
        && get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'owner';
    }

    // Queues: real-time, bisa dibaca semua yang login
    match /queues/{queueId} {
      allow read: if request.auth != null;
      allow write: if request.auth != null;
    }

    // Orders: hanya customer yang bersangkutan atau owner resto
    match /orders/{orderId} {
      allow read, write: if request.auth != null;
    }
  }
}
```

---

## 6. Setup Midtrans (Payment Gateway)

1. Daftar akun di [midtrans.com](https://midtrans.com)
2. Gunakan **Sandbox** untuk development
3. Ambil **Server Key** dan **Client Key** dari Dashboard Midtrans
4. Pasang di `.env` backend (lihat langkah 2)
5. Konfigurasi **Webhook URL** di Midtrans Dashboard:
   - `https://api.carimakan.app/v1/payments/webhook`

---

## 7. Akun & Data Default

Setelah setup Firebase selesai, jalankan seeder:

```bash
cd carimakan_admin/setupdb
node firestore-setup.js
```

Ini akan membuat akun default di Firestore collection `users`:

```json
{
  "id": "admin_001",
  "nama": "Admin CariMakan",
  "email": "admin@carimakan.app",
  "password": "password123",
  "role": "admin",
  "status": "aktif"
}
```

> **Login Admin:** Buka `http://localhost:3000`, gunakan email `admin@carimakan.app` dan password `password123`.
> **Keamanan Production:** Ganti password dengan nilai yang kuat dan implementasikan bcrypt hashing sebelum deploy ke production.


---

## Checklist Setup Awal

- [ ] Firebase project dibuat & services diaktifkan
- [ ] `google-services.json` dan `GoogleService-Info.plist` sudah di tempat yang benar
- [ ] Service Account Key sudah di-setup di backend
- [ ] `.env` backend sudah lengkap
- [ ] `.env.local` web-admin sudah lengkap
- [ ] `flutter pub get` berhasil
- [ ] `npm install` di backend & web-admin berhasil
- [ ] Backend bisa jalan di `localhost:3000`
- [ ] Mobile app bisa connect ke backend
- [ ] Midtrans sandbox sudah dikonfigurasi
- [ ] Firestore Security Rules sudah dipasang
- [ ] Akun admin sudah di-seed