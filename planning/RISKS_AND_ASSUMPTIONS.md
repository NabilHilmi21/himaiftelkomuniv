# Analisis Risiko & Dokumen Asumsi Desain — HIMA IF CMS

| Field | Detail |
|---|---|
| **System** | HIMA IF CMS |
| **Document Version** | 1.1.0 |
| **Date** | 14 October 2026 |
| **Status** | Approved Specification |
| **Review Cycle** | Tahunan (Menjelang Regenerasi Pengurus HIMA IF) |

---

## Riwayat Revisi (Revision History)

| Versi | Tanggal | Penulis | Ringkasan Perubahan | Alasan |
|---|---|---|---|---|
| 1.0.0 | 13 Oktober 2026 | Staff Ahli Divisi MIK (Proker Informasite) | Draf awal analisis risiko dan asumsi teknis. | Inisiasi identifikasi risiko dan asumsi baseline. |
| 1.1.0 | 14 Oktober 2026 | Staff Ahli Divisi MIK & Antigravity Auditor | Pembaruan asumsi wajib booking 2 slot untuk 2 divisi, penambahan asumsi pengingat manual WhatsApp H-30 menit, integrasi batas H-1 reschedule mandiri, mitigasi dekompresi bom Pillow, pembaruan sanitasi HTML ke `nh3`, mitigasi IP spoofing di balik reverse proxy, penambahan risiko slot thrashing & impersonasi PIN, serta formalisasi kebijakan retensi data orphan dan penandaan status retensi pelamar (`BELUM DIPUTUSKAN`). | Penyelarasan mitigasi risiko dengan hasil audit keamanan dan keputusan arsitektur v2.0.0. |

---

## 1. Log Asumsi Desain (Design Assumptions)

Dalam merancang spesifikasi teknis HIMA IF CMS, kebutuhan operasional yang telah disepakati ditetapkan sebagai asumsi rekayasa perangkat lunak terverifikasi:

### Asumsi 1: Format Identitas Pseudo-Account & Ketiadaan Mail Server
- Format email (`admin@informasite.himaif`, `admin@namakepanitiaan.himaif`, `...@divisi.namakepanitiaan`) berfungsi murni sebagai **Structured Login Identifier / Username internal**, bukan mailbox nyata.
- Sistem tidak memerlukan mail server (SMTP), DNS MX record, atau layanan email pihak ketiga.
- Validasi input di frontend dan backend menggunakan custom regex terisolasi yang mengizinkan format domain internal `.himaif` tanpa dependensi IANA TLD.
- Pembagian kredensial akun awal dilakukan secara langsung oleh pembuat akun (Super Admin atau Master Formulir).

### Asumsi 2: Kredensial Pendaftar Publik & Proteksi PIN 6-Digit
- Mahasiswa pendaftar publik tidak memiliki akun pada tabel `users`.
- Saat submit berhasil, pendaftar menerima **Kode Registrasi Unik** (misal: `REG-PEMILU-88219`) dan **PIN Keamanan 6-Digit** acak via CSPRNG.
- PIN disimpan secara terenkripsi di level aplikasi menggunakan **AES-256-GCM / Fernet** dengan format `{key_version}:{ciphertext}`.
- PIN hanya dapat dilihat dan di-reset oleh **SUPER_ADMIN** dan **FORM_MASTER** (khusus formulir miliknya). Akun `DIVISION_STAFF` **dilarang keras** melihat atau mereset PIN.
- Setiap aksi melihat PIN dicatat ke `auth_audit_logs` dengan event `VIEW_APPLICANT_PIN`, dan reset PIN dicatat dengan `RESET_APPLICANT_PIN`.

### Asumsi 3: Pendaftar Multi-Divisi Wajib Memesan Dua Slot Wawancara
- Jika pendaftar memilih dua divisi (`primary_division_id` dan `secondary_division_id`), maka pendaftar **wajib memesan dua slot wawancara terpisah** pada saat pengisian formulir.
- Kedua slot harus berasal dari masing-masing divisi terkait dan rentang waktunya tidak boleh saling bertabrakan (*anti-overlap*).
- Sistem memproses reservasi kedua slot secara atomik all-or-nothing menggunakan transaksi database dengan urutan penguncian global anti-deadlock (`ORDER BY id ASC FOR UPDATE`).

