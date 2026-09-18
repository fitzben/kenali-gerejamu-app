# Kenali Dirimu, Kenali Gerejamu — Web App

Single-file web app (`index.html`) untuk Kebaktian Pemuda Advent. Frontend: Tailwind CSS, FontAwesome, Animate.css, Chart.js, html2canvas (semua via CDN). Backend: Supabase (Postgres + Realtime) via `@supabase/supabase-js@2` (CDN).

## 1. Setup Supabase

Skema tabel di `schema.sql` sudah disesuaikan dengan skema yang sudah Anda buat di project Supabase (`diagnostic_responses`, `role_results`, `group_discussions`, `presenter_control`). Jalankan `schema.sql` di **SQL Editor** Supabase Anda — aman dijalankan ulang meskipun tabelnya sudah ada (`CREATE TABLE IF NOT EXISTS`), dan akan otomatis menambahkan 2 kolom baru yang dibutuhkan di `presenter_control` (`timer_started_at`, `timer_label`) lewat `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` — tanpa menghapus data yang sudah ada.

> Kenapa 2 kolom itu perlu? Supaya timer sinkron secara **akurat** untuk HP yang baru dibuka di tengah hitung mundur (perhitungan sisa waktu dihitung dari `timer_started_at`, bukan dengan mengandalkan tick per detik dari server).

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
- Tabel-tabel di atas dibuat tanpa RLS eksplisit — di Supabase ini tetap bisa diakses via anon key (perilaku default), cocok untuk acara satu malam. Lihat komentar di akhir `schema.sql` jika ingin mengaktifkan RLS.
- Reset data sebelum acara berikutnya: `TRUNCATE diagnostic_responses, role_results, group_discussions;` lalu `UPDATE presenter_control SET is_running=false, timer_started_at=null, timer_seconds=900 WHERE id=1;`.
