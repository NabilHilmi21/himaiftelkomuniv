# Modul Blog CMS & Block Editor Engine — HIMA IF CMS

| Field | Detail |
|---|---|
| **System** | HIMA IF CMS (Modern WordPress-like Blog Engine) |
| **Document Version** | 3.0.0 |
| **Date** | 2026-10-23 |
| **Status** | Approved Specification |
| **Editor Engine** | TipTap 2.x (ProseMirror Headless Architecture) |
| **Sanitizer Engine** | Rust `nh3` Server-Side AST & HTML Sanitizer |
| **Scheduling Engine** | Celery-Free Database Query Filter (`published_at <= NOW()`) |
| **Access Scope** | `SUPER_ADMIN` (Seluruh Artikel) & `AUTHOR` (Alur Mandiri Artikel Milik Sendiri) |

---

## Riwayat Revisi (Revision History)

| Versi | Tanggal | Penulis / Reviewer | Bagian yang Diubah | Rincian Perubahan (Sebelum vs Sesudah) | Alasan Perubahan |
|---|---|---|---|---|---|
| 1.0.0 | 2026-10-13 | Staff Ahli Divisi MIK | Dokumen Baru | Baseline arsitektur modul blog dan block editor awal. | Inisiasi spesifikasi CMS blog. |
| 2.0.0 | 2026-10-14 | Staff Ahli MIK & Auditor | Sanitasi & Celery | Adopsi Rust `nh3`, Version Tag Invalidation $O(1)$, rujukan kanonikal thumbnail. | Penutupan celah XSS & optimasi cache. |
| 2.1.0 | 2026-10-14 | Staff Ahli MIK & Auditor | Tautan & Rute | Koreksi tautan gambar dan keselarasan rute RESTful API_DESIGN v2.2.0. | Konsistensi referensi silang. |
| 3.0.0 | 2026-10-23 | Staff Ahli MIK & Antigravity Lead | Seluruh Dokumen | **Sebelum:** Scope eksklusif `SUPER_ADMIN`, penjadwalan via Celery Beat + Redis Broker, struktur kolom lama `content_blocks`.<br>**Sesudah:** Perluasan izin ke peran `AUTHOR` (Alur Mandiri: publish & schedule mandiri), adopsi struktur ganda `content_json` (AST JSONB) dan `content_html` (terfilter Rust `nh3`), eliminasi Celery (penjadwalan tayang murni query SQL), penegasan Kontrak Path Relatif (`/storage/public/{id}.webp`) untuk migrasi bebas broken image, dan larangan mutlak berkas `.svg`. | Pivot WordPress-like Blog CMS, simplifikasi arsitektur bebas Celery, dan jaminan portabilitas server. |

---

## 1. Arsitektur Editor Visual TipTap (WordPress-Equivalent)

Modul Blog dirancang untuk menghadirkan pengalaman menulis modern setara **WordPress Gutenberg**, menggunakan framework headless editor **TipTap** (berbasis ProseMirror). Editor menghasilkan dua format penyimpanan di tabel `blog_posts`:
1. **`content_json` (JSONB)**: Representasi pohon AST kanonik ProseMirror. Format ini sepenuhnya kebal dari eksekusi skrip mentah (*raw script injection*).
2. **`content_html` (TEXT)**: Cache HTML terkompilasi yang **telah disanitasi ketat oleh Rust `nh3` di server** untuk kebutuhan Server-Side Rendering (SSR) Next.js yang instan tanpa membebani browser pengunjung dengan parsing JS berat.

```mermaid
flowchart LR
    Editor["TipTap Visual Editor\n(React Client Frontend)"] -->|1. Generate JSON AST + Dirty HTML| API["FastAPI Endpoint"]
    API -->|2. Verify Session & Role Guard| AuthGuard["RBAC Guard"]
    AuthGuard -->|3. Server-Side Sanitizer (Rust nh3)| Sanitizer["nh3 Allowlist Filter\n(Strip script, onload, iframe jahat)"]
    Sanitizer -->|4. Simpan Bersih| DB[("PostgreSQL 16\n(content_json & content_html)")]
    DB -->|5. SSR HTML Instan| Visitor["Pengunjung Publik\n(Fast LCP < 1.5s, Zero Client Parsing)"]
```

---

## 2. Spesifikasi Blok & Fitur Toolbar Penulis

TipTap dikonfigurasi dengan modularitas ekstensi yang kaya:

### 2.1 Blok Teks & Tipografi
- **Headings**: H1, H2, H3, H4 (dengan auto-generated anchor ID untuk Table of Contents).
- **Paragraf & Formatting**: Bold, Italic, Strikethrough, Underline, Inline Code, Text Alignment (Left, Center, Right, Justify).
- **Lists**: Bullet List, Numbered List, dan Task/Checklist.
- **Kutipan (Blockquote)**: Format elegan dengan garis aksen dan atribusi nama narasumber.
- **Kode Program (Code Block)**: Syntax highlighting dengan pilihan bahasa pemrograman (Python, TypeScript, SQL, Bash).
- **Pemisah**: Horizontal Divider (`<hr>`).

### 2.2 Blok Media & Tabel Interaktif
- **Gambar Tunggal (Custom Image Node)**:
  - Menyisipkan gambar dari Media Library Cloudflare R2 atau upload instan.
  - Penataan tata letak: Rata kiri, tengah, rata kanan, atau *Full-Width*.
  - Caption teks dan Atribut Alt Text wajib untuk aksesibilitas (WCAG 2.1).
