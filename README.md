# Kenali Dirimu, Kenali Gerejamu — Web App

Single-file web app (`index.html`) untuk Kebaktian Pemuda Advent. Frontend: Tailwind CSS, FontAwesome, Animate.css, Chart.js, html2canvas (semua via CDN). Backend: Supabase (Postgres + Realtime) via `@supabase/supabase-js@2` (CDN).

## 1. Setup Supabase

Skema tabel di `schema.sql` sudah disesuaikan dengan skema yang sudah Anda buat di project Supabase (`diagnostic_responses`, `role_results`, `group_discussions`, `presenter_control`). Jalankan `schema.sql` di **SQL Editor** Supabase Anda — aman dijalankan ulang meskipun tabelnya sudah ada (`CREATE TABLE IF NOT EXISTS`), dan akan otomatis:
- menambahkan kolom-kolom baru yang dibutuhkan di `presenter_control` (`timer_started_at`, `timer_label`, `phase`) lewat `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` — tanpa menghapus data yang sudah ada;
- mengaktifkan Row Level Security + policy akses publik di keempat tabel (**wajib** — lihat catatan RLS di bawah).

> Kenapa 2 kolom timer itu perlu? Supaya timer sinkron secara **akurat** untuk HP yang baru dibuka di tengah hitung mundur (perhitungan sisa waktu dihitung dari `timer_started_at`, bukan dengan mengandalkan tick per detik dari server).

> ⚠️ **Penting soal kolom `phase`**: begitu SQL ini dijalankan, kolom `phase` di `presenter_control` otomatis terisi `'locked'` (nilai default) — artinya Kuesioner Diagnostik, Kuis Peran, dan Pilih Studi Kasus di HP peserta akan **langsung terkunci** sampai Anda membukanya sendiri lewat tombol gembok di Presenter View (lihat bagian Kontrol Fase Acara di bawah). Jalankan SQL ini di jeda antar-acara (sebelum peserta mulai mengisi), bukan di tengah-tengah acara yang sedang berjalan.

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
| `presenter_control` | `timer_seconds`, `is_running`, `timer_started_at`, `timer_label`, `phase` | 1 baris tunggal (`id = 1`) untuk sinkronisasi timer & gerbang fase acara |

## 5. Catatan Penting

- **Pertanyaan Kuesioner Diagnostik** (`DIAGNOSTIC_QUESTIONS` di dalam `index.html`) adalah **contoh default** yang selaras dengan tema acara, karena dokumen materi tidak menyertakan teks pertanyaan diagnostik spesifik. Silakan sunting sesuai kebutuhan jemaat Anda sebelum acara.
- Kuis Penemuan 6 Peran, 5 Studi Kasus, ayat Alkitab, dan kutipan Roh Nubuat sudah dimasukkan sesuai dokumen yang diberikan.
- Akses **Presenter View**: `https://domain-anda.pages.dev/#presenter` (atau klik tautan kecil "Mode Presenter" di pojok bawah Beranda).
- Akses **Client View** (default): `https://domain-anda.pages.dev/`.
- Supabase mengaktifkan **Row Level Security secara default** untuk tabel yang dibuat lewat SQL Editor. Tanpa policy eksplisit, semua insert/update/delete dari frontend (lewat anon key) akan ditolak dengan error `42501` ("new row violates row-level security policy"). `schema.sql` sudah menyertakan `ENABLE ROW LEVEL SECURITY` + policy akses publik (SELECT/INSERT/UPDATE/DELETE sesuai kebutuhan tiap tabel) untuk keempat tabel — pastikan bagian itu sudah dijalankan di project Anda.
- **Reset data sebelum acara berikutnya**: klik ikon sapu (🧹) di pojok kanan atas **Presenter View** untuk membuka menu pilihan reset. Presenter bisa pilih mau reset bagian mana saja: Kuesioner Diagnostik + hasil, Kuis Penemuan Peran + hasil, Pemilihan Studi Kasus & Diskusi Kelompok, Komitmen Kelompok saja (checklist & catatan — kasus & refleksi tetap dipertahankan), atau Reset Semuanya (semua data + timer dikembalikan ke 900 detik, berhenti). Ada konfirmasi sebelum tiap aksi dijalankan karena tidak bisa dibatalkan. Fitur ini butuh policy DELETE di `schema.sql` (lihat catatan RLS di atas) sudah dijalankan.
  - Catatan: tombol ini hanya membersihkan data di database. HP peserta yang sebelumnya sudah submit tetap menyimpan status "sudah selesai" secara lokal di browser masing-masing — mereka bisa memakai tombol "Isi ulang jawaban diagnostik" / "Ulangi kuis peran" di HP mereka, atau paling gampang buka link acara di tab/perangkat baru untuk sesi berikutnya.
  - Alternatif manual lewat SQL Editor (kalau perlu): `TRUNCATE diagnostic_responses, role_results, group_discussions;` lalu `UPDATE presenter_control SET is_running=false, timer_started_at=null, timer_seconds=900 WHERE id=1;`.
- **Slide "Studi Kasus" di Presenter View**: slide baru (setelah slide Live Role Assessment, sebelum Timer) menampilkan grid 5 studi kasus ke audience. Tap salah satu kartu untuk menampilkan detail lengkap (subjek, skenario, fokus diskusi) di layar besar, lengkap dengan status kelompok yang sudah mengambilnya (update realtime). Ada tombol "Kembali ke Daftar Kasus" untuk melihat kasus lain.
- **Transisi slide Presenter View**: berpindah slide (tombol next/prev atau panah kiri/kanan keyboard) sekarang punya animasi masuk/keluar yang lebih hidup (slide meluncur + efek bounce sesuai arah navigasi), dan elemen-elemen di dalam tiap slide (judul, kartu, kutipan) muncul bertahap.
- **Kuesioner Diagnostik & Kuis Penemuan Peran di HP peserta**: sekarang ditampilkan satu pertanyaan per layar (bukan daftar panjang sekaligus), dengan tombol Lanjut/Kembali, progress bar, dan font yang lebih besar — supaya pemuda maupun orang tua bisa lebih fokus mengisi dari HP masing-masing.
- **Kontrol Fase Acara (gerbang akses)**: klik ikon gembok di header **Presenter View** (di sebelah kiri tombol reset) untuk membuka menu fase acara: Terkunci → Kuesioner Diagnostik Dibuka → Kuis Penemuan Peran Dibuka → Pilih Studi Kasus Dibuka → Semua Sesi Dibuka. Selama presenter belum membuka suatu fase, peserta yang sudah scan QR/link akan melihat kartu "Belum Dibuka" alih-alih form aslinya — walaupun mereka sudah tahu link-nya dari sebelumnya. Begitu Anda memilih fase baru, semua HP peserta langsung ter-update secara realtime tanpa perlu refresh halaman. Anda juga bisa mundur ke fase sebelumnya kalau perlu koreksi. Fitur ini butuh kolom `phase` di `presenter_control` (lihat catatan di bagian Setup Supabase di atas) — kalau kolom itu belum ada, aplikasi otomatis menganggap semua fase terbuka (perilaku lama, tidak ada yang terkunci) supaya tidak tiba-tiba mengunci acara yang sedang berjalan.
