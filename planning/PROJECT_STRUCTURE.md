# Struktur Proyek & Panduan Repositori — HIMA IF CMS

| Field | Detail |
|---|---|
| **System** | HIMA IF CMS |
| **Document Version** | 1.1.0 |
| **Date** | 14 October 2026 |
| **Status** | Approved Specification |
| **Monorepo Layout** | Decoupled Architecture (`/frontend` + `/backend` + `/docker`) |
| **Frontend Runtime** | Node.js 22 LTS / Bun (Next.js 15+ App Router) |
| **Backend Runtime** | Python 3.12+ (FastAPI + Asyncpg + Celery 5.4+) |

---

## Riwayat Revisi (Revision History)

| Versi | Tanggal | Penulis | Ringkasan Perubahan | Alasan |
|---|---|---|---|---|
| 1.0.0 | 13 Oktober 2026 | Staff Ahli Divisi MIK (Proker Informasite) | Draf awal struktur repositori monorepo. | Inisiasi arsitektur berkas dan folder. |
| 1.1.0 | 14 Oktober 2026 | Staff Ahli Divisi MIK & Antigravity Auditor | Penambahan berkas arsitektur (`src/middleware.ts`, `app/core/encryption.py`, `app/tasks/cleanup_tasks.py`, `app/db/seed.py`), konfigurasi 3 instance Redis terpisah, variabel Turnstile dan kunci enkripsi PIN pada `.env.example`, penyesuaian workspace evaluasi berbasis `[booking_id]`, dan pemeliharaan referensi `/fixed_md`. | Penyelarasan dengan spesifikasi keamanan v2.0.0 dan pemisahan concern tugas latar belakang. |

---

## 1. Root Repository Overview

Proyek diorganisir sebagai **Decoupled Monorepo** di dalam satu repository Git, mempermudah koordinasi deployment via Docker Compose namun tetap menjaga pemisahan *concerns* yang ketat:

```
himaiftelkomuniv/
├── .github/                      # CI/CD Workflows (lint, test, build, deploy)
│   └── workflows/
│       ├── frontend-ci.yml
│       └── backend-ci.yml
├── docker/                       # Konfigurasi containerization multi-service
│   ├── nginx/
│   │   ├── conf.d/default.conf   # Nginx reverse proxy satu origin & rate limiting config
│   │   └── ssl/                  # Sertifikat SSL lokal / staging
│   ├── docker-compose.yml        # Orchestration lokal: PostgreSQL, 3x Redis, S3-storage
│   ├── docker-compose.prod.yml   # Orchestration produksi lengkap
│   ├── Dockerfile.frontend       # Multi-stage Next.js production build
│   └── Dockerfile.backend        # Multi-stage Python FastAPI production build
├── backend/                      # Backend API Service (FastAPI + Celery)
├── frontend/                     # Frontend Application (Next.js App Router)
├── fixed_md/                     # Dokumentasi acuan awal organisasi (dipertahankan)
├── new_md/                       # Dokumen perencanaan teknis mutakhir
├── .gitignore
├── .env.example                  # Template variabel lingkungan terpadu
└── README.md                     # Panduan setup & orientasi pengembang baru
```

---

## 2. Struktur Direktori Frontend (`/frontend`)

Frontend dibangun menggunakan **Next.js 15+ (App Router)** dengan arsitektur modular:

```
frontend/
├── public/                       # Aset statis publik (logo HIMA IF, favicons, illustrations)
│   ├── icons/
│   └── images/
├── src/
│   ├── middleware.ts             # Edge UX routing middleware (baca refresh token untuk UX redirect)
│   ├── app/                      # Next.js App Router (Routing, Layouts, Pages)
│   │   ├── (public)/             # Route Group: Halaman publik tanpa autentikasi
│   │   │   ├── layout.tsx        # Layout publik (Navbar, Footer, Lenis Provider)
│   │   │   ├── page.tsx          # Homepage HIMA IF (Hero, Latest News, Active Events)
│   │   │   ├── blog/
│   │   │   │   ├── page.tsx      # Arsip Blog (Paginasi, filter kategori, search)
│   │   │   │   └── [slug]/
│   │   │   │       └── page.tsx  # Halaman detail artikel blog (SSR + SEO Meta)
│   │   │   ├── forms/
│   │   │   │   └── [slug]/
│   │   │   │       └── page.tsx  # Form Wizard publik (Dynamic fields + Multi-Slot picker + Turnstile)
│   │   │   └── status/
│   │   │       └── page.tsx      # Cek status pendaftaran & reschedule mandiri (Code + NIM + PIN)
│   │   ├── (auth)/               # Route Group: Halaman login internal
│   │   │   └── login/
│   │   │       └── page.tsx      # Portal login pseudo-email untuk seluruh role
│   │   ├── admin/                # Route Group: Dashboard terproteksi (RBAC)
│   │   │   ├── layout.tsx        # Shell layout dashboard (Sidebar, Header, Breadcrumbs)
│   │   │   ├── page.tsx          # Role-based landing dashboard router
│   │   │   ├── super/            # Sub-dashboard khusus Super Admin
│   │   │   │   ├── overview/page.tsx
│   │   │   │   ├── blogs/        # Manajemen blog global (CRUD, Publish, Schedule)
│   │   │   │   │   ├── page.tsx
│   │   │   │   │   ├── new/page.tsx
│   │   │   │   │   └── [id]/edit/page.tsx
│   │   │   │   ├── forms/page.tsx # Sentralisasi seluruh kepanitiaan / acara
│   │   │   │   └── users/page.tsx # Manajemen akun dan reset password global
│   │   │   ├── master/           # Sub-dashboard khusus Master Formulir
│   │   │   │   ├── forms/
│   │   │   │   │   └── [id]/
│   │   │   │   │       ├── overview/page.tsx   # Rekapitulasi pendaftar & interview
│   │   │   │   │       ├── builder/page.tsx    # Dynamic Form Builder (Fields & Rules)
│   │   │   │   │       ├── divisions/page.tsx  # Kelola divisi & akun divisi otomatis
│   │   │   │   │       └── submissions/page.tsx# Seluruh pendaftar lintas divisi
│   │   │   └── division/         # Sub-dashboard khusus Akun Divisi
│   │   │       ├── submissions/page.tsx # Pendaftar divisinya (Divisi 1 atau 2)
│   │   │       ├── schedule/page.tsx    # Manajemen slot & blokir jadwal interview
│   │   │       ├── questions/page.tsx   # Custom Question Builder untuk interview
│   │   │       └── workspace/[booking_id]/page.tsx # Split-screen workspace wawancara berbasis booking
│   │   ├── api/                  # Next.js Route Handlers (BFF / proxy jika dibutuhkan)
│   │   ├── globals.css           # Tailwind CSS imports & global theme CSS variables
│   │   └── layout.tsx            # Root HTML layout (Fonts, Meta, Toast Provider)
│   │
│   ├── components/               # Komponen React modular
│   │   ├── ui/                   # Komponen primitif shadcn/ui (Button, Dialog, Dropdown, dll.)
│   │   ├── common/               # Komponen generik (Navbar, Footer, SEOHead, EmptyState)
│   │   ├── blog/                 # Komponen khusus blog (BlogCard, BlockRenderer, SEOPanel)
│   │   ├── editor/               # Komponen TipTap Rich Text Editor & custom block extensions
│   │   ├── forms/                # Komponen dynamic form renderer, Turnstile widget, input fields
│   │   ├── interview/            # Komponen slot picker multi-divisi, calendar view, rubric sheet
│   │   └── providers/            # React context providers (Lenis, Auth, QueryClient)
│   │
│   ├── hooks/                    # Custom React hooks (useAuth, useDebounce, useAutoSave)
│   ├── lib/                      # Utilities & konfigurasi library pihak ketiga
│   │   ├── api-client.ts         # Axios / Fetch wrapper dengan interceptor JWT refresh
│   │   ├── utils.ts              # cn helper Tailwind class merging
│   │   ├── seo-analyzer.ts       # Algoritma perhitungan skor SEO real-time di browser
│   │   └── constants.ts          # Enum status, batasan file, dan konfigurasi rute
│   ├── stores/                   # State Management global via Zustand
│   │   ├── auth-store.ts         # Session state, user identity (Access Token in-memory murni, non-persisted)
│   │   ├── form-builder-store.ts # State interaktif builder formulir
│   │   └── evaluation-store.ts   # State auto-save lembar kerja wawancara & optimistic version
│   └── types/                    # Definisi TypeScript interfaces & types
│       ├── auth.ts
│       ├── blog.ts
│       ├── form.ts
│       └── interview.ts
├── package.json
├── tailwind.config.ts
├── tsconfig.json
└── next.config.ts
```

---

## 3. Struktur Direktori Backend (`/backend`)

