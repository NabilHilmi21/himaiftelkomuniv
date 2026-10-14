# API Architecture & Endpoint Specification — HIMA IF CMS

| Field | Detail |
|---|---|
| **System** | HIMA IF CMS |
| **Document Version** | 2.0.0 |
| **Date** | 14 October 2026 |
| **Status** | Approved Specification |
| **Framework** | Python FastAPI (ASGI) + Pydantic v2 |
| **Base URL** | `/api/v1` |
| **Auth Protocol** | Asymmetric ES256 JWT (Access Token in-memory) + Signed HttpOnly Cookie (Refresh Token) |

---

## Riwayat Revisi (Revision History)

| Versi | Tanggal | Penulis | Ringkasan Perubahan | Alasan |
|---|---|---|---|---|
| 1.0.0 | 13 Oktober 2026 | Staff Ahli Divisi MIK (Proker Informasite) | Rilis draf awal spesifikasi arsitektur API dan endpoint. | Inisiasi baseline arsitektur API backend. |
| 2.0.0 | 14 Oktober 2026 | Staff Ahli Divisi MIK & Antigravity Auditor | Standardisasi path parameter `{param}`, penambahan endpoint rotasi password wajib & PIN reset, integrasi batas H-1 Turnstile reschedule publik, peleburan evaluasi berbasis `booking_id`, penegasan audit log terenkripsi & scope check ketat, format ekspor RESTful tanpa ekstensi file pada path, spesifikasi baseline Argon2id OWASP dan proteksi rate limiting. | Audit keamanan & keselarasan arsitektur relasional `DATABASE_SCHEMA.md` v2.0.0. |

---

## 1. Architectural Conventions & Standards

### 1.1 Envelope Respons Standar
Seluruh respons API menggunakan format JSON terstandarisasi untuk mempermudah konsumsi di frontend Next.js:

```json
{
  "success": true,
  "data": {},
  "message": "Operasi berhasil diselesaikan",
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 142,
    "total_pages": 8
  }
}
```

Format penanganan error (RFC 7807 compatible):
```json
{
  "success": false,
  "error_code": "RESOURCE_CONFLICT",
  "message": "Slot interview telah terisi oleh pendaftar lain",
  "detail": [
    {
      "loc": ["body", "primary_slot_id"],
      "msg": "Slot capacity reached",
      "type": "capacity_exceeded"
    }
  ]
}
```

Daftar Kode Error Aplikasi Baku:
- `VALIDATION_ERROR` (422): Input body/query gagal melewati skema Pydantic.
- `AUTHENTICATION_FAILED` (401): Kredensial akun tidak valid atau token kedaluwarsa.
- `PASSWORD_CHANGE_REQUIRED` (403): Akun ditandai `must_change_password=true`, akses ke endpoint non-allowlist ditolak.
- `PERMISSION_DENIED` (403): Otorisasi RBAC atau pembatasan scope divisi gagal.
- `RESOURCE_NOT_FOUND` (404): Entitas tidak ditemukan dalam basis data.
- `RESOURCE_CONFLICT` (409): Tabrakan kuota slot, deadlock, atau pelanggaran versi optimistic locking.
- `RATE_LIMIT_EXCEEDED` (429): Ambang batas request terlampaui.
- `INTERNAL_SERVER_ERROR` (500): Galat internal server (pesan detail teknis tidak dibocorkan ke klien).

### 1.2 Hierarki Otorisasi & Dependency Injection FastAPI
FastAPI mengimplementasikan dependency injection bertingkat untuk membatasi akses:
- `get_current_user`: Memverifikasi JWT asimetris ES256 (public key), memvalidasi status aktif akun, dan memeriksa `auth_epoch` di Redis instance 3 (Session).
- `enforce_password_change_guard`: Memeriksa klaim `must_change_password`. Jika bernilai `true`, request hanya diizinkan menuju allowlist:
  1. `POST /api/v1/auth/refresh`
  2. `POST /api/v1/auth/change-password`
  3. `POST /api/v1/auth/logout`
  4. `GET /api/v1/auth/me`
  Seluruh endpoint lainnya akan otomatis mengembalikan HTTP 403 `PASSWORD_CHANGE_REQUIRED`.