### Asumsi 4: Konkurensi Pewawancara Divisi (Advisory Lock + Optimistic Locking)
- Akun divisi (`...@divisi.namakepanitiaan`) dapat digunakan secara bersamaan oleh tim pewawancara pada beberapa komputer browser.
- Untuk mencegah bentrokan data:
  1. Lembar kerja wawancara dilindungi **Redis Advisory Lock** (`interview:lock:{booking_id}`) dengan heartbeat 30 detik via script Lua atomik untuk memberikan penanda visual penilai aktif.
  2. Database menerapkan **Optimistic Locking** melalui kolom `version` pada tabel `interview_bookings`. Jika dua penilai mencoba menyimpan bersamaan, transaksi kedua ditolak dengan HTTP 409 `RESOURCE_CONFLICT`.

### Asumsi 5: Batasan Akses Modul Blog
- Modul Blog CMS adalah representasi resmi HIMA IF secara kelembagaan, sehingga pengelolaan artikel blog (CRUD, penjadwalan, SEO, dan publikasi) dibatasi **eksklusif untuk SUPER_ADMIN**.
- Master Formulir dan Akun Divisi tidak memiliki akses ke modul Blog.

### Asumsi 6: Integritas Struktur Field Formulir yang Telah Berjalan
- Field formulir dinamis yang telah memiliki data isian pendaftar **dilarang dihapus permanen (*hard delete*)**.
- Sistem hanya mengizinkan pengarsipan field (*soft-delete/hide*), menjaga integritas relasional data historis pendaftar terdahulu (*zero data corruption*).

### Asumsi 7: Operasional Pengingat Jadwal (WhatsApp Manual H-30 Menit)
- Mengingat ketiadaan SMTP mail server, mahasiswa pendaftar wajib secara proaktif memeriksa jadwal dan status seleksinya melalui portal `/status`.
- Sebagai prosedur operasional organisasi, panitia divisi akan mengingatkan peserta secara manual melalui **WhatsApp pada H-30 menit** sebelum sesi wawancara dimulai (mengirimkan informasi ruangan fisik / link Google Meet).

### Asumsi 8: Reschedule Mandiri Publik
- Pendaftar diperbolehkan mengubah jadwal wawancaranya secara mandiri tanpa batas jumlah kali, asalkan dilakukan paling lambat **H-1 (24 jam sebelum waktu sesi saat ini)**.
- Setiap request reschedule wajib memvalidasi token **Cloudflare Turnstile** dan dilindungi rate limiter ketat.

---

## 2. Matriks Analisis Risiko Teknis & Mitigasi