Backend dibangun menggunakan **Python 3.12+ dan FastAPI** dengan pola *Service-Repository*:

```
backend/
├── alembic/                      # Script migrasi database SQLAlchemy
│   ├── versions/                 # File revisi migrasi terurut
│   └── env.py
├── app/
│   ├── api/                      # Routing layer & endpoint HTTP controllers
│   │   ├── v1/
│   │   │   ├── router.py         # Master router agregasi v1
│   │   │   ├── endpoints/
│   │   │   │   ├── auth.py       # Login pseudo-email, refresh ES256, change-password, me
│   │   │   │   ├── users.py      # Pengelolaan user & reset password hierarkis
│   │   │   │   ├── blog.py       # CRUD artikel, kategori, revisi, publish, Version Tag cache
│   │   │   │   ├── media.py      # Upload berkas & gambar (Pillow), presigned URL vault
│   │   │   │   ├── forms.py      # Dynamic form CRUD & field builder
│   │   │   │   ├── divisions.py  # Divisi & auto-provisioning akun divisi (must_change_password)
│   │   │   │   ├── submissions.py# Penerimaan form (2 slot), verify-status, reset-pin
│   │   │   │   ├── interviews.py # Generator slot, blocking, public reschedule H-1
│   │   │   │   ├── evaluations.py# Workspace evaluasi berbasis booking_id & optimistic locking
│   │   │   │   └── analytics.py  # Rekapitulasi sentralisasi & ekspor XLSX/CSV RESTful
│   │   └── deps.py               # Dependency injection (DB session, Current User, RBAC scope guards)
│   │
│   ├── core/                     # Konfigurasi inti aplikasi
│   │   ├── config.py             # Pydantic BaseSettings (membaca file .env)
│   │   ├── security.py           # Hashing password (Argon2id OWASP) & JWT asimetris (ES256)
│   │   ├── encryption.py         # Enkripsi simetris tracking_pin pelamar (AES-256-GCM / Fernet)
│   │   ├── redis.py              # Konektor pool 3 instance Redis (Cache, Broker, Session)
│   │   └── exceptions.py         # Custom exception handler & HTTP error mappings RFC 7807
│   │
│   ├── db/                       # Konfigurasi database relasional
│   │   ├── base.py               # SQLAlchemy DeclarativeBase
│   │   ├── session.py            # Async engine & sessionmaker (asyncpg)
│   │   └── seed.py               # Script seeding initial akun Super Admin & kategori bawaan
│   │
│   ├── models/                   # Definisi entitas database SQLAlchemy ORM
│   │   ├── user.py               # Model User & AuthAuditLog
│   │   ├── blog.py               # Model BlogPost, Category, Revision
│   │   ├── media.py              # Model MediaAsset (S3 storage)
│   │   ├── form.py               # Model Form, FormDivision, FormField
│   │   ├── submission.py         # Model FormSubmission (PIN terenkripsi, status keputusan)
│   │   ├── interview.py          # Model ScheduleConfig, InterviewSlot, InterviewBooking (evaluasi terlebur)
│   │   └── evaluation.py         # Model InterviewQuestionSet, InterviewQuestion, EvaluationAnswer
│   │
│   ├── schemas/                  # Schema validasi data input/output Pydantic v2
│   │   ├── auth.py
│   │   ├── blog.py
│   │   ├── form.py
│   │   ├── submission.py
│   │   ├── interview.py
│   │   └── evaluation.py
│   │
│   ├── services/                 # Business logic layer terisolasi
│   │   ├── auth_service.py       # Logika kredensial, rotasi refresh token, ganti password
│   │   ├── blog_service.py       # Logika penerbitan, revisioning, dan sanitasi nh3
│   │   ├── image_optimizer.py    # Logika konversi WebP/AVIF via Pillow & anti-decompression bomb
│   │   ├── form_engine.py        # Validasi dinamis isian formulir terhadap skema
│   │   ├── slot_booking.py       # Global lock ordering (ORDER BY id ASC FOR UPDATE)
│   │   └── export_service.py     # Pembangkitan file spreadsheet Excel/CSV tanpa PIN
│   │
│   ├── tasks/                    # Task asinkron background (Celery Worker)
│   │   ├── celery_app.py         # Inisialisasi Celery worker & koneksi Redis broker
│   │   ├── blog_scheduler.py     # Eksekusi rutin artikel terjadwal (Celery Beat)
│   │   ├── image_worker.py       # Pemrosesan varian gambar asinkron
│   │   └── cleanup_tasks.py      # Retensi berkas orphan (24 jam) & retensi data pelamar
│   │
│   └── cli.py                    # Tool CLI administratif server (darurat reset Super Admin)
│
├── tests/                        # Automated unit & integration testing
│   ├── conftest.py               # Fixtures database test & client mock
│   ├── test_auth.py              # Uji coba login pseudo-email, Argon2id, & RBAC
│   ├── test_blog.py              # Uji coba CRUD blog, SEO scoring, Version Tag invalidation
│   ├── test_slot_concurrency.py  # Uji coba k6/Locust konkurensi booking & anti-deadlock
│   └── test_reschedule.py        # Uji coba reschedule publik H-1 cut-off
├── alembic.ini
├── pyproject.toml / requirements.txt
└── main.py                       # ASGI entrypoint aplikasi FastAPI
```