- `require_super_admin`: Hanya boleh diakses oleh akun dengan role `SUPER_ADMIN`.
- `require_form_master`: Mengizinkan `SUPER_ADMIN` atau `FORM_MASTER` yang memiliki kepemilikan atas `form_id` terkait.
- `require_division_staff`: Mengizinkan `SUPER_ADMIN`, `FORM_MASTER` terkait, atau `DIVISION_STAFF` yang memiliki `division_id` terkait (memeriksa `primary_division_id` maupun `secondary_division_id` pada pelamar).
- `verify_submission_credentials`: Khusus pendaftar publik yang memverifikasi `submission_code`, `applicant_nim`, dan `tracking_pin` 6-digit terenkripsi dengan perbandingan *constant-time*.

---

## 2. Master Endpoint Inventory

### 2.1 Modul Autentikasi & Akun (`/api/v1/auth`, `/api/v1/users`)

| Method | Endpoint | Deskripsi | Request Body / Query | Role yang Diizinkan |
|---|---|---|---|---|
| `POST` | `/auth/login` | Login menggunakan pseudo-email & password | `{ email, password }` | Public (Rate Limited) |
| `POST` | `/auth/refresh` | Perpanjangan Access Token dengan Refresh Token | Cookie HttpOnly: `refresh_token` | Public (Cookie Valid) |
| `POST` | `/auth/logout` | Revokasi sesi dan pembersihan token di Redis | Header Authorization Bearer | Authenticated |
| `GET` | `/auth/me` | Mengambil profil user aktif, role, dan scope ID | - | Authenticated |
| `POST` | `/auth/change-password` | Mengubah password akun sendiri (wajib saat awal aktivasi) | `{ current_password, new_password }` | Authenticated (Rate Limited) |
| `GET` | `/users` | List seluruh akun terdaftar dalam sistem | Query: `role, search, is_active, page, limit` | `SUPER_ADMIN` |
| `POST` | `/users/reset-password` | Reset password akun bawahan secara hierarkis | `{ target_user_id, new_password }` | `SUPER_ADMIN`, `FORM_MASTER` |
| `PATCH` | `/users/{user_id}/status` | Mengaktifkan atau menonaktifkan akun | `{ is_active: boolean }` | `SUPER_ADMIN` |

#### Spesifikasi Detail Autentikasi:
1. **POST `/auth/login`**:
   - Menerapkan Argon2id baseline OWASP ($m=19456\text{ KiB}, t=2, p=1$).
   - Proteksi brute-force bertingkat via Redis: Rate limit 5 kegagalan per 15 menit per kombinasi IP dan email identifier. Jika terlampaui, akun terkunci sementara (lockout) selama 15 menit.
   - Respons sukses menghasilkan Access Token (disimpan in-memory oleh Zustand di Next.js) dan menyematkan Refresh Token bertanda tangan ES256 pada HttpOnly cookie (`Secure`, `SameSite=Strict`, `Path=/`, `Max-Age=604800`).
2. **POST `/auth/change-password`**:
   - Memvalidasi `current_password` via Argon2id.
   - Memastikan `new_password` memiliki panjang minimal 12 karakter dan berbeda dari `current_password`.
   - Menghash `new_password` dengan Argon2id, mengubah kolom `must_change_password = false`, dan menaikkan `auth_epoch` di DB/Redis untuk merevokasi seluruh token lama.
   - Menerbitkan token baru dengan klaim `must_change_password: false`.
   - Mencatat event audit `CHANGE_PASSWORD` ke `auth_audit_logs`.
3. **POST `/users/reset-password`**:
   - `SUPER_ADMIN` dapat mereset akun mana pun.
   - `FORM_MASTER` **hanya** dapat mereset akun `DIVISION_STAFF` di bawah formulir miliknya.
   - Mereset kolom `must_change_password = true` dan mencatat event audit `RESET_PASSWORD`.

