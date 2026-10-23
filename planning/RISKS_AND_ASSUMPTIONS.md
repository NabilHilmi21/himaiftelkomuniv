# Analisis Risiko & Dokumen Asumsi Desain — HIMA IF CMS

| Field | Detail |
|---|---|
| **System** | HIMA IF CMS (Modern WordPress-like Blog Engine) |
| **Document Version** | 2.0.0 |
| **Date** | 2026-10-23 |
| **Status** | Approved Specification |
| **Review Cycle** | Tahunan (Menjelang Regenerasi Pengurus HIMA IF) |

---

## Riwayat Revisi (Revision History)

| Versi | Tanggal | Penulis / Reviewer | Bagian yang Diubah | Rincian Perubahan (Sebelum vs Sesudah) | Alasan Perubahan |
|---|---|---|---|---|---|
| 1.0.0 | 2026-10-13 | Staff Ahli Divisi MIK | Dokumen Baru | Draf awal analisis risiko dan asumsi teknis sistem dual-engine. | Inisiasi identifikasi risiko baseline. |
| 1.1.0 | 14 Oktober 2026 | Staff Ahli MIK & Auditor | Concurrency & XSS | Asumsi booking 2 slot, mitigasi bom Pillow, sanitasi nh3, mitigasi IP spoofing. | Penyelarasan audit keamanan v2.0.0. |
| 1.2.0 | 14 Oktober 2026 | Staff Ahli MIK & Auditor | Kepatuhan UU PDP | Penetapan retensi data pelamar, jadwal Celery Beat, eliminasi BELUM DIPUTUSKAN. | Kepatuhan regulasi UU PDP No. 27/2022. |
| 2.0.0 | 2026-10-23 | Staff Ahli MIK & Antigravity Lead | Seluruh Dokumen | **Sebelum:** Asumsi modul formulir dinamis, jadwal wawancara, PIN pelamar, dan retensi data pelamar.<br>**Sesudah:** Eliminasi seluruh risiko modul interview; fokuskan asumsi pada arsitektur murni Blog CMS, alur mandiri Author, penyimpanan Cloudflare R2, dan mitigasi risiko migrasi bertahap (Hostinger $\to$ VPS Kampus Tel-U), risiko Stored XSS TipTap via Rust `nh3`, larangan SVG, dan keamanan cookie `__Host-`. | Penyesuaian pivot WordPress-like Blog CMS dan penyelarasan strategi deployment bertahap. |

---

## 1. Log Asumsi Desain (Design Assumptions)

Dalam merancang spesifikasi teknis HIMA IF CMS yang baru, kebutuhan operasional ditetapkan sebagai asumsi rekayasa perangkat lunak terverifikasi:

### Asumsi 1: Fokus Murni Platform Publikasi Blog (WordPress-Equivalent)
- Sistem berpusat 100% pada penerbitan berita, dokumentasi kegiatan, opini mahasiswa, dan profil kelembagaan HIMA IF.
- Seluruh kebutuhan formulir rekrutmen dinamis dan booking jadwal wawancara ditiadakan, merampingkan kompleksitas sistem sebesar ~70%.

### Asumsi 2: Dua Peran dengan Alur Kerja Mandiri (*Direct Publishing Workflow*)
- Sistem hanya menggunakan 2 peran pengguna: `SUPER_ADMIN` dan `AUTHOR`.
- Untuk mencegah bottleneck editorial, peran `AUTHOR` diizinkan mempublikasikan langsung (`PUBLISHED`) dan menjadwalkan (`SCHEDULED`) artikel buatannya sendiri tanpa menunggu antrean approval admin.

### Asumsi 3: Nol BaaS (*Self-Hosted Baseline*)
- Seluruh data relasional, autentikasi, dan riwayat revisi dikelola mandiri di dalam container PostgreSQL 16 di server VPS organisasi, tanpa ketergantungan pada vendor BaaS (Supabase, Firebase, PocketBase).

### Asumsi 4: Sentralisasi Media di Cloudflare R2 & Cache SSD Nginx
- Berkas gambar artikel tidak disimpan di filesystem harddisk lokal VPS, melainkan di Cloudflare R2 (S3-compatible, 0 biaya egress).
- Nginx reverse proxy di VPS bertindak sebagai caching proxy lokal sebesar 5 GB SSD (`/storage/public/*`), menyajikan gambar dengan latensi < 10 ms bagi pengunjung blog.

### Asumsi 5: Strategi Peluncuran Bertahap (*Staged Rollout*)
- Tahap 1: Deployment awal di VPS Hostinger menggunakan domain sementara (`himaif.my.id`) agar publikasi dapat berjalan segera tanpa menunggu birokrasi DTI kampus.
- Tahap 2: Migrasi ke VPS Kampus Telkom University dengan subdomain resmi (`himaif.telkomuniversity.ac.id`) setelah alokasi server dan DNS DTI selesai.
- VPS Hostinger dialihkan menjadi Nginx 301 Permanent Redirector untuk mengamankan link medsos dan reputasi SEO Google.

### Asumsi 6: Kontrak Path Relatif pada Konten TipTap
- Seluruh gambar di dalam dokumen TipTap wajib disimpan menggunakan relative path (`/storage/public/{id}.webp`), bukan absolute URL.
- Menjamin artikel historis bebas broken image dan bebas error CORS saat domain berganti.