- **Tabel Data Dinamis**: Menambah/menghapus baris dan kolom, *header toggle*, dan penataan perataan sel.
- **Embed Aman (YouTube & Spotify)**:
  - Menggunakan custom embed node ber-sandbox.
  - URL divalidasi dengan regex ketat di server (hanya domain `youtube.com`, `youtu.be`, dan `spotify.com` yang diizinkan).
  - Tag `<iframe>` mentah dari sembarang sumber ditolak keras.

---

## 3. Kontrak Path Relatif (Anti-Broken Image Saat Migrasi Server)

Untuk memastikan gambar artikel tidak rusak saat website bermigrasi dari VPS Hostinger (`himaif.my.id`) ke VPS Kampus (`himaif.telkomuniversity.ac.id`):

> [!CRITICAL]
> **Aturan Wajib URL Gambar di Database:**
> Seluruh gambar di dalam `content_json` dan `content_html` **WAJIB disimpan sebagai Path Relatif**, contoh:
> ```json
> {
>   "type": "image",
>   "attrs": {
>     "src": "/storage/public/2026/10/a4b2c1d0-uuid.webp",
>     "alt": "Dokumentasi Seminar Informatika 2026"
>   }
> }
> ```
> **DILARANG KERAS** menyimpan URL absolut seperti `https://himaif.my.id/storage/...` atau `https://himaif.telkomuniversity.ac.id/storage/...`.
> Dengan path relatif, browser pengunjung secara otomatis memuat gambar dari origin host yang aktif saat ini, sehingga terbebas dari ancaman broken image dan error CORS pasca-migrasi.

---

## 4. Keamanan Sanitasi Ganda Anti-Stored XSS (Pustaka Rust `nh3`)

Dilarang mempercayai HTML yang dikirimkan oleh browser klien (bahkan dari akun Author yang sah). Backend FastAPI menyaring `content_html` menggunakan binding resmi Rust `nh3`:

```python
# app/core/sanitizer.py
import nh3

ALLOWED_TAGS = {
    "h1", "h2", "h3", "h4", "p", "blockquote", "ul", "ol", "li",
    "strong", "em", "strike", "u", "code", "pre", "hr",
    "table", "thead", "tbody", "tr", "th", "td",
    "img", "figure", "figcaption", "a"
}

ALLOWED_ATTRIBUTES = {
    "a": {"href", "title", "target", "rel"},
    "img": {"src", "alt", "title", "width", "height", "loading"},
    "th": {"colspan", "rowspan", "style"},
    "td": {"colspan", "rowspan", "style"},
    "p": {"style"},
    "h1": {"id"}, "h2": {"id"}, "h3": {"id"}, "h4": {"id"},
}

ALLOWED_URL_SCHEMES = {"http", "https", "mailto"}

def clean_blog_html(raw_html: str) -> str:
    cleaned = nh3.clean(
        raw_html,
        tags=ALLOWED_TAGS,
        attributes=ALLOWED_ATTRIBUTES,
        url_schemes=ALLOWED_URL_SCHEMES,
        link_rel="noopener noreferrer nofollow"
    )
    return cleaned
```

### Proteksi Tambahan:
1. **Larangan Format SVG**: File berformat `.svg` **dilarang keras untuk diunggah oleh penulis** guna melenyapkan seluruh vektor serangan SVG Stored XSS XML payload. Format gambar yang diizinkan eksklusif: `.webp`, `.png`, `.jpg`, `.jpeg`.
2. **Skema URL Terlarang**: Skema `javascript:`, `data:`, `vbscript:`, atau event handler HTML (`onload=`, `onerror=`) dilucuti secara total oleh `nh3`.

---

## 5. Penjadwalan Tayang Bebas Celery (*Celery-Free Scheduling*)

Untuk memangkas kompleksitas sistem (menghilangkan Celery Worker, Celery Beat, dan Redis Broker):
- Artikel yang dijadwalkan (`status = 'SCHEDULED'`) menyimpan waktu target pada kolom `published_at` di masa depan.
- **Logika Kueri Publik di Backend**:
  ```sql
  SELECT * FROM blog_posts 
  WHERE status = 'PUBLISHED' 
     OR (status = 'SCHEDULED' AND published_at <= NOW())
  ORDER BY published_at DESC;
  ```
- Dengan pendekatan ini, artikel terjadwal otomatis tampil di portal publik saat jam tayang terlampaui tanpa membutuhkan cron background worker yang rumit.
- Opsional: Cron job ringan harian menjalankan mutasi status dari `SCHEDULED` menjadi `PUBLISHED`:
  ```sql
  UPDATE blog_posts 
  SET status = 'PUBLISHED' 
  WHERE status = 'SCHEDULED' AND published_at <= NOW();
  ```

---

## 6. Real-Time SEO Scoring Engine & Metadata Otomatis

Platform dilengkapi modul kalkulasi skor SEO interaktif di editor frontend Next.js:
1. **Pemeriksaan Kepadatan Kata Kunci**: Memeriksa apakah kata kunci utama muncul pada Judul Artikel, Paragraf Pertama, dan Meta Description.
2. **Analisis Keterbacaan**: Menghitung estimasi waktu membaca (*Reading Time*) secara otomatis dengan rasio baku 200 kata per menit.
3. **Pemeriksaan Aksesibilitas**: Memvalidasi seluruh gambar di dalam dokumen memiliki atribut `alt` yang deskriptif.
4. **Pratinjau SERP Google & Social Card**: Simulasi visual tampilan tautan saat dibagikan ke Google Search, WhatsApp, Twitter/X, dan LinkedIn.

Spesifikasi ini menghadirkan engine publikasi berita yang ramah pengguna bagi pengurus himpunan, tangguh terhadap ancaman siber, dan memiliki performa tinggi di mesin pencari.