---

### 2.2 Modul Blog CMS (`/api/v1/blog`)

| Method | Endpoint | Deskripsi | Request Body / Query | Role yang Diizinkan |
|---|---|---|---|---|
| `GET` | `/blog/posts` | Mengambil daftar artikel terbit (publik, paginasi, filter) | Query: `category_slug, search, page, limit` | Public (Cached) |
| `GET` | `/blog/posts/{slug}` | Mengambil detail artikel publik berdasarkan slug | Path: `slug` | Public (Cached) |
| `GET` | `/blog/categories` | Mengambil seluruh kategori blog yang aktif | - | Public (Cached) |
| `GET` | `/blog/admin/posts` | Mengambil semua artikel (Draft, Scheduled, Published) | Query: `status, category_id, page, limit` | `SUPER_ADMIN` |
| `POST` | `/blog/admin/posts` | Membuat draft artikel baru dengan block JSONB | `{ title, slug, category_id, content_blocks, excerpt, seo_title, meta_description, seo_focus_keyword }` | `SUPER_ADMIN` |
| `GET` | `/blog/admin/posts/{id}` | Mengambil detail artikel lengkap untuk editor | Path: `id` | `SUPER_ADMIN` |
| `PUT` | `/blog/admin/posts/{id}` | Memperbarui konten blok dan metadata artikel | Request body artikel lengkap | `SUPER_ADMIN` |
| `PATCH` | `/blog/admin/posts/{id}/status` | Mengubah status tayang (Publish, Schedule, Archive) | `{ status: "PUBLISHED" \| "SCHEDULED" \| "ARCHIVED", scheduled_at?: string }` | `SUPER_ADMIN` |
| `DELETE` | `/blog/admin/posts/{id}` | Menghapus artikel blog (soft delete / purge) | Path: `id` | `SUPER_ADMIN` |
| `POST` | `/blog/admin/seo-analyze` | Menjalankan analisis SEO real-time pada draft teks | `{ title, slug, content_blocks, focus_keyword, seo_title, meta_description }` | `SUPER_ADMIN` |
| `GET` | `/blog/admin/posts/{id}/revisions` | Mengambil daftar riwayat revisi artikel | Path: `id` | `SUPER_ADMIN` |
| `POST` | `/blog/admin/posts/{id}/rollback` | Mengembalikan artikel ke snapshot revisi tertentu | `{ revision_id }` | `SUPER_ADMIN` |
| `POST` | `/blog/admin/categories` | Membuat kategori blog baru | `{ name, slug, description, color_hex }` | `SUPER_ADMIN` |
| `PUT` | `/blog/admin/categories/{id}` | Memperbarui nama/warna kategori blog | `{ name, slug, description, color_hex }` | `SUPER_ADMIN` |
| `DELETE` | `/blog/admin/categories/{id}` | Menghapus kategori blog (jika tidak terikat post) | Path: `id` | `SUPER_ADMIN` |

*Catatan Sanitasi & Cache:*
- Seluruh konten HTML pada blok teks (`customHtml`) disanitasi menggunakan pustaka Rust-based `nh3` di backend sebelum disimpan ke JSONB.
- Mutasi status atau konten artikel secara otomatis mengeksekusi *Version Tag Invalidation* $O(1)$ (`INCR cms:blog:version:list`) dan menghapus cache key spesifik artikel (`DEL cms:blog:post:{slug}`).

---

### 2.3 Modul Media & Storage (`/api/v1/media`)

| Method | Endpoint | Deskripsi | Request Body / Query | Role yang Diizinkan |
|---|---|---|---|---|
| `POST` | `/media/upload-image` | Mengunggah gambar (thumbnail/content) -> auto-convert WebP/AVIF | `multipart/form-data` (file) | `SUPER_ADMIN` |
| `POST` | `/media/upload-attachment` | Mengunggah lampiran pendaftar publik (CV/Portofolio) | `multipart/form-data` (file) | Public (Turnstile & Rate Limited) |
| `GET` | `/media/{id}/download-url` | Mendapatkan pre-signed URL untuk berkas terproteksi | Path: `id` | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisi terkait) |

