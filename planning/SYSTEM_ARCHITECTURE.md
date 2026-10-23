# System Architecture & Technical Blueprint — HIMA IF CMS

| Field | Detail |
|---|---|
| **System** | HIMA IF CMS (Modern WordPress-like Blog & Content Platform) |
| **Document Version** | 3.0.0 |
| **Date** | 2026-10-23 |
| **Status** | Implementation-Ready Technical Plan |
| **Target Organization** | HIMA IF Telkom University (Himpunan Mahasiswa Informatika) |
| **Primary Maintainer & Designer** | Staff Ahli Divisi Media Informasi & Kreatif (MIK), Program Kerja Informasite (Informatics Website) HIMA IF Telkom University |

---

## Riwayat Revisi (Revision History)

| Versi | Tanggal | Penulis / Reviewer | Bagian yang Diubah | Rincian Perubahan (Sebelum vs Sesudah) | Alasan Perubahan |
|---|---|---|---|---|---|
| 1.0.0 | 2026-10-13 | Staff Ahli Divisi MIK | Dokumen Baru | Baseline arsitektur teknis sistem dual-engine. | Inisiasi dokumen awal. |
| 2.0.0 | 2026-10-14 | Staff Ahli MIK & Auditor | Concurrency & Redis | Row-locking, 3 Redis terpisah, ES256 JWT, S3 storage. | Penguatan keamanan & locking modul pendaftaran. |
| 2.1.0 | 2026-10-14 | Staff Ahli MIK & Auditor | Driver Storage & Nginx | Pembakuan Cloudflare R2, direktif body 25M, isolasi port Redis 6379. | Eliminasi ambiguitas payload & port. |
| 3.0.0 | 2026-10-23 | Staff Ahli MIK & Antigravity Lead | Seluruh Dokumen | **Sebelum:** Dual-engine (CMS & Recruitment), Celery Worker + Beat, segregasi 3 instance Redis, dan locking pendaftaran.<br>**Sesudah:** Single-engine murni Blog CMS sekelas WordPress, eliminasi Celery Worker & Beat, penyatuan 1 instance Redis (cache & rate limit), Nginx Single-Origin R2 Proxy Cache 5GB, cetak biru deployment bertahap (VPS Hostinger $\to$ VPS Kampus Telkom University dengan 301 Permanent Redirect), proteksi cookie `__Host-` & anti-CSRF eTLD+1. | Pivot keputusan organisasi: membatalkan modul interview/rekrutmen dan menyederhanakan arsitektur agar dapat tuntas 100% dalam 1 bulan tanpa BaaS. |

---

## 1. Executive Summary & Architectural Goals

Website CMS HIMA IF Telkom University dirancang sebagai **Mesin Publikasi Berita, Opini, dan Informasi Organisasi Modern (WordPress-Equivalent Blog CMS)** dengan integrasi visual editor WYSIWYG (TipTap ProseMirror), manajemen kategori/tag berkinerja tinggi, dan optimasi SEO otomatis.

### Prinsip Utama Arsitektur:
1. **Zero BaaS (No Vendor Lock-in)**: Beroperasi 100% pada server mandiri (*self-hosted* PostgreSQL 16 di VPS), tanpa ketergantungan pada Supabase, Firebase, PocketBase, atau Appwrite.
2. **Resource Efficiency & High Feasibility (1 Bulan)**: Menghilangkan seluruh kerumitan distributed task queue (Celery) dan segregasi 3 Redis. Sistem berjalan sangat ringan (total reservasi RAM < 1 GB) dan stabil pada server VPS 4 vCPU / 8 GB RAM.
3. **Decoupled Single-Origin Monorepo**: Frontend Next.js 15 dan Backend Python FastAPI berjalan secara terpisah namun disatukan di balik reverse proxy Nginx satu origin (`https://<domain>`), meniadakan friksi CORS dan preflight request.
4. **Sentralisasi Media Global**: Berkas gambar artikel disimpan secara terpusat di Cloudflare R2 (0 egress fee), di-cache secara lokal di SSD VPS via Nginx Proxy-Cache 5 GB (Cache HIT < 10ms).
5. **Staged Deployment Ready**: Mendukung alur rilis bertahap dari VPS Hostinger (`himaif.my.id`) ke VPS Kampus Telkom University (`himaif.telkomuniversity.ac.id`) secara mulus dengan waktu transfer data < 15 menit dan 0 byte transfer media.

---

## 2. High-Level Component Architecture

