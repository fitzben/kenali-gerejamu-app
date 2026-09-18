# Kenali Dirimu, Kenali Gerejamu — Web App

Single-file web app (`index.html`) untuk Kebaktian Pemuda Advent. Frontend: Tailwind CSS, FontAwesome, Animate.css, Chart.js, html2canvas (semua via CDN). Backend: Supabase (Postgres + Realtime) via `@supabase/supabase-js@2` (CDN).

## 1. Setup Supabase

Skema tabel di `schema.sql` sudah disesuaikan dengan skema yang sudah Anda buat di project Supabase (`diagnostic_responses`, `role_results`, `group_discussions`, `presenter_control`). Jalankan `schema.sql` di **SQL Editor** Supabase Anda — aman dijalankan ulang meskipun tabelnya sudah ada (`CREATE TABLE IF NOT EXISTS`), dan akan otomatis:
- menambahkan 2 kolom baru yang dibutuhkan di `presenter_control` (`timer_started_at`, `timer_label`) lewat `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` — tanpa menghapus data yang sudah ada;
- mengaktifkan Row Level Security + policy akses publik di keempat tabel (**wajib** — lihat catatan RLS di bawah).

> Kenapa 2 kolom timer itu perlu? Supaya timer sinkron secara **akurat** untuk HP yang baru dibuka di tengah hitung mundur (perhitungan sisa waktu dihitung dari `timer_started_at`, bukan dengan mengandalkan tick per detik dari server).

Setelah itu, buka **Database → Replication** dan pastikan keempat tabel berstatus aktif untuk Realtime.

## 2. Konfigurasi index.html

Buka `index.html`, cari baris berikut di bagian atas tag `<script>`:

```js
const SUPABASE_URL = window.ENV_SUPABASE_URL || 'YOUR_SUPABASE_URL_HERE';
const SUPABASE_ANON_KEY = window.ENV_SUPABASE_ANON_KEY || 'YOUR_SUPABASE_ANON_KEY_HERE';
```

Ganti dengan **Project URL** dan **anon public key** dari **Project Settings → API** di Supabase Anda. Selama nilai ini masih placeholder, aplikasi otomatis berjalan dalam **MODE DEMO** (banner peringatan muncul, form tetap bisa dicoba tapi data tidak tersimpan).

## 3. Deploy ke Cloudflare Pages

**Opsi A — Drag & drop:** Cloudflare Dashboard → Workers & Pages → Create → Pages → Upload assets → upload `index.html`.

**Opsi B — Lewat Git:** hubungkan repo ini (`kenali-gerejamu-app`) di Cloudflare Pages → Create Project → Connect to Git. Build command: kosongkan. Build output directory: `/` (root).

## 4. Struktur Data (sesuai schema.sql)

| Tabel | Kolom kunci | Catatan |
|---|---|---|
| `diagnostic_responses` | `user_type`, `question_id`, `answer_value` | 1 baris per jawaban (6 baris per partisipan) |
| `role_results` | `user_type`, `role_name` | 1 baris per partisipan (hasil peran dominan) |
| `group_discussions` | `group_name`, `case_id` (UNIQUE), `understanding_text`, `communication_text`, `commitments` (jsonb `{checklist, note}`) | `case_id` UNIQUE = mekanisme locking studi kasus |
| `presenter_control` | `timer_seconds`, `is_running`, `timer_started_at`, `timer_label` | 1 baris tunggal (`id = 1`) untuk sinkronisasi timer |

## 5. Catatan Penting

- **Pertanyaan Kuesioner Diagnostik** (`DIAGNOSTIC_QUESTIONS` di dalam `index.html`) adalah **contoh default** yang selaras dengan tema acara, karena dokumen materi tidak menyertakan teks pertanyaan diagnostik spesifik. Silakan sunting sesuai kebutuhan jemaat Anda sebelum acara.
- Kuis Penemuan 6 Peran, 5 Studi Kasus, ayat Alkitab, dan kutipan Roh Nubuat sudah dimasukkan sesuai dokumen yang diberikan.
- Akses **Presenter View**: `https://domain-anda.pages.dev/#presenter` (atau klik tautan kecil "Mode Presenter" di pojok bawah Beranda).
- Akses **Client View** (default): `https://domain-anda.pages.dev/`.
- Supabase mengaktifkan **Row Level Security secara default** untuk tabel yang dibuat lewat SQL Editor. Tanpa policy eksplisit, semua insert/update/delete dari frontend (lewat anon key) akan ditolak dengan error `42501` ("new row violates row-level security policy"). `schema.sql` sudah menyertakan `ENABLE ROW LEVEL SECURITY` + policy akses publik (SELECT/INSERT/UPDATE/DELETE sesuai kebutuhan tiap tabel) untuk keempat tabel — pastikan bagian itu sudah dijalankan di project Anda.
- **Reset data sebelum acara berikutnya**: klik ikon sapu (🧹) di pojok kanan atas **Presenter View** — ini akan menghapus semua jawaban diagnostik, hasil kuis peran, dan diskusi kelompok, lalu mereset timer ke kondisi awal (900 detik, berhenti). Ada konfirmasi sebelum aksi ini dijalankan karena tidak bisa dibatalkan. Tombol ini butuh policy DELETE di `schema.sql` (lihat catatan RLS di atas) sudah dijalankan.
  - Catatan: tombol ini hanya membersihkan data di database. HP peserta yang sebelumnya sudah submit tetap menyimpan status "sudah selesai" secara lokal di browser masing-masing — mereka bisa memakai tombol "Isi ulang jawaban diagnostik" / "Ulangi kuis peran" di HP mereka, atau paling gampang buka link acara di tab/perangkat baru untuk sesi berikutnya.
  - Alternatif manual lewat SQL Editor (kalau perlu): `TRUNCATE diagnostic_responses, role_results, group_discussions;` lalu `UPDATE presenter_control SET is_running=false, timer_started_at=null, timer_seconds=900 WHERE id=1;`.