*Aturan Akses & Audit Berkas:*
- Lampiran publik pendaftar disimpan pada bucket private vault (`is_protected = true`).
- Endpoint `GET /media/{id}/download-url` memvalidasi relasi pendaftar terhadap akun yang merequest (mencegah IDOR). Setiap penerbitan presigned download URL dicatat ke `auth_audit_logs` dengan aksi `DOWNLOAD_APPLICANT_CV`. URL presigned berlaku selama 5 menit.

---

### 2.4 Modul Dynamic Form Builder (`/api/v1/forms`)

| Method | Endpoint | Deskripsi | Request Body / Query | Role yang Diizinkan |
|---|---|---|---|---|
| `GET` | `/forms/{slug}` | Mengambil schema formulir aktif untuk dirender di publik | Path: `slug` | Public (Cached) |
| `GET` | `/forms` | Mengambil daftar seluruh formulir kepanitiaan | Query: `status, page, limit` | `SUPER_ADMIN` |
| `POST` | `/forms` | Membuat formulir baru + Akun Master Formulir otomatis | `{ title, slug, description, starts_at, ends_at, master_password }` | `SUPER_ADMIN` |
| `GET` | `/forms/{id}/details` | Mengambil konfigurasi detail formulir | Path: `id` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `PUT` | `/forms/{id}` | Memperbarui informasi umum & status formulir | `{ title, description, starts_at, ends_at, status, settings }` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `GET` | `/forms/{id}/fields` | Mengambil seluruh daftar field custom formulir | Path: `id` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `POST` | `/forms/{id}/fields` | Menambahkan field custom baru ke formulir | `{ field_label, field_name, field_type, is_required, placeholder, validation_rules, options, division_id }` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `PUT` | `/forms/{id}/fields/{field_id}` | Memperbarui konfigurasi field custom | Update field schema | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `DELETE` | `/forms/{id}/fields/{field_id}` | Menghapus field custom formulir | Path: `id, field_id` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `POST` | `/forms/{id}/fields/reorder` | Mengatur ulang urutan tampil (*order_index*) field | `{ field_ids: [uuid] }` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |

---

### 2.5 Modul Manajemen Divisi & Auto-Provisioning (`/api/v1/forms/{form_id}/divisions`)

| Method | Endpoint | Deskripsi | Request Body / Query | Role yang Diizinkan |
|---|---|---|---|---|
| `GET` | `/forms/{form_id}/divisions` | List seluruh divisi dalam formulir | Path: `form_id` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `POST` | `/forms/{form_id}/divisions` | Menambah divisi baru + otomatis generate akun login | `{ name, slug, description, max_quota, custom_password }` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `PUT` | `/forms/{form_id}/divisions/{div_id}` | Memperbarui deskripsi dan kuota divisi | `{ name, description, max_quota }` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `POST` | `/forms/{form_id}/divisions/{div_id}/reset-password` | Mereset password akun divisi secara mandiri | `{ new_password }` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `DELETE` | `/forms/{form_id}/divisions/{div_id}` | Menghapus divisi (jika belum ada pendaftar) | Path: `form_id, div_id` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |

---

### 2.6 Modul Pendaftar & Submisi Publik (`/api/v1/submissions`)