```mermaid
flowchart TB
    subgraph ClientLayer["Frontend Client Tier (Next.js 15 App Router)"]
        PublicUI["Portal Berita Publik\n(Home, Feed, Detail Artikel, SEO SSR)\nLenis + Tailwind CSS + shadcn/ui"]
        AdminUI["CMS Management Dashboard\n(Super Admin & Author Editor)\nTipTap WYSIWYG + Media Library"]
    end

    subgraph GatewayLayer["Single-Origin Ingress & Reverse Proxy (Nginx)"]
        Nginx["Nginx Reverse Proxy\n(SSL Termination, Single-Origin Routing,\nTrusted Proxy X-Forwarded-For, R2 Proxy Cache 5GB SSD)"]
    end

    subgraph BackendLayer["Application Tier (Python FastAPI)"]
        API["FastAPI Core Engine\n(REST API, Pydantic v2, SQLAlchemy 2.0 Async,\nSession Auth Guard, Rust nh3 HTML Sanitizer)"]
    end

    subgraph PersistenceLayer["Data & Persistence Tier"]
        PG[("PostgreSQL 16\n(Users, Sessions, Blog Posts,\nProseMirror AST JSONB, Revisions)")]
        RedisCache[("Redis 7 (Instance Tunggal)\n(API Response Cache, Version Tags,\nActive Session Index, Rate Limiter)")]
        R2Storage[("Cloudflare R2 Object Storage\n(WebP Image Assets, Zero-Egress Storage)")]
    end

    ClientLayer -->|HTTPS (Single Origin)| Nginx
    Nginx -->|Route '/' & '/_next'| ClientLayer
    Nginx -->|Route '/api/v1'| API
    Nginx -->|Route '/storage/public' (Cache HIT/MISS)| R2Storage

    API -->|Async Read/Write| PG
    API -->|Cache Get/Set & Rate Limit| RedisCache
    API -->|Generate Presigned Upload URL| R2Storage
```

---

## 3. Technology Stack & Technical Rationale

| Layer / Komponen | Teknologi Terpilih | Versi | Alasan Pemilihan & Manfaat Teknis |
|---|---|---|---|
| **Frontend Framework** | Next.js (App Router, React 19) | 15.x / 19.x | Mendukung Server-Side Rendering (SSR) untuk SEO publik instan, React Server Components (RSC) untuk payload JS minimal, dan fast client-side navigation. |
| **UI Components** | shadcn/ui & Tailwind CSS | 3.4.x (LTS) | Komponen UI accessible (Radix UI primitives), modular, mudah dikustomisasi, stabil pada Tailwind v3.4.x. |
| **Rich Text Editor & Sanitizer** | TipTap 2.x (ProseMirror AST) + Rust `nh3` | 2.x / latest | Headless WYSIWYG editor menghasilkan AST JSONB kanonik di frontend. Di backend, pustaka Rust `nh3` (menggantikan bleach) menjamin sanitasi HTML dari celah Stored XSS. |
| **Smooth Scrolling** | Lenis Scroll | 1.1+ | Memberikan pengalaman scroll visual premium pada artikel blog publik tanpa merusak scroll native mobile. |
| **Backend Framework** | Python FastAPI | 0.115+ | Framework asinkron berkinerja tinggi berbasis Starlette/ASGI, validasi data otomatis via Pydantic v2, auto-generated OpenAPI/Swagger. |
| **Database ORM & Migration** | SQLAlchemy (Async) + Alembic | 2.0+ | Asynchronous I/O via `asyncpg`, pengelolaan transaksi murni, dukungan native untuk kolom PostgreSQL `JSONB`, version control skema via Alembic. |
| **Relational Database** | PostgreSQL | 16+ | Mesin database relasional enterprise-grade, mendukung JSONB GIN indexing untuk full-text search bahasa Indonesia. |
| **In-Memory Caching & Rate Limiting** | Redis | 7.x (Single Instance) | Satu instance Redis hemat memori (maks 200 MB, allkeys-lru) untuk caching respons artikel, pencatatan session id aktif, dan atomic rate-limiting counters. |
| **Object Storage** | Cloudflare R2 | S3-Compatible API | Penyimpanan media terpusat, **0 biaya egress**, redundansi global, membuat migrasi server antar-VPS bernilai 0 detik. |

---

## 4. Single-Origin Ingress & Nginx Routing Blueprint

Seluruh sistem beroperasi pada **satu origin tunggal** (`https://<domain>`), menghilangkan masalah CORS pada browser:

```nginx
# /docker/nginx/conf.d/default.conf

# 1. Local Disk Cache Zone untuk Cloudflare R2 (Maksimal 5 GB SSD VPS)
proxy_cache_path /var/cache/nginx/r2_media levels=1:2 keys_zone=r2_cache:20m max_size=5g inactive=30d use_temp_path=off;

# 2. Rate Limiting Zones (Menggunakan Real Client IP)
limit_req_zone $binary_remote_addr zone=general_limit:10m rate=20r/s;
limit_req_zone $binary_remote_addr zone=auth_limit:10m rate=5r/m;

upstream frontend_upstream {
    server frontend:3000;
    keepalive 32;
}

upstream backend_upstream {
    server backend:8000;
    keepalive 32;
}

# HTTP: Redirect ke HTTPS & ACME Webroot
server {
    listen 80;
    server_name himaif.telkomuniversity.ac.id himaif.my.id;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
        try_files $uri =404;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS: Main Application Ingress
server {
    listen 443 ssl http2;
    server_name himaif.telkomuniversity.ac.id himaif.my.id;

    ssl_certificate /etc/letsencrypt/live/himaif/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/himaif/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Trusted Upstream Proxy (Subnet Gateway DTI Kampus / Hostinger)
    set_real_ip_from 10.0.0.0/8;
    set_real_ip_from 172.16.0.0/12;
    set_real_ip_from 192.168.0.0/16;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Global Buffer & Payload Limit
    client_max_body_size 25M;

    # Route: Backend REST API
    location /api/ {
        limit_req zone=general_limit burst=30 nodelay;
        proxy_pass http://backend_upstream;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Route: Proteksi Brute Force Khusus Login
    location /api/v1/auth/login {
        limit_req zone=auth_limit burst=3 nodelay;
        proxy_pass http://backend_upstream;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Route: Proxy Media Cache Cloudflare R2 (Single-Origin Media)
    location /storage/public/ {
        proxy_pass https://pub-xxxxxxxxxxxx.r2.dev/;
        proxy_set_header Host pub-xxxxxxxxxxxx.r2.dev;
        
        proxy_cache r2_cache;
        proxy_cache_valid 200 302 30d;
        proxy_cache_valid 404 1m;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_lock on;

        add_header X-Cache-Status $upstream_cache_status;
        add_header Cache-Control "public, max-age=31536000, immutable";
        proxy_hide_header Set-Cookie;
        proxy_ignore_headers Set-Cookie;
    }

    # Route: Next.js Static Build Assets (Cache Agresif)
    location /_next/static/ {
        proxy_pass http://frontend_upstream;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }

    # Route: Next.js App Router (SSR) & Fast Refresh
    location / {
        proxy_pass http://frontend_upstream;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 5. Strategi Deployment Bertahap (Staged Rollout Plan)

Arsitektur sistem dirancang mendukung migrasi dua tahap secara transparan:

```
[Tahap 1: Go-Live di VPS Hostinger (Domain: himaif.my.id)]
- Setup Docker Compose standar (Nginx + Frontend + Backend + Postgres + Redis).
- Seluruh media gambar tersimpan di Cloudflare R2 (bukan disk Hostinger).
- TipTap menyimpan gambar dengan relative path '/storage/public/{id}.webp'.
                                │
                                ▼  (Izin DTI Kampus Selesai)
[Tahap 2: Cutover ke VPS Kampus (Domain: himaif.telkomuniversity.ac.id)]
- Setup VPS Kampus dengan repository & Docker Compose yang identik.
- Database PostgreSQL di-stream langsung via SSH pipe (pg_dump | ssh | pg_restore).
- Waktu transfer media = 0 detik karena data gambar tetap berada di Cloudflare R2.
- VPS Hostinger dikonversi menjadi Nginx 301 Redirector permanen (preservasi SEO).
```

---

## 6. Parameter Keamanan & Otorisasi Kunci

1. **Host-Only Session Cookie (`__Host-himaif_session`)**:
   - Atribut `Domain` **wajib dikosongkan (`None`)**. Menggunakan prefix standar `__Host-` dengan atribut `HttpOnly`, `Secure`, dan `SameSite=Lax`.
   - Mengisolasi cookie sesi secara mutlak agar tidak bocor atau tertimpa (*cookie tossing*) dari sibling subdomain kampus lain di `*.telkomuniversity.ac.id`.
2. **Sibling Subdomain Anti-CSRF**:
   - Memvalidasi header `Sec-Fetch-Site: same-origin` dan kewajiban custom header (`X-Requested-With`) pada seluruh request mutasi data (`POST`, `PUT`, `DELETE`).
3. **Penjadwalan Tayang Otomatis Tanpa Background Worker**:
   - Kueri publik otomatis menyajikan artikel terjadwal saat jam tayang terlampaui:
     `WHERE status = 'PUBLISHED' OR (status = 'SCHEDULED' AND published_at <= NOW())`.

Arsitektur ini menjamin keandalan teknis, skalabilitas SEO, efisiensi sumber daya server, dan kesiapan rilis dalam 1 bulan tanpa BaaS.
