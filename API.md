# API Documentation - CariMakan

## Base URL


## Endpoints

### 1. Get Daftar Restoran
GET /restaurants

Response:
- id
- name
- status_antrian (buka/tutup)
- jumlah_antrian
- estimasi_waktu

---

### 2. Get Detail Antrian Restoran
GET /restaurants/:id/queue

Response:
- restaurant_id
- current_queue
- max_queue
- is_open
- estimated_time

---

### 3. Ambil Antrian
POST /queue

Request:
- user_id
- restaurant_id

Response:
- queue_number
- status

---

### 4. Update Status Antrian (Dipanggil)
PUT /queue/:id/call

---

### 5. Selesaikan Antrian
PUT /queue/:id/complete

---

### 6. Update Status Restoran
PUT /restaurants/:id/status

Request:
- is_open (true/false)

---

## Notes
- Semua data antrian real-time (Firestore)
- Validasi:
  - Antrian harus dibuka
  - Antrian tidak boleh penuh