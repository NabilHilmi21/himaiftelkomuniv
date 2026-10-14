# Dokumentasi Perencanaan Teknis HIMA IF CMS (Technical Planning Index)

| Field | Detail |
|---|---|
| **Sistem** | HIMA IF CMS (Content Management & Recruitment System) |
| **Versi Dokumen** | 2.0.0 (Lengkap, Selaras & Terverifikasi Arsitektur) |
| **Tanggal Pembuatan** | 14 Oktober 2026 |
| **Status** | Approved Technical Planning Baseline |
| **Organisasi** | Himpunan Mahasiswa Informatika (HIMA IF) Telkom University |
| **Pengelola & Perancang** | Staff Ahli Divisi Media Informasi & Kreatif (MIK), Program Kerja Informasite (Informatics Website) |
| **Tech Stack Utama** | Next.js 15+, Tailwind CSS, Python FastAPI, PostgreSQL 16, 3x Segregated Redis 7, S3-compatible Storage |
| **Lokasi Berkas** | `/new_md/` |

---

## Riwayat Revisi (Revision History)

| Versi | Tanggal | Penulis | Ringkasan Perubahan | Alasan |
|---|---|---|---|---|
| 1.0.0 | 13 Oktober 2026 | Staff Ahli Divisi MIK (Proker Informasite) | Draf awal indeks dokumentasi perencanaan teknis. | Inisiasi katalog dokumen perencanaan. |
| 2.0.0 | 14 Oktober 2026 | Staff Ahli Divisi MIK & Antigravity Auditor | Sinkronisasi versi seluruh 11 berkas perencanaan, pembaruan ringkasan skema 18 tabel (peleburan evaluasi ke baris booking), penegasan kriptografi asimetris ES256 & Argon2id OWASP, segregasi 3 instance Redis, invalidasi cache $O(1)$, pustaka sanitasi `nh3`, dan pemeliharaan referensi `/fixed_md`. | Harmonisasi master dokumen setelah audit komprehensif fase 1-6. |

---

## 1. Daftar Dokumen Perencanaan Teknis

Seluruh dokumen perencanaan sistem telah disusun secara terpisah per topik sesuai dengan format, standar, dan gaya penulisan pada folder `/fixed_md`:

