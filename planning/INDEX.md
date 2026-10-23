# Dokumentasi Perencanaan Teknis HIMA IF CMS (Technical Planning Index)

| Field | Detail |
|---|---|
| **Sistem** | HIMA IF CMS (Modern WordPress-like Blog Engine) |
| **Versi Dokumen** | 3.0.0 (Lengkap, Selaras, Nol Ambiguitas & Terverifikasi Arsitektur) |
| **Tanggal Pembuatan** | 23 Oktober 2026 |
| **Status** | Approved Technical Planning Baseline |
| **Organisasi** | Himpunan Mahasiswa Informatika (HIMA IF) Telkom University |
| **Pengelola & Perancang** | Staff Ahli Divisi Media Informasi & Kreatif (MIK), Program Kerja Informasite (Informatics Website) |
| **Tech Stack Utama** | Next.js 15+, Tailwind CSS v3.4.x LTS, TipTap 2.x, Python FastAPI, PostgreSQL 16, Redis 7 (Single Instance), Cloudflare R2 Storage |
| **Lokasi Berkas** | `/planning/` (10 Berkas Master) |

---

## Riwayat Revisi (Revision History)

| Versi | Tanggal | Penulis / Reviewer | Bagian yang Diubah | Rincian Perubahan (Sebelum vs Sesudah) | Alasan Perubahan |
|---|---|---|---|---|---|
| 1.0.0 | 2026-10-13 | Staff Ahli Divisi MIK | Dokumen Baru | Draf awal indeks dokumentasi perencanaan teknis. | Inisiasi katalog dokumen perencanaan. |
| 2.0.0 | 2026-10-14 | Staff Ahli MIK & Auditor | Sinkronisasi Versi | Pembaruan ringkasan skema 18 tabel, ES256, segregasi 3 Redis, sanitasi `nh3`. | Harmonisasi master dokumen v2.0.0. |
| 2.1.0 | 2026-10-14 | Staff Ahli MIK & Implementer | Keputusan Tahap 0 | Grace period refresh 10s, auth_epoch Cache-Aside, 3 revisi Alembic. | Keputusan implementasi Fase 1. |
| 2.2.0 | 2026-10-14 | Staff Ahli MIK & Architect | Status Nol Ambiguitas | Penetapan Storage R2, retensi data UU PDP, penutupan celah BOLA, sinkronisasi 11 dokumen. | Eliminasi total ambiguitas & placeholder. |
| 3.0.0 | 2026-10-23 | Staff Ahli MIK & Antigravity Lead | Seluruh Dokumen | **Sebelum:** Katalog 11 berkas dual-engine (mencakup formulir dinamis & interview booking massal).<br>**Sesudah:** Katalog 10 berkas master murni Blog CMS modern sekelas WordPress (`FORM_AND_INTERVIEW_MODULE.md` dihapus permanen), pemutakhiran seluruh dokumen ke v3.0.0 dan v2.0.0, integrasi arsitektur deployment bertahap (Hostinger VPS $\to$ VPS Kampus Tel-U), penyatuan 1 instance Redis, TipTap AST JSONB + Rust `nh3`, dan Alur Mandiri Author. | Pivot arsitektur sistem menjadi WordPress-like Blog CMS murni sesuai keputusan organisasi. |

---

## 1. Daftar Dokumen Perencanaan Teknis Master (10 Berkas)

Seluruh berkas perencanaan teknis telah dimutakhirkan, disinkronkan, dan bebas dari ambiguitas atau placeholder:

| No | Nama File | Versi | Topik Utama | Ringkasan Isi & Cakupan Teknis Terkini |
|---|---|:---:|---|---|
| **01** | [`SYSTEM_ARCHITECTURE.md`](./SYSTEM_ARCHITECTURE.md) | **3.0.0** | **Arsitektur Sistem & Ingress** | Arsitektur Decoupled Next.js 15 & FastAPI satu origin (`himaif.telkomuniversity.ac.id`), eliminasi Celery & 3 Redis (diganti 1 Redis ramping), Nginx local proxy-cache 5GB SSD untuk Cloudflare R2, dan strategi deployment bertahap Hostinger $\to$ Kampus dengan 301 Permanent Redirect. |
| **02** | [`DATABASE_SCHEMA.md`](./DATABASE_SCHEMA.md) | **3.0.0** | **Skema Database Relasional** | Skema 7 tabel murni blog (`users`, `sessions`, `auth_audit_logs`, `blog_categories`, `blog_tags`, `blog_post_tags`, `blog_posts`, `blog_post_revisions`, `media_assets`), TipTap AST JSONB (`content_json`) & sanitized HTML (`content_html`), dan pembagian 2 revisi migrasi Alembic. |
| **03** | [`API_DESIGN.md`](./API_DESIGN.md) | **3.0.0** | **Desain API & Kontrak Data** | Kontrak endpoint RESTful `/api/v1` terpusat pada Auth, Manajemen User, Taksonomi, Blog Posts (alur mandiri publish & schedule), dan Media Presigned URL S3v4 ke Cloudflare R2 (zero VPS egress). |
| **04** | [`AUTH_AND_RBAC.md`](./AUTH_AND_RBAC.md) | **3.0.0** | **Autentikasi & Otorisasi RBAC** | Matriks hak akses 2 Peran (`SUPER_ADMIN` dan `AUTHOR` alur mandiri), cookie sesi Host-Only `__Host-himaif_session` (anti-cookie tossing antar subdomain kampus), proteksi anti-CSRF eTLD+1, password Argon2id OWASP, dan instant revocation. |
| **05** | [`BLOG_MODULE.md`](./BLOG_MODULE.md) | **3.0.0** | **Modul Blog & TipTap Editor** | Editor visual TipTap setara Gutenberg, sanitasi HTML ganda via Rust `nh3` allowlist, penjadwalan tayang murni query SQL bebas Celery, kontrak relative URI (`/storage/public/{id}.webp`), dan modul real-time SEO scoring. |
| **06** | [`IMAGE_OPTIMIZATION_PERFORMANCE.md`](./IMAGE_OPTIMIZATION_PERFORMANCE.md) | **2.0.0** | **Optimasi Gambar & Performa** | Pipeline direct-to-storage Cloudflare R2 via Presigned URL, Nginx local SSD cache 5 GB (Cache HIT < 10 ms), larangan mutlak berkas `.svg`, batas resolusi 16 MP (anti-decompression bomb), dan target LCP < 1.5s. |
| **07** | [`PROJECT_STRUCTURE.md`](./PROJECT_STRUCTURE.md) | **2.0.0** | **Struktur Repositori Monorepo** | Tata letak monorepo terpadu `/frontend`, `/backend`, dan `/docker`, Next.js App Router dengan TipTap suite components, FastAPI modular clean architecture, dan orkestrasi `docker-compose.prod.yml` ramping (< 1 GB RAM). |
| **08** | [`RISKS_AND_ASSUMPTIONS.md`](./RISKS_AND_ASSUMPTIONS.md) | **2.0.0** | **Analisis Risiko & Asumsi Desain** | Log 6 asumsi desain Blog CMS, mitigasi 8 risiko teknis utama (Stored XSS, eksploitasi SVG, dekompresi bom 16 MP, sibling subdomain session hijacking, broken image migrasi domain, dan SEO retention). |
| **09** | [`IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md) | **2.0.0** | **Roadmap Implementasi 1 Bulan** | Roadmap Sprint 4 Minggu terukur, alokasi deliverables mingguan (Minggu 1: Fondasi & Auth, Minggu 2: Editor & Media, Minggu 3: Blog Workflow, Minggu 4: Hardening & Staging Deploy Hostinger), beserta Quality Gates numerik. |
| **10** | [`INDEX.md`](./INDEX.md) | **3.0.0** | **Katalog Master Dokumentasi** | Indeks master navigasi pengembang, ringkasan hak akses 2 peran operasional, status versi seluruh 10 dokumen mutakhir, dan panduan transisi server. |

---

## 2. Ringkasan Hak Akses Operasional (Role Scopes)

| Role Enum | Cakupan Akses (Scope) | Hak Publikasi Artikel | Manajemen Pengguna & Kategori | Akses Media Library |
|---|---|---|---|---|
| `SUPER_ADMIN` | Dipegang oleh PIC Proker Informasite (Staff Ahli Divisi MIK). Seluruh sistem: artikel semua penulis, kategori, tags, akun author, pengaturan website, dan audit logs. | Dapat menerbitkan, menjadwalkan, mengedit, menurunkan (*unpublish*), dan menghapus artikel milik siapa pun. | Akses penuh: tambah/edit/nonaktifkan author, kelola kategori global. | Akses penuh ke seluruh aset media dan log audit. |
| `AUTHOR` | Staf Divisi MIK / Kontributor Artikel HIMA IF. Hanya artikel karyanya sendiri. | **Alur Mandiri (*Direct Publishing*)**: Dapat membuat, mengedit, mempublikasikan langsung (`PUBLISHED`), dan menjadwalkan (`SCHEDULED`) artikel karyanya sendiri. | **Ditolak**: Dilarang mengelola pengguna atau mengubah taksonomi global. | Hanya dapat mengunggah dan menghapus aset gambar miliknya sendiri. |

---

## 3. Panduan Navigasi Dokumen untuk Pengembang

- **Untuk DevOps & Sysadmin**: Rujuk [`SYSTEM_ARCHITECTURE.md`](./SYSTEM_ARCHITECTURE.md) dan [`PROJECT_STRUCTURE.md`](./PROJECT_STRUCTURE.md).
- **Untuk Database Engineer**: Pelajari [`DATABASE_SCHEMA.md`](./DATABASE_SCHEMA.md).
- **Untuk Backend Engineer (FastAPI)**: Rujuk [`API_DESIGN.md`](./API_DESIGN.md), [`AUTH_AND_RBAC.md`](./AUTH_AND_RBAC.md), dan [`BLOG_MODULE.md`](./BLOG_MODULE.md).
- **Untuk Frontend Engineer (Next.js)**: Rujuk [`BLOG_MODULE.md`](./BLOG_MODULE.md), [`IMAGE_OPTIMIZATION_PERFORMANCE.md`](./IMAGE_OPTIMIZATION_PERFORMANCE.md), dan [`PROJECT_STRUCTURE.md`](./PROJECT_STRUCTURE.md).
- **Untuk Project Lead & QA Engineer**: Rujuk [`IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md) dan [`RISKS_AND_ASSUMPTIONS.md`](./RISKS_AND_ASSUMPTIONS.md).