| # | Skenario Risiko | Tingkat Risiko | Dampak Operasional | Strategi Mitigasi Terpilih |
|---|---|:---:|---|---|
| **R-01** | **Race Condition & Deadlock Pemesanan Slot Multi-Divisi**<br>Banyak pelamar memesan slot bersilangan secara bersamaan. | **Kritis (Critical)** | Deadlock database, kegagalan pemesanan massal, atau *overbooking*. | Terapkan urutan penguncian global anti-deadlock: kunci slot dengan `SELECT ... FROM interview_slots WHERE id IN (...) ORDER BY id ASC FOR UPDATE`, lalu eksekusi validasi kapasitas dan update secara atomik. |
| **R-02** | **Insecure Direct Object References (IDOR) Antar Divisi**<br>Akun Divisi memanipulasi parameter URL untuk mengakses pendaftar atau evaluasi divisi lain. | **Kritis (Critical)** | Kebocoran data pribadi (PII) dan nilai rahasia pendaftar. | Terapkan dependency guard di FastAPI (`verify_submission_access` dan `verify_booking_evaluator`) yang memvalidasi `current_user.division_id` terhadap `primary_division_id`, `secondary_division_id`, atau relasi kepemilikan booking. |
| **R-03** | **Lupa Password Akun Tanpa Mail Server**<br>Koordinator divisi atau ketua panitia lupa kredensial login. | **Sedang (Medium)** | Akses terhenti, proses seleksi wawancara tertunda. | Terapkan delegasi hierarkis: Master Formulir mereset akun divisi, Super Admin mereset Master Formulir. Disediakan CLI darurat server: `python -m app.cli reset-superadmin`. |
| **R-04** | **Pembengkakan Penyimpanan (Storage Bloat) & Berkas Orphan**<br>Ratusan pendaftar mengunggah berkas besar atau mengunggah berkas tanpa menyelesaikan form. | **Tinggi (High)** | Disk server penuh, biaya penyimpanan melonjak. | 1. Batasi ukuran berkas 5–10 MB di Nginx & backend.<br>2. Pipeline kompresi Pillow ke WebP/AVIF.<br>3. Task Celery otomatis menghapus berkas orphan setelah 24 jam jika tidak direferensikan oleh submisi aktif. |
| **R-05** | **Stored XSS via Blok Custom HTML di Blog**<br>Penyisipan tag script atau event handler jahat pada block editor. | **Kritis (Critical)** | Pengambilalihan sesi akun admin atau defacement website. | Sanitasi ganda: sanitasi client-side menggunakan `DOMPurify` dan sanitasi server-side menggunakan library Rust modern `nh3` (menggantikan `bleach` yang telah deprecated) dengan whitelist tag ketat. |
| **R-06** | **Decompression Bomb & Pixel Flood Attack**<br>Pengunggahan gambar berdimensi raksasa berukuran byte kecil. | **Kritis (Critical)** | Crash server backend akibat lonjakan alokasi RAM (OOM Killer). | 1. Konfigurasi `Image.MAX_IMAGE_PIXELS = 16_777_216`.<br>2. Eskalasi `DecompressionBombWarning` menjadi error fatal.<br>3. Validasi dimensi header (`img.size`) sebelum melakukan dekompresi bitmap penuh ke RAM. |
| **R-07** | **Brute-force Login & Enumerasi PIN Pendaftar**<br>Penyerang menebak password internal atau PIN 6-digit pendaftar di portal publik. | **Tinggi (High)** | Akun admin dibajak atau jadwal wawancara mahasiswa diutak-atik. | 1. Rate limiting bertingkat via Redis (5 kegagalan per 15 menit, disusul lockout bertahap).<br>2. Evaluasi kredensial pendaftar dengan `secrets.compare_digest` (*constant-time*).<br>3. Respons error identik untuk mencegah enumerasi NIM/kode. |
| **R-08** | **Slot Thrashing via Reschedule Publik Tanpa Batas**<br>Bot atau pelamar memindahkan jadwal berulang kali untuk memonopoli slot. | **Tinggi (High)** | Slot kosong palsu, ketidakpastian jadwal pewawancara. | 1. Cut-off waktu ketat H-1 (24 jam sebelum sesi).<br>2. Proteksi Cloudflare Turnstile *fail-closed* pada setiap request reschedule.<br>3. Rate limit per NIM dan per IP. |
| **R-09** | **Pemalsuan Header IP (IP Spoofing) di Balik Proxy**<br>Penyerang memanipulasi header `X-Forwarded-For` untuk membypass rate limit. | **Tinggi (High)** | Rate limiter tidak berfungsi, server rentan terhadap DoS. | FastAPI dijalankan dengan flag `--proxy-headers` dan `--forwarded-allow-ips` yang dikonfigurasi secara eksplisit hanya mempercayai IP container Nginx proxy satu origin. |
| **R-10** | **CPU Blocking Akibat Invalidation Cache Berbasis Pattern**<br>Penggunaan perintah `KEYS` atau `SCAN` memblokir Redis event loop. | **Tinggi (High)** | Latensi API melonjak drastis, request timeout massal. | Hapus perintah `SCAN`+`DEL`. Terapkan **Version Tag Invalidation $O(1)$** (`INCR cms:blog:version:list`). |

---

## 3. Rencana Keberlanjutan & Tata Kelola Data (Governance & Retention)

### 3.1 Kebijakan Retensi Berkas & Data Pelamar
1. **Berkas Upload Orphan**:
   - Seluruh berkas lampiran yang diunggah tetapi tidak direferensikan oleh baris `form_submissions` dalam waktu **24 jam** akan dihapus secara permanen dari object storage melalui task Celery Beat harian.
2. **Data Pelamar Ditolak**:
   - Berkas lampiran dihapus permanen dan baris database dianonimkan (seluruh data identitas pribadi seperti NIM, kontak, dan jawaban dihapus/diganti hash anonim) setelah: **[BELUM DIPUTUSKAN]** hari sejak tanggal pengumuman final kepanitiaan.
3. **Data Pelamar Diterima**:
   - Kebijakan retensi dan pengarsipan data pelamar yang lolos seleksi: **[BELUM DIPUTUSKAN]**.

### 3.2 Protokol Pergantian Kabinet Tahunan
- Setiap akhir periode kepengurusan HIMA IF, Super Admin lama wajib melakukan rotasi kredensial utama (`admin@informasite.himaif`), merotasi keypair JWT asimetris (ES256), dan mengarsipkan seluruh formulir kepanitiaan periode tersebut.
- Riwayat `auth_audit_logs` disimpan selama minimal 2 tahun akademik untuk kebutuhan verifikasi arsip dan audit organisasi mahasiswa.