| No | Nama File | Versi | Topik Utama | Ringkasan Isi & Cakupan Teknis Terkini |
|---|---|:---:|---|---|
| **01** | [`SYSTEM_ARCHITECTURE.md`](./SYSTEM_ARCHITECTURE.md) | **2.0.0** | **Gambaran Umum & Arsitektur Sistem** | Arsitektur Decoupled Next.js & FastAPI satu origin (`himaif.id`), diagram C4, urutan penguncian global anti-deadlock (`ORDER BY id ASC FOR UPDATE`), segregasi 3 instance Redis, dan Version Tag Invalidation $O(1)$. |
| **02** | [`DATABASE_SCHEMA.md`](./DATABASE_SCHEMA.md) | **2.0.0** | **Skema Database Relasional** | Skema 18 tabel PostgreSQL (evaluasi dilebur ke baris `interview_bookings`), tipe data `TIMESTAMPTZ`, indeks GIN/B-tree, enkripsi aplikasi AES-256 untuk PIN pelamar, dan constraint kapasitas slot. |
| **03** | [`API_DESIGN.md`](./API_DESIGN.md) | **2.0.0** | **Desain API FastAPI & Kontrak Data** | Kontrak endpoint RESTful `/api/v1` dengan path parameter terstandarisasi `{param}`, endpoint ganti password wajib, reset PIN pelamar, reschedule publik H-1, evaluasi per booking, dan ekspor RESTful tanpa PIN. |
| **04** | [`AUTH_AND_RBAC.md`](./AUTH_AND_RBAC.md) | **2.0.0** | **Autentikasi & Otorisasi RBAC** | Identitas *pseudo-account*, hashing Argon2id OWASP, dual JWT asimetris (ES256), rotasi refresh token dengan deteksi reuse `jti`, guard `must_change_password` (min 12 karakter), dan perbaikan IDOR multi-divisi. |
| **05** | [`BLOG_MODULE.md`](./BLOG_MODULE.md) | **2.0.0** | **Modul Blog CMS & Block Editor** | Editor blok TipTap setara Gutenberg, sanitasi HTML server-side via pustaka Rust `nh3`, siklus hidup artikel, Celery Beat scheduler dengan invalidasi Version Tag $O(1)$, dan modul evaluasi SEO real-time. |
| **06** | [`FORM_AND_INTERVIEW_MODULE.md`](./FORM_AND_INTERVIEW_MODULE.md) | **2.0.0** | **Modul Formulir Dinamis & Interview** | Form builder tanpa batas input, auto-provisioning akun divisi, kewajiban memesan 2 slot untuk 2 divisi, reschedule mandiri publik (H-1 + Turnstile), workspace interview per booking, dan pengingat WhatsApp manual H-30 menit. |
| **07** | [`IMAGE_OPTIMIZATION_PERFORMANCE.md`](./IMAGE_OPTIMIZATION_PERFORMANCE.md) | **1.1.0** | **Optimasi Gambar & Performa Sistem** | Standar dimensi thumbnail (16:9), pipeline kompresi Pillow dengan proteksi anti-decompression bomb (`MAX_IMAGE_PIXELS = 16_777_216`), LQIP base64 (CLS = 0), topologi presigned URL satu origin, dan Lenis smooth scroll. |
| **08** | [`PROJECT_STRUCTURE.md`](./PROJECT_STRUCTURE.md) | **1.1.0** | **Struktur Folder Frontend & Backend** | Tata letak monorepo terpadu, struktur Next.js App Router dengan `src/middleware.ts`, struktur FastAPI dengan modul enkripsi AES, task retensi orphan, script database seed, dan template `.env.example` 3 instance Redis. |
| **09** | [`IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md) | **1.1.0** | **Rencana Implementasi Bertahap** | Roadmap 6 fase sprint-based, alokasi prioritas P0/P1/P2, matriks dependensi antar modul, deliverables teknis mutakhir per fase, dan kriteria penerimaan terukur (*quality gates*). |
| **10** | [`RISKS_AND_ASSUMPTIONS.md`](./RISKS_AND_ASSUMPTIONS.md) | **1.1.0** | **Risiko Teknis & Asumsi Desain** | Log 8 asumsi rekayasa terverifikasi, mitigasi 10 risiko teknis utama (anti-deadlock, IDOR, dekompresi bom Pillow, slot thrashing, IP spoofing, sanitasi `nh3`), dan formalisasi kebijakan retensi data. |
| **11** | [`INDEX.md`](./INDEX.md) | **2.0.0** | **Katalog & Master Index Dokumentasi** | Indeks utama navigasi pengembang, ringkasan hak akses operasional antar peran, status revisi, dan pemeliharaan referensi `/fixed_md`. |

---

## 2. Ringkasan Hak Akses Operasional (Role Scopes)

| Role Enum | Identitas Akun Login | Lingkup Tanggung Jawab Utama | Hak Akses PIN Pelamar | Mekanisme Reset Password |
|---|---|---|---|---|
| `SUPER_ADMIN` | `admin@informasite.himaif` | Dipegang oleh PIC Proker Informasite (Staff Ahli Divisi MIK). Mengelola seluruh sistem (semua blog, semua formulir acara, semua akun, dan audit log global). | Dapat melihat & mereset PIN pelamar mana pun (tercatat audit log). | Dapat mereset password akun mana pun. Tersedia CLI darurat server: `python -m app.cli reset-superadmin`. |
| `FORM_MASTER` | `admin@namakepanitiaan.himaif`<br>*(cth: `admin@pemiluif.himaif`)* | Mengelola formulir kepanitiaannya sendiri, menambah divisi, melihat seluruh pendaftar, dan rekap interview seluruh divisi miliknya. | Dapat melihat & mereset PIN pelamar pada form miliknya (tercatat audit log). | Direset oleh Super Admin. Master Formulir dapat mereset akun divisi (`DIVISION_STAFF`) di bawah formulirnya. |
| `DIVISION_STAFF` | `...@divisi.namakepanitiaan`<br>*(cth: `acara@divisi.pemiluif`)* | Mengelola pendaftar divisinya (Divisi 1 atau 2), jadwal interview divisi, pertanyaan kustom, dan lembar kerja penilaian kandidat. | **Dilarang keras** melihat atau mereset PIN pelamar (selalu di-*redact*). | Direset oleh Master Formulir kepanitiaannya atau Super Admin. Dapat mengganti password sendiri jika tahu password lama. |

---

## 3. Panduan Navigasi Dokumen untuk Pengembang

- **Untuk Arsitek & DevOps**: Mulai dari [`SYSTEM_ARCHITECTURE.md`](./SYSTEM_ARCHITECTURE.md) dan [`PROJECT_STRUCTURE.md`](./PROJECT_STRUCTURE.md).
- **Untuk Database Engineer**: Pelajari [`DATABASE_SCHEMA.md`](./DATABASE_SCHEMA.md).
- **Untuk Backend Engineer (FastAPI)**: Rujuk [`API_DESIGN.md`](./API_DESIGN.md), [`AUTH_AND_RBAC.md`](./AUTH_AND_RBAC.md), [`BLOG_MODULE.md`](./BLOG_MODULE.md), dan [`FORM_AND_INTERVIEW_MODULE.md`](./FORM_AND_INTERVIEW_MODULE.md).
- **Untuk Frontend Engineer (Next.js)**: Rujuk [`BLOG_MODULE.md`](./BLOG_MODULE.md), [`IMAGE_OPTIMIZATION_PERFORMANCE.md`](./IMAGE_OPTIMIZATION_PERFORMANCE.md), dan [`PROJECT_STRUCTURE.md`](./PROJECT_STRUCTURE.md).
- **Untuk Project Lead & QA**: Gunakan [`IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md) dan [`RISKS_AND_ASSUMPTIONS.md`](./RISKS_AND_ASSUMPTIONS.md).