| Method | Endpoint | Deskripsi | Request Body / Query | Role yang Diizinkan |
|---|---|---|---|---|
| `POST` | `/forms/{slug}/submit` | Mengirim pendaftaran + memesan slot interview | Body Pendaftaran (lihat spesifikasi di bawah) | Public (Turnstile & Rate Limited) |
| `POST` | `/submissions/verify-status` | Autentikasi portal pendaftar mandiri | `{ submission_code, applicant_nim, tracking_pin }` | Public (Rate Limited & Constant-Time) |
| `GET` | `/forms/{form_id}/submissions` | Mengambil seluruh pendaftar formulir (sentralisasi) | Query: `division_id, status, search, page, limit` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `GET` | `/divisions/me/submissions` | Mengambil pendaftar khusus divisinya sendiri | Query: `status, search, page, limit` | `DIVISION_STAFF` |
| `GET` | `/submissions/{id}` | Mengambil detail lengkap berkas & data pendaftar | Path: `id` | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisinya) |
| `POST` | `/submissions/{id}/reset-pin` | Mereset `tracking_pin` pelamar secara acak | Path: `id` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `PATCH` | `/submissions/{id}/status` | Mengubah status keputusan akhir pelamar | `{ status, notes }` | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisi terkait) |

#### Spesifikasi Body `POST /forms/{slug}/submit`:
```json
{
  "applicant_name": "Fulan bin Fulan",
  "applicant_nim": "1301210001",
  "applicant_email": "fulan@student.telkomuniversity.ac.id",
  "applicant_phone": "081234567890",
  "primary_division_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "secondary_division_id": "c9a646d3-9c61-4cd7-bf16-444a1eb40277",
  "primary_slot_id": "4a7f2921-b3b3-4f9e-a0e1-067756f8f53a",
  "secondary_slot_id": "7f1e680a-5c21-4d1a-8c34-9d560b411d33",
  "responses": {
    "alasan_mendaftar": "Ingin berkontribusi nyata untuk himpunan",
    "portofolio_url": "https://github.com/fulan"
  },
  "consent_agreed": true,
  "turnstile_token": "0.XXXXX.YYYYY"
}
```
*Aturan Bisnis Submisi:*
- Validasi Turnstile wajib lolos secara *fail-closed*.
- Jika `secondary_division_id` dipilih, maka `secondary_slot_id` **wajib** diisi. Kedua slot harus berasal dari divisi yang bersangkutan, slot tidak boleh sama, dan rentang waktu keduanya tidak boleh tumpang tindih (*overlap*).
- Menjalankan urutan penguncian global anti-deadlock (`SELECT ... FOR UPDATE` terurut `id ASC`).
- Menghasilkan `tracking_pin` 6-digit acak dengan `secrets.randbelow(1000000)`, dienkripsi via format `{key_version}:{ciphertext}` (AES-256-GCM / Fernet) pada level aplikasi sebelum disimpan ke DB.
- PIN hanya ditampilkan **sekali** di layar respons submisi.

#### Aturan Akses & Reset PIN Pelamar:
- **GET `/submissions/{id}`**: Field `tracking_pin` yang didekripsi **hanya** dikembalikan kepada `SUPER_ADMIN` dan `FORM_MASTER` (khusus formulir miliknya). Akun `DIVISION_STAFF` tidak pernah menerima nilai PIN (selalu di-*redact* atau null). Tindakan melihat PIN ini dicatat pada `auth_audit_logs` dengan event `VIEW_APPLICANT_PIN`. Field PIN tidak pernah muncul di endpoint list/tabel maupun file ekspor.
- **POST `/submissions/{id}/reset-pin`**: Hanya dapat dipanggil oleh `SUPER_ADMIN` dan `FORM_MASTER` (khusus form miliknya). Menghasilkan PIN baru 6-digit acak, mengenkripsinya di DB, mencatat event `RESET_APPLICANT_PIN`, dan mengembalikan nilai plaintext baru tersebut satu kali di respons untuk diinformasikan manual oleh panitia kepada mahasiswa via WhatsApp.
- **POST `/submissions/verify-status`**: Dilindungi rate limit ketat (5 kegagalan per 15 menit per IP dan NIM). Menggunakan `secrets.compare_digest` (*constant-time*) untuk mencegah timing attack, dan mengembalikan pesan kesalahan identik jika kode pendaftaran, NIM, atau PIN keliru.

---

### 2.7 Modul Penjadwalan & Slot Interview (`/api/v1/interviews`)

