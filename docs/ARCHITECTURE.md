# System Architecture - CariMakan

## Overview
CariMakan adalah aplikasi mobile untuk melihat dan mengambil antrian restoran secara real-time.

## Components

### 1. Frontend
- Mobile App (Flutter / React Native)
- Menampilkan restoran & antrian

### 2. Backend
- Node.js + Express
- REST API

### 3. Database
- Firebase Firestore (real-time)

---

## Flow Arsitektur

User → Mobile App → API → Firestore → Response → User

Restoran → Dashboard → API → Firestore

---

## Key Concepts
- Real-time queue update
- Stateless API
- Firestore sebagai single source of truth