---

## 4. Konfigurasi Lingkungan (`.env.example`)

```ini
# ==========================================
# KONFIGURASI GLOBAL & ENVIRONMENT
# ==========================================
ENVIRONMENT=production
PROJECT_NAME="HIMA IF Telkom University CMS"
API_V1_PREFIX=/api/v1
CORS_ALLOWED_ORIGINS=https://himaif.id

# ==========================================
# ASYMMETRIC JWT (ES256) KEYPATHS
# ==========================================
JWT_ALGORITHM=ES256
JWT_PRIVATE_KEY_PATH=/etc/secrets/jwt_es256_private.pem
JWT_PUBLIC_KEY_PATH=/etc/secrets/jwt_es256_public.pem
ACCESS_TOKEN_EXPIRE_MINUTES=15
REFRESH_TOKEN_EXPIRE_DAYS=7

# ==========================================
# ENKRIPSI DATA PELAMAR (AES-256-GCM / Fernet)
# ==========================================
APPLICANT_PIN_ENCRYPTION_KEY=dGhpc19pc19hXzMyX2J5dGVfc2VjcmV0X2tleV9mb3JfYWVzIQ==
APPLICANT_PIN_KEY_VERSION=v1

# ==========================================
# BOT PROTECTION (CLOUDFLARE TURNSTILE)
# ==========================================
NEXT_PUBLIC_TURNSTILE_SITE_KEY=0x4AAAAAAABBBBBCCCCCDDDD
TURNSTILE_SECRET_KEY=0x4AAAAAAABBBBBCCCCCDDDD_SECRET_KEY

# ==========================================
# POSTGRESQL DATABASE (DOCKER / CLOUD)
# ==========================================
POSTGRES_SERVER=postgres
POSTGRES_PORT=5432
POSTGRES_USER=himaif_user
POSTGRES_PASSWORD=himaif_strong_password_2026
POSTGRES_DB=himaif_cms_db
DATABASE_URL=postgresql+asyncpg://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_SERVER}:${POSTGRES_PORT}/${POSTGRES_DB}

# ==========================================
# SEGREGATED REDIS INSTANCES (3 SEPARATE INSTANCES)
# ==========================================
# Instance 1: Dedicated Response & Model Cache
REDIS_CACHE_URL=redis://:redis_auth_cache_2026@redis-cache:6379/0

# Instance 2: Dedicated Celery Task Broker & Backend
REDIS_CELERY_BROKER_URL=redis://:redis_auth_celery_2026@redis-celery:6379/0

# Instance 3: Active Session, Token JTI, Rate Limiting & Advisory Locks
REDIS_SESSION_URL=redis://:redis_auth_session_2026@redis-session:6379/0

# ==========================================
# S3-COMPATIBLE OBJECT STORAGE (PRODUK: BELUM DIPUTUSKAN)
# ==========================================
STORAGE_PROVIDER=s3-compatible
STORAGE_ENDPOINT=http://storage-service:9000
STORAGE_ACCESS_KEY=storage_admin_user
STORAGE_SECRET_KEY=storage_super_secure_key_2026
STORAGE_PUBLIC_BUCKET=himaif-public-assets
STORAGE_VAULT_BUCKET=himaif-protected-vault
STORAGE_PUBLIC_BASE_URL=https://himaif.id/storage/public

# ==========================================
# SUPER ADMIN INITIAL SEED CREDENTIALS
# ==========================================
SUPER_ADMIN_IDENTIFIER=admin@informasite.himaif
SUPER_ADMIN_INITIAL_PASSWORD=InformatikaJaya2026!
```