| Method | Endpoint | Deskripsi | Request Body / Query | Role yang Diizinkan |
|---|---|---|---|---|
| `GET` | `/interviews/public/{form_slug}/available-slots` | Slot tersedia untuk pendaftar publik | Query: `division_id, date` | Public (Cached pendek) |
| `GET` | `/interviews/config/{division_id}` | Mengambil konfigurasi jadwal interview divisi | Path: `division_id` | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisinya) |
| `POST` | `/interviews/config/{division_id}` | Menyimpan konfigurasi & generate slot otomatis | `{ start_date, end_date, daily_start_time, daily_end_time, slot_duration_minutes, buffer_minutes, quota_per_slot, default_location, active_days }` | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisinya) |
| `GET` | `/interviews/slots/{division_id}` | Mengambil kalender slot lengkap divisi | Query: `start_date, end_date` | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisinya) |
| `PATCH` | `/interviews/slots/{slot_id}/block` | Memblokir slot (cegah booking baru) | `{ is_blocked: true, block_reason: string }` | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisinya) |
| `PATCH` | `/interviews/slots/{slot_id}/unblock` | Membuka kembali slot yang diblokir | - | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisinya) |
| `POST` | `/interviews/public/reschedule` | Reschedule mandiri oleh pelamar publik | `{ submission_code, applicant_nim, tracking_pin, division_id, new_slot_id, turnstile_token }` | Public (H-1 Cut-off, Turnstile, Rate Limited) |
| `POST` | `/interviews/admin/reassign-slot` | Pemindahan jadwal slot oleh panitia | `{ booking_id, new_slot_id, reason }` | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisinya) |

#### Aturan Bisnis Reschedule:
1. **Reschedule Publik (`POST /interviews/public/reschedule`)**:
   - Pelamar diizinkan melakukan reschedule tanpa batas jumlah kali, asalkan dilakukan paling lambat **H-1 (24 jam sebelum `start_time` slot saat ini)**.
   - Slot tujuan baru (`new_slot_id`) juga harus memiliki waktu mulai minimal 24 jam dari waktu request, memiliki kuota tersisa (`booked_count < capacity`), dan tidak bertabrakan dengan jadwal interview divisi pasangannya jika pelamar memiliki 2 booking.
   - Wajib memverifikasi Cloudflare Turnstile token dan kredensial pelamar.
   - Menjalankan urutan penguncian global atomik: kunci baris `interview_bookings` terlebih dahulu, lalu kunci baris slot lama dan slot baru (`ORDER BY id ASC FOR UPDATE`).
   - Mencatat event audit `RESCHEDULE_INTERVIEW` di `auth_audit_logs`.
2. **Reassign oleh Panitia (`POST /interviews/admin/reassign-slot`)**:
   - Ditujukan untuk kasus darurat atau penataan ulang oleh interviewer.
   - **Tidak terikat** batasan H-1 cut-off.
   - Menerapkan penguncian atomik yang sama dan mencatat event audit `RESCHEDULE_INTERVIEW`.

---

### 2.8 Modul Pertanyaan Custom & Evaluasi Interview (`/api/v1/evaluations`)

> **Pembaruan Arsitektur v2.0.0**: Sesuai skema relasional `DATABASE_SCHEMA.md` v2.0.0, lembar evaluasi kini melekat secara granular pada baris `interview_bookings` melalui parameter `{booking_id}` (bukan `submission_id`), sehingga evaluasi Divisi 1 dan Divisi 2 terisolasi secara independen dan mendukung penguncian optimistik (`version`).

