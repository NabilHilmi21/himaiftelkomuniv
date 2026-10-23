# Struktur Proyek & Panduan Repositori — HIMA IF CMS

| Field | Detail |
|---|---|
| **System** | HIMA IF CMS (Modern WordPress-like Blog Engine) |
| **Document Version** | 2.0.0 |
| **Date** | 2026-10-23 |
| **Status** | Approved Specification |
| **Monorepo Layout** | Decoupled Architecture (`/frontend` + `/backend` + `/docker`) |
| **Frontend Runtime** | Node.js 22 LTS (Next.js 15+ App Router, React 19, TipTap 2.x) |
| **Backend Runtime** | Python 3.12+ (FastAPI + SQLAlchemy 2.0 Async + Rust nh3) |

---

## Riwayat Revisi (Revision History)

| Versi | Tanggal | Penulis / Reviewer | Bagian yang Diubah | Rincian Perubahan (Sebelum vs Sesudah) | Alasan Perubahan |
|---|---|---|---|---|---|
| 1.0.0 | 2026-10-13 | Staff Ahli Divisi MIK | Dokumen Baru | Draf awal struktur repositori monorepo dual-engine. | Inisiasi arsitektur berkas monorepo. |
| 1.1.0 | 14 Oktober 2026 | Staff Ahli MIK & Auditor | Task & Redis | Penambahan cleanup tasks, 3 instance Redis, enkripsi PIN. | Mitigasi audit keamanan v2.0.0. |
| 1.2.0 | 14 Oktober 2026 | Staff Ahli MIK & Architect | Harmonization | Penyelarasan env S3/R2, perintah reencrypt-pins, standardisasi broker. | Eliminasi ambiguitas konfigurasi. |
| 2.0.0 | 2026-10-23 | Staff Ahli MIK & Antigravity Lead | Seluruh Dokumen | **Sebelum:** Direktori Celery worker/tasks, modul formulir dinamis & interview booking, segregasi 3 instance Redis.<br>**Sesudah:** Eliminasi seluruh komponen formulir & Celery tasks, adopsi struktur murni Blog CMS sekelas WordPress, penyatuan 1 instance Redis di Docker Compose, perapian TipTap components di frontend, dan standarisasi konfigurasi deployment bertahap (Hostinger $\to$ Kampus Tel-U). | Pivot arsitektur WordPress-like Blog CMS, penyederhanaan tumpukan teknologi, dan portabilitas deploy. |

---

## 1. Root Repository Overview

Proyek dikelola sebagai **Decoupled Monorepo** di dalam satu repository Git:

```
himaiftelkomuniv/
├── .github/                      # CI/CD Workflows (lint, test, build, deploy)
│   └── workflows/
│       ├── frontend-ci.yml
│       └── backend-ci.yml
├── docker/                       # Konfigurasi containerization multi-service
│   ├── nginx/
│   │   ├── conf.d/
│   │   │   ├── default.conf      # Reverse proxy satu origin, R2 cache 5GB, rate limit
│   │   │   └── redirect_301.conf # Template redirector permanen untuk domain lama
│   │   └── ssl/                  # Sertifikat SSL
│   ├── docker-compose.yml        # Orchestration lokal: PostgreSQL, 1x Redis, Nginx
│   ├── docker-compose.prod.yml   # Orchestration produksi (Hostinger / VPS Kampus)
│   ├── Dockerfile.frontend       # Multi-stage Next.js production build
│   └── Dockerfile.backend        # Multi-stage Python FastAPI production build
├── backend/                      # Backend API Service (FastAPI + SQLAlchemy 2.0)
├── frontend/                     # Frontend Application (Next.js App Router + TipTap)
├── planning/                     # Dokumen perencanaan teknis master (10 berkas acuan)
├── .gitignore
├── .env.example                  # Template variabel lingkungan terpadu
└── README.md                     # Panduan setup & orientasi pengembang
```

---

## 2. Struktur Direktori Frontend (`/frontend`)

Frontend dibangun menggunakan **Next.js 15+ (App Router)** dengan styling Tailwind CSS dan komponen `shadcn/ui`:

```
frontend/
├── public/                       # Aset statis publik (logo HIMA IF, favicons, dummy)
├── src/
│   ├── middleware.ts             # UX Routing Guard (redirect login / dashboard)
│   ├── app/                      # Next.js App Router
│   │   ├── (public)/             # Route Group: Portal Publik (SEO SSR)
│   │   │   ├── layout.tsx        # Layout publik (Navbar, Footer, Lenis Smooth Scroll)
│   │   │   ├── page.tsx          # Homepage (Hero, Berita Terkini, Kategori Unggulan)
│   │   │   ├── blog/
│   │   │   │   ├── page.tsx      # Arsip Blog (Pagination, Filter Kategori/Tag, Search)
│   │   │   │   └── [slug]/
│   │   │   │       └── page.tsx  # Detail Artikel (SSR, Clean HTML Render, OpenGraph)
│   │   │   ├── categories/
│   │   │   │   └── [slug]/page.tsx # Halaman Filter Berita per Kategori
│   │   │   ├── sitemap.ts        # Dynamic sitemap.xml generator
│   │   │   └── robots.ts         # Dynamic robots.txt
│   │   ├── (auth)/               # Route Group: Autentikasi
│   │   │   ├── login/page.tsx    # Halaman Login Pengurus
│   │   │   └── change-password/  # Halaman Paksa Ganti Password Awal
│   │   └── (admin)/              # Route Group: CMS Dashboard (TipTap Editor)
│   │       ├── layout.tsx        # Sidebar Admin, User Header, Session Guard
│   │       ├── dashboard/page.tsx# Ringkasan Statistik (Total Post, Views, Draft)
│   │       ├── posts/
│   │       │   ├── page.tsx      # Tabel Manajemen Artikel (Filter Status, Author)
│   │       │   ├── new/page.tsx  # Editor Artikel Baru (TipTap WYSIWYG)
│   │       │   └── [id]/edit/    # Editor Perbarui Artikel & Riwayat Revisi
│   │       ├── categories/       # Manajemen Kategori (Super Admin only)
│   │       ├── tags/             # Manajemen Tags
│   │       ├── media/            # Media Library (Upload WebP, Gallery, Copy Link)
│   │       ├── users/            # Manajemen Akun Author (Super Admin only)
│   │       └── audit-logs/       # Log Keamanan (Super Admin only)
│   ├── components/
│   │   ├── ui/                   # Komponen primitif shadcn/ui (Button, Dialog, dll.)
│   │   ├── editor/               # TipTap Block Editor Suite
│   │   │   ├── tiptap-editor.tsx # Komponen Utama Editor Visual
│   │   │   ├── toolbar.tsx       # Toolbar Tombol Format Teks & Heading
│   │   │   ├── image-dialog.tsx  # Modal Penyisipan Gambar (Media Library / Upload)
│   │   │   ├── table-controls.tsx# Kontrol Tambah/Hapus Baris & Kolom Tabel
│   │   │   └── extensions/       # Custom TipTap Extensions (YouTube, Spotify, Code)
│   │   ├── blog/                 # Komponen Publik (PostCard, ShareButton, AuthorBio)
│   │   └── layout/               # Navbar, Footer, MobileNav
│   ├── lib/
│   │   ├── api-client.ts         # Axios/Fetch client terstandarisasi (Relative BaseURL)
│   │   ├── utils.ts              # Helper Tailwind (cn), formatting tanggal
│   │   └── seo.ts                # Helper kalkulasi Reading Time & SEO meta
│   └── stores/
│       └── auth-store.ts         # Zustand in-memory auth state (Current User, Role)
├── tailwind.config.ts            # Konfigurasi Tailwind CSS v3.4.x LTS
├── package.json
└── tsconfig.json
```

---

## 3. Struktur Direktori Backend (`/backend`)

Backend dibangun menggunakan **Python FastAPI** dengan pendekatan asinkron modular:

```
backend/
├── alembic/                      # Database Migration Engine
│   ├── versions/
│   │   ├── 001_auth_and_sessions.py
│   │   └── 002_blog_cms_and_media.py
│   └── env.py
├── app/
│   ├── api/                      # Routing & Controller Tier
│   │   └── v1/
│   │       ├── api.py            # Aggregator Router API v1
│   │       └── endpoints/
│   │           ├── auth.py       # Login, Logout, Me, Change Password
│   │           ├── users.py      # CRUD Akun Penulis (Super Admin)
│   │           ├── categories.py # CRUD Kategori Artikel
│   │           ├── tags.py       # CRUD & Search Tags
│   │           ├── posts.py      # CRUD Artikel, Publish, Schedule, Revisions
│   │           ├── media.py      # Presigned URL Cloudflare R2, Confirm Upload
│   │           └── audit_logs.py # Jejak Audit Sistem
│   ├── core/                     # Keamanan & Infrastruktur Inti
│   │   ├── config.py             # Pydantic BaseSettings (.env loader dinamis)
│   │   ├── security.py           # Argon2id password hashing & verifikasi
│   │   ├── auth_cookie.py        # Penerbitan cookie __Host-himaif_session
│   │   ├── sanitizer.py          # Rust nh3 HTML & URL allowlist sanitizer
│   │   └── csrf_guard.py         # Subdomain eTLD+1 CSRF Guard
│   ├── db/                       # Database Session & Seeding
│   │   ├── session.py            # Async engine SQLAlchemy (asyncpg)
│   │   ├── base.py               # Declarative Base
│   │   └── seed.py               # Akun Super Admin & Kategori Default
│   ├── models/                   # SQLAlchemy ORM Models
│   │   ├── user.py               # Model User & UserRoleEnum
│   │   ├── session.py            # Model Active Session
│   │   ├── category.py           # Model BlogCategory
│   │   ├── tag.py                # Model BlogTag & BlogPostTag
│   │   ├── post.py               # Model BlogPost (content_json, content_html)
│   │   ├── revision.py           # Model BlogPostRevision
│   │   ├── media.py              # Model MediaAsset
│   │   └── audit_log.py          # Model AuthAuditLog
│   ├── schemas/                  # Pydantic v2 DTOs (Request / Response validation)
│   │   ├── auth.py
│   │   ├── user.py
│   │   ├── category.py
│   │   ├── post.py
│   │   └── media.py
│   └── services/                 # Domain Logic Services
│       ├── post_service.py       # Logika publikasi, kalkulasi read time, tag sync
│       ├── storage_service.py    # Generator S3 SigV4 Presigned URL ke R2
│       └── session_service.py    # Redis cache get/set/invalidate sesi aktif
├── tests/                        # Unit & Integration Tests (Pytest)
├── Dockerfile                    # Multi-stage production container
├── requirements.txt              # Dependensi Python terkunci
└── main.py                       # FastAPI ASGI Entrypoint
```

---

## 4. Orkestrasi Docker Compose (`/docker`)

### 4.1 Spesifikasi Layanan (`docker-compose.prod.yml`)

Sistem terdiri dari **5 container ramping** yang mengonsumsi RAM total < 1 GB:

```yaml
version: "3.8"

networks:
  himaif-public:
    driver: bridge
  himaif-internal:
    driver: bridge

volumes:
  postgres_data:
  nginx_cache:
  certbot_www:
  certbot_conf:

services:
  # 1. Reverse Proxy & Ingress
  nginx:
    image: nginx:1.27-alpine
    container_name: himaif_nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/conf.d:/etc/nginx/conf.d:ro
      - nginx_cache:/var/cache/nginx
      - certbot_www:/var/www/certbot:ro
      - certbot_conf:/etc/letsencrypt:ro
    networks:
      - himaif-public
      - himaif-internal
    depends_on:
      - frontend
      - backend

  # 2. Frontend Next.js 15
  frontend:
    build:
      context: ./frontend
      dockerfile: ../docker/Dockerfile.frontend
    container_name: himaif_frontend
    restart: unless-stopped
    environment:
      - PORT=3000
      - NODE_ENV=production
      - INTERNAL_BACKEND_URL=http://backend:8000/api/v1
    networks:
      - himaif-internal
    deploy:
      resources:
        limits: { cpus: "1.00", memory: "512M" }
        reservations: { cpus: "0.20", memory: "128M" }

  # 3. Backend FastAPI
  backend:
    build:
      context: ./backend
      dockerfile: ../docker/Dockerfile.backend
    container_name: himaif_backend
    restart: unless-stopped
    command: uvicorn main:app --host 0.0.0.0 --port 8000 --proxy-headers --forwarded-allow-ips="*" --workers 2
    env_file: .env.production
    networks:
      - himaif-internal
    depends_on:
      - postgres
      - redis
    deploy:
      resources:
        limits: { cpus: "1.00", memory: "384M" }
        reservations: { cpus: "0.20", memory: "128M" }

  # 4. PostgreSQL 16
  postgres:
    image: postgres:16-alpine
    container_name: himaif_postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - himaif-internal
    deploy:
      resources:
        limits: { cpus: "1.00", memory: "512M" }
        reservations: { cpus: "0.20", memory: "256M" }

  # 5. Redis Cache & Session
  redis:
    image: redis:7-alpine
    container_name: himaif_redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 200mb --maxmemory-policy allkeys-lru
    networks:
      - himaif-internal
    deploy:
      resources:
        limits: { cpus: "0.25", memory: "200M" }
        reservations: { cpus: "0.05", memory: "64M" }
```

Struktur proyek ini menjamin keteraturan kode, kecepatan pengembangan, dan kemudahan deployment bertahap di berbagai jenis server VPS.
