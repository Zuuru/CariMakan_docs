# Workflow - CariMakan

## 1. Customer Flow
1. Buka aplikasi
2. Pilih restoran
3. Lihat antrian
4. Ambil antrian
5. Menunggu
6. Datang ke restoran

---

## 2. Restoran Flow
1. Login dashboard
2. Buka/tutup antrian
3. Melihat daftar antrian
4. Memanggil antrian
5. Menyelesaikan antrian

---

## 3. System Flow
1. Ambil data dari Firestore
2. Validasi:
   - Antrian dibuka?
   - Antrian penuh?
3. Simpan data antrian
4. Update status
5. Kirim response ke user

---

## Key Logic
- Queue only created if:
  - is_open = true
  - current_queue < max_queue