| Method | Endpoint | Deskripsi | Request Body / Query | Role yang Diizinkan |
|---|---|---|---|---|
| `GET` | `/evaluations/questions/{division_id}` | Mengambil instrumen pertanyaan interview divisi | Path: `division_id` | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisinya) |
| `POST` | `/evaluations/questions/{division_id}` | Menambah butir pertanyaan custom divisi | `{ question_text, question_type, options, rubric_guide, max_score, weight_percent, order_index }` | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisinya) |
| `PUT` | `/evaluations/questions/{question_id}` | Memperbarui butir pertanyaan custom | Update question payload | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisinya) |
| `DELETE` | `/evaluations/questions/{question_id}` | Menghapus butir pertanyaan interview | Path: `question_id` | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisinya) |
| `GET` | `/evaluations/workspace/{booking_id}` | Membuka lembar kerja interview spesifik booking | Path: `booking_id` | `SUPER_ADMIN`, `FORM_MASTER`, `DIVISION_STAFF` (divisinya) |
| `POST` | `/evaluations/workspace/{booking_id}/draft` | Menyimpan draf penilaian (auto-save ber-versi) | Body Draft Evaluasi (lihat spesifikasi di bawah) | `DIVISION_STAFF` (divisinya), `FORM_MASTER` |
| `POST` | `/evaluations/workspace/{booking_id}/finalize` | Mengunci penilaian final & status kehadiran | Body Finalisasi Evaluasi | `DIVISION_STAFF` (divisinya), `FORM_MASTER` |

#### Spesifikasi Lembar Evaluasi Workspace:
```json
// POST /evaluations/workspace/{booking_id}/draft
{
  "version": 3,
  "attendance_status": "ATTENDED",
  "score": 85.5,
  "notes": "Komunikasi baik, pemahaman teknis sesuai harapan divisi.",
  "answers": [
    {
      "question_id": "2b3c4d5e-6f7a-8b9c-0d1e-2f3a4b5c6d7e",
      "score": 90,
      "answer_text": "Menjelaskan arsitektur microservices dengan terstruktur",
      "notes": "Poin plus pada pemahaman REST"
    }
  ]
}
```
*Kontrol Konkurensi & Optimistic Locking:*
- Request wajib menyertakan kolom `version`. Jika versi yang dikirim tidak sesuai dengan `version` pada baris `interview_bookings`, API mengembalikan HTTP 409 `RESOURCE_CONFLICT` untuk mencegah tertimpanya penilaian oleh penilai lain yang sedang membuka workspace bersamaan.
- Pada Redis instance 3, terdapat advisory lock sementara bertenggat waktu (`interview:lock:{booking_id}`) dengan heartbeat 30 detik untuk menandai penilai aktif (*active interviewer*).

---

### 2.9 Modul Sentralisasi & Ekspor Laporan (`/api/v1/analytics`, `/api/v1/exports`)

| Method | Endpoint | Deskripsi | Request Body / Query | Role yang Diizinkan |
|---|---|---|---|---|
| `GET` | `/analytics/super-admin/overview` | Statistik global (total blog view, acara, pendaftar) | - | `SUPER_ADMIN` |
| `GET` | `/analytics/forms/{form_id}/summary` | Ringkasan pendaftar & progres interview semua divisi | Path: `form_id` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `GET` | `/analytics/forms/{form_id}/submissions/export` | Mengunduh data pendaftar format XLSX/CSV | Query: `format=xlsx\|csv, division_id, status` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `GET` | `/analytics/forms/{form_id}/interview-results/export` | Mengunduh rekap skor & evaluasi interview lengkap | Query: `format=xlsx\|csv, division_id` | `SUPER_ADMIN`, `FORM_MASTER` (milik sendiri) |
| `GET` | `/analytics/divisions/my-results/export` | Mengunduh hasil interview divisi sendiri | Query: `format=xlsx\|csv` | `DIVISION_STAFF` |

*Kepatuhan Keamanan & Privasi Ekspor Data:*
- Ekstensi file dihilangkan dari URL path parameter dan dipindahkan ke query parameter `format` (`xlsx` atau `csv`) sesuai kaidah RESTful modern.
- Kolom sensitif `tracking_pin` **tidak pernah diikutsertakan** dalam seluruh file ekspor laporan!
- Setiap aktivitas ekspor data pendaftar secara otomatis dicatat pada `auth_audit_logs` dengan event `EXPORT_SUBMISSIONS`, merekam `user_id`, `form_id`, filter yang diterapkan, dan alamat IP pemohon.