---

## 2. Matriks Analisis Risiko Teknis & Mitigasi

| ID | Kategori | Skenario Risiko | Dampak (*Impact*) | Tingkat | Strategi Mitigasi Terverifikasi |
|---|---|---|---|:---:|---|
| **R-01** | **Keamanan Aplikasi** | **Stored XSS via WYSIWYG Content**<br>Penulis menyisipkan payload `<script>`, tag `<iframe>` asing, atau atribut `onload` di dalam artikel blog. | Pembajakan sesi admin saat membaca artikel di dashboard, defacement web publik. | **CRITICAL** | 1. Input client divalidasi sebagai pohon AST ProseMirror.<br>2. HTML disanitasi ketat di backend menggunakan pustaka Rust `nh3` dengan *tag/attribute/scheme allowlist* ketat.<br>3. Seluruh tautan otomatis disuntikkan `rel="noopener noreferrer nofollow"`. |
| **R-02** | **Keamanan Media** | **Eksploitasi SVG Scripting**<br>Pengguna mengunggah gambar `.svg` yang memuat embedded JavaScript XML. | Eksekusi skrip jahat di browser pengunjung saat membuka gambar. | **CRITICAL** | **Larangan Mutlak Berkas SVG**: Format `.svg` ditolak keras oleh backend. Format yang diizinkan eksklusif: `.webp`, `.png`, `.jpg`, `.jpeg`. Verifikasi biner melalui *magic bytes*. |
| **R-03** | **Keamanan Infrastruktur** | **Decompression Bomb / Pixel Flood**<br>Penyerang mengunggah gambar berukuran file kecil namun berdimensi $50.000 \times 50.000$ piksel. | Crash server akibat kehabisan memori RAM (*Out-Of-Memory / OOM Killer*). | **HIGH** | 1. Menetapkan batas maksimum resolusi `Image.MAX_IMAGE_PIXELS = 16_777_216` (16 Megapiksel / $4096 \times 4096$ px) di Pillow.<br>2. Eskalasi warning Pillow menjadi fatal error `DecompressionBombError`.<br>3. Batas ukuran file maksimal 5 MB di Nginx dan FastAPI. |
| **R-04** | **Keamanan Sesi** | **Session Hijacking Sibling Subdomain Kampus**<br>Cookie sesi bocor ke subdomain kampus lain di `*.telkomuniversity.ac.id`. | Pembajakan akun pengurus melalui server praktikum/lab mahasiswa lain yang rentan. | **HIGH** | 1. Wajib menggunakan cookie ber-prefix `__Host-himaif_session`.<br>2. Dilarang menyetel atribut `Domain` (murni Host-Only).<br>3. Atribut wajib: `HttpOnly=true`, `Secure=true`, `SameSite=Lax`. |
| **R-05** | **Keamanan Sesi** | **Cross-Subdomain CSRF (eTLD+1)**<br>Script di subdomain kampus lain memicu request mutasi ke CMS HIMA IF. | Perubahan data artikel atau pengaturan sistem tanpa izin. | **HIGH** | 1. Strict CORS whitelist tanpa regex wildcard.<br>2. Validasi header `Sec-Fetch-Site: same-origin` pada seluruh mutasi data (`POST`, `PUT`, `DELETE`).<br>3. Mewajibkan custom header `X-Requested-With`. |
| **R-06** | **Operasional & Data** | **Broken Image Pasca-Migrasi Domain**<br>Peralihan dari domain Hostinger ke domain kampus menyebabkan gambar artikel lama gagal dimuat. | Tampilan artikel historis rusak, error 404 pada ratusan gambar. | **HIGH** | 1. **Kontrak Path Relatif**: TipTap hanya menyimpan `/storage/public/{id}.webp`.<br>2. Media dipusatkan di Cloudflare R2 sehingga file tidak perlu dipindahkan antar-VPS.<br>3. Skrip sanitasi regex otomatis disiapkan sebagai fail-safe. |
| **R-07** | **SEO & Trafik** | **Kehilangan Trafik & Penalti Google Pasca-Migrasi**<br>Link di media sosial (@himaiftelkom) dan hasil pencarian Google menghasilkan error 404 saat domain berganti. | Anjloknya peringkat SEO dan reputasi digital organisasi. | **MEDIUM** | 1. Mempertahankan VPS Hostinger minimal 6–12 bulan sebagai Nginx 301 Permanent Redirector.<br>2. Pendaftaran fitur *Change of Address* di Google Search Console.<br>3. Dynamic sitemap.xml dan canonical URL tag otomatis di Next.js. |
| **R-08** | **Ketersediaan Layanan** | **Brute-Force Serangan Login Pengurus**<br>Penyerang mencoba menebak password akun admin secara massal. | Kompromi akun Super Admin, beban tinggi pada hashing Argon2id. | **MEDIUM** | 1. Rate limiter IP sliding window via Redis: maksimal 5 percobaan per menit per IP.<br>2. Progressive backoff non-blocking dengan eksekusi dummy Argon2id hash untuk mencegah timing leak enumeration.<br>3. Kebijakan password minimal 12 karakter. |

Matriks risiko dan asumsi ini menjadi fondasi operasional rekayasa sistem yang aman, andal, dan siap menghadapi skenario migrasi produksi tanpa kendala.
